# Daily View vs Weekly View: Staggered Capacity Calculation

## Overview

When implementing **Option 2: Proportional Split** for staggered classes, you need to decide where to calculate the adjusted max capacity. This document explains the trade-offs between calculating in the **daily view** vs the **weekly view**.

---

## How to Detect Staggered Classes

### The Challenge

Staggered classes are classes that run **at the same time** (or overlapping times) and **share the same capacity pool**. For example:
- 6:15am class and 6:30am class both running simultaneously
- Both share BLIGH's 10-spot maximum capacity

### Detection Methods

#### Method 1: Time Window Approach (Recommended)

**Concept:** Define time windows where classes are considered "concurrent" if they start within the same window.

**Example:**
```sql
-- Classes starting between 6:00-6:45 are in the same window
CASE 
    WHEN session_start BETWEEN '06:00:00' AND '06:45:00' THEN '06:00-06:45'
    WHEN session_start BETWEEN '07:00:00' AND '07:45:00' THEN '07:00-07:45'
    -- ... etc
END as time_window
```

**Logic:**
- If multiple classes have the same `time_window`, `gym`, and `session_date` → they're staggered
- If only one class in a window → not staggered (uses full max capacity)

**Pros:**
- ✅ Simple to understand
- ✅ Configurable (can adjust window sizes)
- ✅ Handles most real-world scenarios

**Cons:**
- ❌ Requires defining all time windows
- ❌ May miss edge cases (e.g., 6:44 and 6:46 are in different windows but might overlap)

#### Method 2: Overlapping Time Ranges

**Concept:** Classes are staggered if their time ranges overlap.

**Example:**
```sql
-- Assume classes run for 30 minutes
WITH class_ranges AS (
    SELECT 
        *,
        session_start as range_start,
        session_start + INTERVAL '30 minutes' as range_end
    FROM view_data_attendance_26_mvp_csu
)
-- Find overlapping classes
SELECT 
    a.*,
    COUNT(*) OVER (
        PARTITION BY a.session_date, a.gym
        WHERE b.range_start < a.range_end 
          AND b.range_end > a.range_start
    ) as concurrent_count
FROM class_ranges a
LEFT JOIN class_ranges b ON 
    a.session_date = b.session_date 
    AND a.gym = b.gym
    AND a.session_type != b.session_type
```

**Pros:**
- ✅ More accurate (handles actual overlaps)
- ✅ No manual window definitions

**Cons:**
- ❌ More complex SQL
- ❌ Requires knowing class duration
- ❌ Performance impact (self-join)

#### Method 3: Fixed Time Buckets (15-minute intervals)

**Concept:** Round start times to 15-minute buckets, classes in the same bucket are concurrent.

**Example:**
```sql
-- Round to nearest 15 minutes
DATE_TRUNC('hour', session_start) + 
    (EXTRACT(MINUTE FROM session_start)::int / 15) * INTERVAL '15 minutes' 
    as time_bucket
```

**Pros:**
- ✅ Very simple
- ✅ No configuration needed
- ✅ Fast performance

**Cons:**
- ❌ May group classes that don't actually overlap (e.g., 6:14 and 6:16)
- ❌ May miss classes that do overlap but are in different buckets

---

## Daily View Calculation (Option 2b)

### How It Works

Calculate the adjusted max capacity **once** in the daily view, then the weekly view just aggregates the pre-calculated values.

### Implementation Example

```sql
-- In view_data_attendance_26_mvp_csu
WITH time_windows AS (
    SELECT 
        *,
        -- Define time windows (e.g., 6:00-6:45, 7:00-7:45)
        CASE 
            WHEN session_start BETWEEN '06:00:00' AND '06:45:00' THEN '06:00-06:45'
            WHEN session_start BETWEEN '07:00:00' AND '07:45:00' THEN '07:00-07:45'
            -- Add more windows as needed
            ELSE 'other'
        END as time_window
),
concurrent_capacity AS (
    SELECT 
        *,
        -- Sum of all capacities in the same window
        SUM(class_instance_capacity) OVER (
            PARTITION BY session_date, gym, time_window
        ) as total_window_capacity,
        -- Count of classes in the same window
        COUNT(*) OVER (
            PARTITION BY session_date, gym, time_window
        ) as concurrent_count
    FROM time_windows
)
SELECT 
    *,
    CASE 
        -- Only adjust if: BLIGH gym, multiple classes in window, and total exceeds max
        WHEN gym = 'BLIGH' 
         AND concurrent_count > 1 
         AND total_window_capacity > max_class_type_capacity THEN
            -- Proportional allocation
            (class_instance_capacity::numeric / total_window_capacity::numeric) * max_class_type_capacity
        ELSE
            -- Use full max capacity
            max_class_type_capacity
    END as adjusted_max_capacity
FROM concurrent_capacity
```

### Pros ✅

1. **Performance**
   - Calculation happens **once** during daily view refresh
   - Weekly view just aggregates (faster)
   - No recalculation during weekly aggregation

2. **Debugging**
   - Can see adjusted capacity at the daily level
   - Easy to verify: "Did 6:15am get 7.5 spots on Monday?"
   - Can query daily view directly to troubleshoot

3. **Consistency**
   - Same adjusted capacity used everywhere
   - If daily view is used in other reports, they get correct values
   - No risk of different calculations in different views

4. **Granularity**
   - See per-day allocation breakdowns
   - Understand daily variations
   - Better for operational insights

### Cons ❌

1. **Complexity**
   - Daily view becomes more complex
   - More logic to maintain
   - Harder to understand the daily view definition

2. **Modification Impact**
   - Changes to daily view affect all downstream views
   - Need to refresh daily view for changes to take effect
   - If daily view is used elsewhere, changes affect everything

3. **Time Window Maintenance**
   - Need to define time windows in daily view
   - If schedule changes, need to update window definitions
   - More places to maintain configuration

---

## Weekly View Calculation (Option 2a)

### How It Works

Calculate the adjusted max capacity **during** weekly aggregation, based on concurrent classes in the weekly grouping.

### Implementation Example

```sql
-- In view_attendance_weekly_26_mvp_csu
WITH daily_sessions AS (
    SELECT * FROM view_data_attendance_26_mvp_csu
    WHERE EXTRACT(DOW FROM session_date) BETWEEN 1 AND 5
),
time_windows AS (
    SELECT 
        *,
        CASE 
            WHEN session_start BETWEEN '06:00:00' AND '06:45:00' THEN '06:00-06:45'
            WHEN session_start BETWEEN '07:00:00' AND '07:45:00' THEN '07:00-07:45'
            ELSE 'other'
        END as time_window
),
concurrent_capacity AS (
    SELECT 
        *,
        SUM(class_instance_capacity) OVER (
            PARTITION BY session_date, gym, time_window
        ) as total_window_capacity,
        COUNT(*) OVER (
            PARTITION BY session_date, gym, time_window
        ) as concurrent_count
    FROM time_windows
),
adjusted_daily AS (
    SELECT 
        *,
        CASE 
            WHEN gym = 'BLIGH' 
             AND concurrent_count > 1 
             AND total_window_capacity > max_class_type_capacity THEN
                (class_instance_capacity::numeric / total_window_capacity::numeric) * max_class_type_capacity
            ELSE
                max_class_type_capacity
        END as adjusted_max_capacity
    FROM concurrent_capacity
)
SELECT 
    DATE_TRUNC('week', session_date)::date AS week_start,
    session_type,
    session_start,
    gym,
    MAX(adjusted_max_capacity) AS max_class_type_capacity,  -- Use adjusted value
    -- ... rest of weekly aggregation
FROM adjusted_daily
GROUP BY week_start, session_type, session_start, gym
```

### Pros ✅

1. **Isolation**
   - Doesn't touch the daily view
   - Daily view stays simple and unchanged
   - Changes isolated to weekly view only

2. **Flexibility**
   - Can experiment with different calculation methods
   - Easy to revert if needed
   - Doesn't affect other uses of daily view

3. **Simplicity**
   - Daily view remains straightforward
   - All complex logic in one place (weekly view)
   - Easier to understand daily view definition

### Cons ❌

1. **Performance**
   - Calculation happens **during** weekly aggregation
   - More complex weekly view = slower refresh
   - Recalculates every time weekly view refreshes

2. **Debugging**
   - Harder to see daily breakdowns
   - Can't easily verify: "What was the adjusted capacity on Monday?"
   - Need to query with time window logic to debug

3. **Inconsistency**
   - Daily view shows unadjusted max capacity
   - Weekly view shows adjusted max capacity
   - Different values in different places (confusing)

4. **Reusability**
   - If daily view is used elsewhere, it won't have adjusted capacity
   - Other reports might show incorrect max capacity
   - Need to duplicate logic if used in multiple places

---

## Comparison Table

| Factor | Daily View | Weekly View |
|--------|------------|-------------|
| **Performance** | ⭐⭐⭐ Faster (calculated once) | ⭐⭐ Slower (recalculated on refresh) |
| **Debugging** | ⭐⭐⭐ Easy (see daily values) | ⭐ Hard (aggregated values) |
| **Complexity** | ⭐⭐ More complex daily view | ⭐⭐⭐ More complex weekly view |
| **Isolation** | ⭐ Affects all downstream | ⭐⭐⭐ Isolated to weekly only |
| **Consistency** | ⭐⭐⭐ Same everywhere | ⭐ Different in different views |
| **Maintenance** | ⭐⭐ One place to maintain | ⭐⭐⭐ Only weekly view |
| **Flexibility** | ⭐ Harder to change | ⭐⭐⭐ Easy to experiment |

---

## Recommendation

### Use **Daily View Calculation** if:
- ✅ You need to see daily breakdowns for debugging
- ✅ Daily view is used in multiple places (consistency matters)
- ✅ Performance is important (frequent refreshes)
- ✅ You want granular insights at the daily level

### Use **Weekly View Calculation** if:
- ✅ Daily view is used elsewhere and you don't want to change it
- ✅ You want to experiment with different calculation methods
- ✅ Weekly view is the only place that needs adjusted capacity
- ✅ You prefer keeping daily view simple

---

## Implementation Recommendation

**I recommend Daily View Calculation** because:

1. **Better Performance**: Calculate once, use many times
2. **Better Debugging**: Can see exactly what happened each day
3. **Consistency**: Same adjusted capacity everywhere
4. **Future-Proof**: If you add other reports, they automatically get correct values

The only downside is making the daily view slightly more complex, but the benefits outweigh this.

---

## Detection Logic Summary

**How the system knows staggered vs non-staggered:**

1. **Group classes by:**
   - Same `session_date`
   - Same `gym`
   - Same `time_window` (e.g., 6:00-6:45)

2. **Check if staggered:**
   - If `COUNT(*) > 1` in the same group → **staggered**
   - If `COUNT(*) = 1` in the group → **not staggered**

3. **Apply adjustment:**
   - **Staggered**: Proportional split `(slot_capacity / total_capacity) × max`
   - **Not staggered**: Full max capacity

**Example:**
- Monday 6:15am: 3 coaches, 6 spots
- Monday 6:30am: 1 coach, 2 spots
- Both in "06:00-06:45" window
- Total: 8 spots, but max is 10
- **Result**: Not over limit, but if total was 12, would split proportionally

---

## Next Steps

1. **Choose detection method** (Time Window recommended)
2. **Define time windows** for staggered classes
3. **Implement in chosen view** (Daily recommended)
4. **Test with sample data**
5. **Monitor and adjust** as needed

