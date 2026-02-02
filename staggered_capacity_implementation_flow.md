# Staggered Capacity Implementation Flow

## Overview

This document explains the complete flow of how staggered capacity calculation will work from the daily view through to the weekly view.

---

## Current Data Flow

### Step 1: Raw Data → Daily View
**Source:** `member_daily_sessions_attended` table
**Output:** `view_data_attendance_26_mvp_csu` (daily materialized view)

**What happens:**
1. Raw attendance data is inserted into `member_daily_sessions_attended`
2. Daily view aggregates by: `session_date`, `session_start`, `session_type`, `gym`
3. Calculates:
   - `class_type_capacity` = base capacity from system_config
   - `class_instance_capacity` = base capacity × number of coaches
   - `max_class_type_capacity` = gym-specific max (BLIGH=10, BRIDGE=8, COLLIN=6)

**Current Daily View Columns:**
- `session_date`
- `session_start`
- `session_type`
- `gym`
- `class_type`
- `coaches` (aggregated list)
- `members_attended` (aggregated list)
- `actual_attendance` (count)
- `class_type_capacity` (base capacity)
- `class_instance_capacity` (capacity × coaches)
- `max_class_type_capacity` (gym max - currently unadjusted)

### Step 2: Daily View → Weekly View
**Source:** `view_data_attendance_26_mvp_csu` (daily view)
**Output:** `view_attendance_weekly_26_mvp_csu` (weekly materialized view)

**What happens:**
1. Weekly view reads from daily view
2. Filters: Monday-Friday only (`EXTRACT(DOW FROM session_date) BETWEEN 1 AND 5`)
3. Groups by: `week_start`, `session_type`, `session_start`, `gym`
4. Aggregates:
   - `MAX(max_class_type_capacity)` = takes the max capacity value
   - `MAX(class_instance_capacity)` = takes the max current capacity
   - `SUM(actual_attendance)` = sums attendance across the week
   - `COUNT(DISTINCT session_date)` = operational days
5. Calculates:
   - `utilization_rate` = total_attendance / total_capacity
   - `sale_without_change` = formula using utilization
   - `sale_add_change` = formula using max capacity

---

## Proposed Flow with Staggered Capacity

### Step 1: Raw Data → Daily View (WITH Staggered Calculation)

**What we're adding:**
- Calculate `adjusted_max_capacity` in the daily view
- This replaces `max_class_type_capacity` with the staggered-adjusted value

**New Logic in Daily View:**

```sql
-- In view_data_attendance_26_mvp_csu
WITH base_data AS (
    -- Current daily view logic (grouping, aggregating, etc.)
    SELECT 
        session_date,
        session_start,
        session_type,
        gym,
        class_type,
        class_instance_capacity,
        max_class_type_capacity,
        -- ... other columns
    FROM member_daily_sessions_attended m
    LEFT JOIN system_config sc ON sc.match_pattern = m.class_type
    WHERE ...
    GROUP BY ...
),
block_calculation AS (
    SELECT 
        *,
        -- Calculate which 40-minute block (0-11 only for staggered)
        CASE 
            WHEN gym = 'BLIGH' THEN
                FLOOR(
                    EXTRACT(EPOCH FROM (session_start - '05:30:00'::time)) / 2400
                )::int
            ELSE NULL
        END as block_number
    FROM base_data
),
concurrent_capacity AS (
    SELECT 
        *,
        -- Sum of all capacities in the same block (for BLIGH only)
        SUM(class_instance_capacity) OVER (
            PARTITION BY session_date, gym, block_number
        ) as total_block_capacity,
        COUNT(*) OVER (
            PARTITION BY session_date, gym, block_number
        ) as classes_in_block
    FROM block_calculation
)
SELECT 
    *,
    -- Calculate adjusted max capacity
    CASE 
        -- Only apply staggered logic to BLIGH, blocks 0-11
        WHEN gym = 'BLIGH' 
         AND block_number < 12
         AND classes_in_block > 1 
         AND total_block_capacity > max_class_type_capacity THEN
            -- Proportional allocation
            (class_instance_capacity::numeric / total_block_capacity::numeric) * max_class_type_capacity
        ELSE
            -- Use full max capacity (not BLIGH, or block 12+, or single class)
            max_class_type_capacity
    END as adjusted_max_capacity
FROM concurrent_capacity
```

**Result:**
- Daily view now has `adjusted_max_capacity` column
- This value is calculated once per day
- Each row in daily view has its adjusted max capacity

### Step 2: Daily View → Weekly View (Aggregation)

**What happens:**
1. Weekly view reads from daily view (which now has `adjusted_max_capacity`)
2. Groups by week, session type, start time, gym
3. Uses `MAX(adjusted_max_capacity)` instead of `MAX(max_class_type_capacity)`

**Updated Weekly View Logic:**

```sql
-- In view_attendance_weekly_26_mvp_csu
SELECT 
    DATE_TRUNC('week', v.session_date)::date AS week_start,
    v.session_type,
    v.session_start,
    v.gym,
    v.class_type,
    MAX(v.class_type_capacity) AS class_type_capacity,
    MAX(v.class_instance_capacity) AS class_instance_capacity,
    MAX(v.adjusted_max_capacity) AS max_class_type_capacity,  -- ← Uses adjusted value
    -- ... rest of aggregation
    -- sale_add_change formula uses max_class_type_capacity (which is now adjusted)
    CASE
        WHEN ... THEN
            sale_without_change + 
            ((MAX(v.adjusted_max_capacity) - MAX(v.class_instance_capacity)) * 
             COUNT(DISTINCT v.session_date) / avg_perform_consumption)
    END AS sale_add_change
FROM view_data_attendance_26_mvp_csu v
WHERE ...
GROUP BY ...
```

**Key Point:**
- Weekly view uses `MAX(adjusted_max_capacity)` from daily view
- This means the staggered calculation happens **once** in daily view
- Weekly view just takes the maximum adjusted value across the week

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Raw Data Entry                                           │
│    member_daily_sessions_attended                           │
│    - Individual session records                             │
│    - session_date, session_start, member_name, etc.         │
└────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Daily View Calculation                                   │
│    view_data_attendance_26_mvp_csu                          │
│                                                              │
│    A. Group by date/time/gym                                │
│    B. Calculate base capacities                             │
│    C. Calculate block_number (40-min blocks, 0-11 only)     │
│    D. Find concurrent classes in same block                 │
│    E. Calculate adjusted_max_capacity:                     │
│       - If BLIGH + block < 12 + multiple classes:           │
│         → Proportional split                                │
│       - Otherwise:                                          │
│         → Full max capacity                                 │
│                                                              │
│    OUTPUT: Each row has adjusted_max_capacity               │
└────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Weekly View Aggregation                                   │
│    view_attendance_weekly_26_mvp_csu                        │
│                                                              │
│    A. Filter: Monday-Friday only                            │
│    B. Group by: week_start, session_type, session_start     │
│    C. Aggregate:                                            │
│       - MAX(adjusted_max_capacity) → max_class_type_capacity│
│       - MAX(class_instance_capacity)                        │
│       - SUM(actual_attendance)                              │
│       - COUNT(DISTINCT session_date) → operational_days    │
│    D. Calculate formulas:                                   │
│       - utilization_rate                                    │
│       - sale_without_change                                 │
│       - sale_add_change (uses adjusted max)                │
│                                                              │
│    OUTPUT: Weekly summary with adjusted capacities          │
└────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Retool Dashboard                                         │
│    Queries weekly view for display                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Points

### ✅ Why Calculate in Daily View?

1. **Performance**
   - Calculation happens **once** during daily view refresh
   - Weekly view just aggregates (faster)
   - No recalculation during weekly aggregation

2. **Consistency**
   - Same adjusted capacity used everywhere
   - If daily view is queried directly, it has correct values
   - No risk of different calculations in different places

3. **Debugging**
   - Can see adjusted capacity at daily level
   - Easy to verify: "Did 6:10am get 7.5 spots on Monday?"
   - Can query daily view to troubleshoot

### ✅ How Weekly Aggregation Works

**The weekly view uses `MAX(adjusted_max_capacity)`:**
- Takes the **maximum** adjusted capacity across all days in the week
- If Monday: 7.5 spots, Tuesday: 8.0 spots, Wednesday: 7.5 spots
- Weekly view shows: **8.0 spots** (the max)

**Why MAX?**
- Represents the "best case" capacity for that session across the week
- If capacity varies day-to-day, we use the highest value
- Ensures we don't underestimate potential capacity

### ✅ Example Flow

**Monday, 6:10am class:**
1. **Daily View:**
   - Block 1 (6:10am - 6:50am)
   - 6:10am: 6 spots, 6:30am: 2 spots (total 8)
   - Adjusted: (6/8) × 10 = **7.5 spots** ✅

2. **Weekly View:**
   - Groups Monday-Friday 6:10am classes
   - Takes MAX(adjusted_max_capacity) across all days
   - If Monday=7.5, Tuesday=8.0, Wednesday=7.5
   - Shows: **8.0 spots** as max_class_type_capacity

3. **Sale Add Change Calculation:**
   - Uses 8.0 (from weekly aggregation)
   - Formula: `sale_without_change + ((8.0 - current_capacity) × days / 0.4)`

---

## Summary

**Flow:**
1. **Daily View** → Calculates `adjusted_max_capacity` once per day
2. **Weekly View** → Aggregates using `MAX(adjusted_max_capacity)` 
3. **Formulas** → Use the aggregated adjusted max capacity

**Benefits:**
- ✅ Calculate once, use many times (performance)
- ✅ Consistent values across all views
- ✅ Easy to debug at daily level
- ✅ Weekly view stays simple (just aggregation)

**The staggered logic lives in the daily view, and the weekly view simply uses the pre-calculated adjusted values.**

