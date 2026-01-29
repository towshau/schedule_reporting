# Sale Capacity Formula Explanation

## Overview

The `sale_without_change` and `sale_add_change` formulas calculate how many new members can be added to a class, either:
- **Without changing capacity** (using existing buffer)
- **With capacity expansion** (increasing to maximum capacity)

---

## Current Formula (As Implemented)

### `sale_without_change`
```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
```

**What it means:** How many members can be added using the existing capacity buffer (space between current utilization and target utilization).

### `sale_add_change` (Current Implementation)
```
sale_add_change = sale_without_change + max_class_type_capacity
```

**Issue:** This adds the full maximum capacity value, which may not be correct if current capacity is already close to max.

---

## Formula Options

### Option 1: Add Capacity Increase (Recommended)
```
sale_add_change = sale_without_change + ((max_capacity - current_capacity) × operational_days / avg_perform_consumption)
```

**What it means:** Members from buffer + members from capacity expansion.

### Option 2: Calculate from Max Capacity Directly
```
sale_add_change = ((lc_buffer × max_capacity × operational_days) - total_attendance) / avg_perform_consumption
```

**What it means:** Total potential members if we expand to max capacity and fill to target utilization.

### Option 3: Current Formula (May Need Review)
```
sale_add_change = sale_without_change + max_class_type_capacity
```

**What it means:** Members from buffer + maximum capacity value (unclear if this represents members or spots).

---

## Configuration Values

- **`lc_buffer`**: 0.9 (90% target utilization)
- **`avg_perform_consumption`**: 0.4 (average sessions per member per week)
- **PERFORM class max capacities:**
  - BLIGH: 10 spots/day
  - BRIDGE: 8 spots/day
  - COLLIN: 6 spots/day

---

## Example 1: Class That Is Quite Full

### Scenario
- **Gym**: BRIDGE
- **Session**: PERFORM - BRG AM at 05:30:00
- **Current capacity**: 6 spots/day (3 coaches × 2 base capacity)
- **Max capacity**: 8 spots/day
- **Operational days**: 5 (Mon-Fri)
- **Total weekly capacity**: 30 spots (6 × 5)
- **Total attendance**: 27 attendees
- **Utilization rate**: 90% (27/30)

### Calculations

#### Step 1: Utilization Rate
```
utilization_rate = total_attendance / total_capacity
                 = 27 / 30
                 = 0.90 (90%)
```

#### Step 2: Average Spots Per Day
```
average_spots = total_capacity / operational_days
              = 30 / 5
              = 6.0 spots/day
```

#### Step 3: Sale Without Change
```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
                    = ((0.9 - 0.9) × 6.0) / 0.4
                    = (0.0 × 6.0) / 0.4
                    = 0 / 0.4
                    = 0 members
```

**Interpretation:** No members can be added without changing capacity (already at 90% target).

#### Step 4: Sale Add Change (Current Formula)
```
sale_add_change = sale_without_change + max_class_type_capacity
                = 0 + 8
                = 8 members
```

**Issue:** This suggests 8 members can be added, but capacity can only increase by 2 spots/day.

#### Step 4a: Sale Add Change (Option 1 - Recommended)
```
capacity_increase = max_capacity - current_capacity
                  = 8 - 6
                  = 2 spots/day

additional_weekly_spots = capacity_increase × operational_days
                        = 2 × 5
                        = 10 spots/week

additional_members = additional_weekly_spots / avg_perform_consumption
                   = 10 / 0.4
                   = 25 members

sale_add_change = sale_without_change + additional_members
                = 0 + 25
                = 25 members
```

**Interpretation:** If capacity is expanded from 6 to 8 spots/day, you can add 25 new members.

#### Step 4b: Sale Add Change (Option 2)
```
max_weekly_capacity = max_capacity × operational_days
                    = 8 × 5
                    = 40 spots/week

target_attendance = max_weekly_capacity × lc_buffer
                  = 40 × 0.9
                  = 36 attendees

available_spots = target_attendance - current_attendance
                = 36 - 27
                = 9 spots

sale_add_change = available_spots / avg_perform_consumption
                = 9 / 0.4
                = 22.5 members
```

**Interpretation:** If expanded to max capacity and filled to 90% utilization, you can add 22.5 members.

---

## Example 2: Class That Is Empty

### Scenario
- **Gym**: BLIGH
- **Session**: PERFORM - BLIGH AM at 05:30:00
- **Current capacity**: 4 spots/day (2 coaches × 2 base capacity)
- **Max capacity**: 10 spots/day
- **Operational days**: 5 (Mon-Fri)
- **Total weekly capacity**: 20 spots (4 × 5)
- **Total attendance**: 2 attendees
- **Utilization rate**: 10% (2/20)

### Calculations

#### Step 1: Utilization Rate
```
utilization_rate = total_attendance / total_capacity
                 = 2 / 20
                 = 0.10 (10%)
```

#### Step 2: Average Spots Per Day
```
average_spots = total_capacity / operational_days
              = 20 / 5
              = 4.0 spots/day
```

#### Step 3: Sale Without Change
```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
                    = ((0.9 - 0.1) × 4.0) / 0.4
                    = (0.8 × 4.0) / 0.4
                    = 3.2 / 0.4
                    = 8 members
```

**Interpretation:** 8 members can be added at current capacity before reaching 90% utilization.

#### Step 4: Sale Add Change (Current Formula)
```
sale_add_change = sale_without_change + max_class_type_capacity
                = 8 + 10
                = 18 members
```

**Issue:** This suggests 18 members total, but doesn't clearly show the breakdown.

#### Step 4a: Sale Add Change (Option 1 - Recommended)
```
capacity_increase = max_capacity - current_capacity
                  = 10 - 4
                  = 6 spots/day

additional_weekly_spots = capacity_increase × operational_days
                        = 6 × 5
                        = 30 spots/week

additional_members = additional_weekly_spots / avg_perform_consumption
                   = 30 / 0.4
                   = 75 members

sale_add_change = sale_without_change + additional_members
                = 8 + 75
                = 83 members
```

**Interpretation:** 
- 8 members can be added at current capacity (4 spots/day)
- 75 additional members if capacity expands to max (10 spots/day)
- **Total potential: 83 members**

#### Step 4b: Sale Add Change (Option 2)
```
max_weekly_capacity = max_capacity × operational_days
                    = 10 × 5
                    = 50 spots/week

target_attendance = max_weekly_capacity × lc_buffer
                  = 50 × 0.9
                  = 45 attendees

available_spots = target_attendance - current_attendance
                = 45 - 2
                = 43 spots

sale_add_change = available_spots / avg_perform_consumption
                = 43 / 0.4
                = 107.5 members
```

**Interpretation:** If expanded to max capacity (10 spots/day) and filled to 90% utilization, you can add 107.5 members.

---

## Comparison Summary

| Scenario | Current Formula | Option 1 (Capacity Increase) | Option 2 (Max Direct) |
|----------|----------------|------------------------------|----------------------|
| **Full Class** (90% util) | 8 members | 25 members | 22.5 members |
| **Empty Class** (10% util) | 18 members | 83 members | 107.5 members |

### Key Differences

1. **Current Formula**: Adds max capacity value directly (unclear if this represents members or spots)
2. **Option 1**: Calculates additional members from capacity expansion (clearer logic)
3. **Option 2**: Calculates total potential from max capacity directly (simpler but less granular)

---

## Recommendation

**Option 1** is recommended because it:
- Clearly separates "members at current capacity" vs "members from expansion"
- Uses the actual capacity increase (max - current), not the full maximum
- Provides a logical breakdown: `sale_without_change + expansion_members`
- Makes it clear that you're adding the capacity increase, not the full maximum

### Proposed Formula Change

```sql
sale_add_change = sale_without_change + 
                  ((max_class_type_capacity - class_instance_capacity) × operational_days / avg_perform_consumption)
```

This would change:
- **Full class example**: 0 + ((8-6)×5/0.4) = 0 + 25 = **25 members**
- **Empty class example**: 8 + ((10-4)×5/0.4) = 8 + 75 = **83 members**

---

## Questions to Consider

1. **What does the current formula intend to represent?**
   - Is `max_class_type_capacity` meant to represent members or spots?
   - Should it be the full max or the increase?

2. **What is the business logic?**
   - Do you want to show: "members at current + members from expansion"?
   - Or: "total potential if expanded to max"?

3. **Which is more useful for decision-making?**
   - Option 1 shows incremental value of expansion
   - Option 2 shows total potential at maximum capacity
