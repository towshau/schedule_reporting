# Staggered Class Capacity Allocation Options

## Problem Statement

At **BLIGH Street**, the maximum capacity is set to **10 spots** for PERFORM classes. However, classes can run at **staggered times** (e.g., 6:15am and 6:30am) simultaneously, effectively **sharing** that 10-spot capacity pool.

### Current Situation
- **Max capacity (BLIGH)**: 10 spots
- **Staggered classes**: Multiple time slots running concurrently
- **Example**: 6:15am class + 6:30am class both running at the same time
- **Issue**: Both classes can't each have 10 spots - they share the 10 total

### Example Scenario
- **6:15am class**: 3 coaches = 6 spots capacity
- **6:30am class**: 1 coach = 2 spots capacity
- **Total**: 8 spots used out of 10 max
- **Problem**: Current system might show each class can expand to 10, but they share the pool

---

## Option 1: Fixed Split (Simplest)

### Approach
Set the maximum capacity to **5 spots per class** for all staggered time slots at BLIGH.

### Implementation
```sql
-- Update system_config
UPDATE system_config 
SET max_bligh = 5 
WHERE match_pattern = 'PERFORM';
```

### Pros
- ✅ **Simple**: One value, easy to understand
- ✅ **Fair**: Equal allocation for all time slots
- ✅ **Safe**: Prevents over-allocation
- ✅ **Easy to maintain**: No complex calculations
- ✅ **Clear**: Everyone knows each slot gets 5 max

### Cons
- ❌ **Rigid**: Doesn't adapt to actual coach allocation
- ❌ **Inefficient**: If one slot has 3 coaches and another has 1, both still limited to 5
- ❌ **May be too conservative**: Total capacity might be underutilized

### When to Use
- If staggered classes are roughly equal in size
- If you want simplicity over optimization
- If capacity is rarely a constraint

---

## Option 2: Proportional Split Based on Coach Allocation (Complex)

### Approach
Split the 10-spot maximum proportionally based on the number of coaches at each time slot.

### Formula
```
slot_capacity = (slot_coaches / total_coaches_at_time) × max_bligh
```

### Example Calculation
**Scenario:**
- 6:15am: 3 coaches
- 6:30am: 1 coach
- Total: 4 coaches
- Max BLIGH: 10

**Allocation:**
- 6:15am: (3/4) × 10 = **7.5 spots**
- 6:30am: (1/4) × 10 = **2.5 spots**

### Implementation Options

#### Option 2a: Calculate in View (Dynamic)
Modify the view to calculate max capacity dynamically based on concurrent classes:

```sql
-- Pseudo-code concept
WITH concurrent_classes AS (
    SELECT 
        session_date,
        session_start,
        COUNT(*) OVER (PARTITION BY session_date, time_window) as concurrent_count,
        class_instance_capacity,
        SUM(class_instance_capacity) OVER (PARTITION BY session_date, time_window) as total_concurrent_capacity
    FROM view_data_attendance_26_mvp_csu
    WHERE gym = 'BLIGH'
)
SELECT 
    *,
    CASE 
        WHEN gym = 'BLIGH' AND concurrent_count > 1 THEN
            (class_instance_capacity / total_concurrent_capacity) × max_bligh
        ELSE max_bligh
    END as adjusted_max_capacity
FROM ...
```

#### Option 2b: Pre-calculate in Daily View
Add logic to the daily view to identify concurrent classes and allocate capacity:

```sql
-- In view_data_attendance_26_mvp_csu
WITH concurrent_sessions AS (
    SELECT 
        session_date,
        session_start,
        gym,
        -- Identify time windows (e.g., 6:15-6:45 = same window)
        CASE 
            WHEN session_start BETWEEN '06:00' AND '06:45' THEN '06:00-06:45'
            WHEN session_start BETWEEN '07:00' AND '07:45' THEN '07:00-07:45'
            -- ... etc
        END as time_window,
        class_instance_capacity,
        SUM(class_instance_capacity) OVER (
            PARTITION BY session_date, gym, time_window
        ) as total_window_capacity
    FROM ...
)
SELECT 
    *,
    CASE 
        WHEN gym = 'BLIGH' AND total_window_capacity > max_bligh THEN
            (class_instance_capacity / total_window_capacity) × max_bligh
        ELSE max_bligh
    END as adjusted_max_capacity
FROM concurrent_sessions
```

### Pros
- ✅ **Accurate**: Reflects actual capacity allocation
- ✅ **Flexible**: Adapts to different coach allocations
- ✅ **Optimized**: Uses full 10-spot capacity efficiently
- ✅ **Fair**: Allocates based on actual resources (coaches)

### Cons
- ❌ **Complex**: Requires time window detection logic
- ❌ **Maintenance**: Need to define time windows for staggered classes
- ❌ **Decimal values**: Results in 7.5, 2.5 (may need rounding)
- ❌ **Edge cases**: What if classes don't perfectly overlap?
- ❌ **Performance**: More complex queries may be slower

### When to Use
- If coach allocation varies significantly between time slots
- If you want to maximize capacity utilization
- If you have development resources for complex logic

---

## Option 3: Time-Window Based Allocation

### Approach
Define specific time windows where classes share capacity, and allocate proportionally within each window.

### Implementation
Create a configuration table for time windows:

```sql
CREATE TABLE class_time_windows (
    gym TEXT,
    window_start TIME,
    window_end TIME,
    max_capacity INTEGER,
    PRIMARY KEY (gym, window_start, window_end)
);

-- Example data
INSERT INTO class_time_windows VALUES
('BLIGH', '06:00:00', '06:45:00', 10),
('BLIGH', '07:00:00', '07:45:00', 10),
-- etc
```

Then calculate allocation:
```sql
WITH window_allocation AS (
    SELECT 
        v.*,
        ctw.max_capacity as window_max,
        SUM(v.class_instance_capacity) OVER (
            PARTITION BY v.session_date, v.gym, ctw.window_start
        ) as total_window_capacity
    FROM view_data_attendance_26_mvp_csu v
    JOIN class_time_windows ctw ON 
        v.gym = ctw.gym 
        AND v.session_start >= ctw.window_start 
        AND v.session_start < ctw.window_end
)
SELECT 
    *,
    CASE 
        WHEN total_window_capacity > window_max THEN
            (class_instance_capacity / total_window_capacity) × window_max
        ELSE window_max
    END as adjusted_max_capacity
FROM window_allocation
```

### Pros
- ✅ **Configurable**: Easy to adjust time windows
- ✅ **Flexible**: Different windows can have different max capacities
- ✅ **Clear**: Explicit definition of when classes share capacity
- ✅ **Maintainable**: Changes don't require code updates

### Cons
- ❌ **Setup required**: Need to define all time windows
- ❌ **Maintenance**: Windows need updating if schedule changes
- ❌ **Complexity**: Still requires proportional calculation logic

---

## Option 4: Coach-Based Maximum (Alternative Simple)

### Approach
Instead of a fixed gym maximum, set maximum based on typical coach allocation per time slot.

### Implementation
```sql
-- Update system_config to have per-time-slot maxes
-- Or use a rule: max = base_capacity × max_coaches_per_slot

-- Example: If max coaches per slot is typically 2.5
-- Then max = 2 × 2.5 = 5 spots per slot
```

### Pros
- ✅ **Simple**: Based on coach allocation, not time windows
- ✅ **Realistic**: Reflects actual operational capacity
- ✅ **No time logic**: Doesn't need to detect concurrent classes

### Cons
- ❌ **Static**: Doesn't adapt to actual concurrent situations
- ❌ **May not reflect sharing**: Doesn't account for staggered classes sharing pool

---

## Option 5: Hybrid Approach

### Approach
Use **Option 1 (fixed 5)** as default, but allow **manual override** for specific time slots that don't share capacity.

### Implementation
```sql
-- Add override table
CREATE TABLE capacity_overrides (
    gym TEXT,
    session_start TIME,
    max_capacity INTEGER,
    PRIMARY KEY (gym, session_start)
);

-- Default: Use 5 for BLIGH
-- Override: Specific slots can have different values
```

### Pros
- ✅ **Simple default**: Most cases use 5
- ✅ **Flexible**: Can override when needed
- ✅ **Gradual**: Can implement overrides as you discover needs

### Cons
- ❌ **Manual work**: Requires identifying which slots need overrides
- ❌ **Maintenance**: Need to keep overrides updated

---

## Recommendation Matrix

| Option | Complexity | Accuracy | Maintenance | Best For |
|--------|-----------|----------|-------------|----------|
| **Option 1: Fixed 5** | ⭐ Low | ⭐⭐ Medium | ⭐⭐⭐ Easy | Quick fix, equal allocation |
| **Option 2: Proportional** | ⭐⭐⭐ High | ⭐⭐⭐ High | ⭐⭐ Medium | Optimal utilization |
| **Option 3: Time Windows** | ⭐⭐ Medium | ⭐⭐⭐ High | ⭐⭐ Medium | Configurable, clear rules |
| **Option 4: Coach-Based** | ⭐ Low | ⭐⭐ Medium | ⭐⭐⭐ Easy | Simple, coach-focused |
| **Option 5: Hybrid** | ⭐⭐ Medium | ⭐⭐⭐ High | ⭐⭐ Medium | Balance of simple + flexible |

---

## Questions to Answer

1. **How often do staggered classes occur?**
   - If rare → Option 1 (fixed 5) is fine
   - If common → Consider Option 2 or 3

2. **Do coach allocations vary significantly?**
   - If similar → Option 1 works
   - If very different → Option 2 or 3 better

3. **What's the typical ratio?**
   - If usually 50/50 → Option 1 (5/5) is perfect
   - If varies (e.g., 75/25, 60/40) → Option 2 or 3

4. **Do you need exact accuracy or is approximation okay?**
   - Approximation → Option 1
   - Exact → Option 2 or 3

5. **How technical is your team?**
   - Less technical → Option 1 or 4
   - More technical → Option 2 or 3

---

## My Recommendation

**Start with Option 1 (Fixed 5)** because:
1. ✅ Solves the immediate problem simply
2. ✅ Easy to implement (one config change)
3. ✅ Can always upgrade to Option 2 or 3 later if needed
4. ✅ If classes are typically balanced, 5/5 split is fair

**Then consider Option 3 (Time Windows)** if:
- You find 5 is too restrictive for some slots
- You want more accuracy
- You're willing to maintain time window definitions

**Avoid Option 2 (Pure Proportional)** unless:
- You have strong SQL skills
- You need maximum accuracy
- You're willing to maintain complex logic

---

## Implementation Steps for Option 1 (Recommended)

1. **Update system_config:**
   ```sql
   UPDATE system_config 
   SET max_bligh = 5 
   WHERE match_pattern = 'PERFORM';
   ```

2. **Refresh views:**
   ```sql
   SELECT refresh_view_data_attendance_26_mvp_csu();
   SELECT refresh_view_attendance_weekly_26_mvp_csu();
   ```

3. **Verify results:**
   ```sql
   SELECT 
       gym,
       session_start,
       max_class_type_capacity,
       COUNT(*) as session_count
   FROM view_data_attendance_26_mvp_csu
   WHERE gym = 'BLIGH'
   GROUP BY gym, session_start, max_class_type_capacity
   ORDER BY session_start;
   ```

4. **Monitor for a few weeks** to see if 5 is appropriate

5. **Adjust if needed** - can always change to 6, 7, or implement Option 3

---

## Next Steps

1. **Decide which option fits your needs**
2. **Test with sample data** if going with Option 2 or 3
3. **Implement chosen option**
4. **Monitor results** and adjust as needed

Would you like me to implement Option 1, or would you prefer to explore one of the more complex options first?
