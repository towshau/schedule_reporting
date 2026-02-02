# 40-Minute Blocks Starting from 5:30am - Full Day Map

## Overview

This document shows how 40-minute blocks starting from **5:30am** map across the entire day. Classes that start within the same 40-minute block are considered "staggered" and share capacity.

---

## Full Day Block Map

**Important:** Staggered capacity calculation only applies to **Blocks 0-11** (5:30am - 1:30pm). After 1:30pm, use full maximum capacity (no staggered grouping).

| Block # | Block Start | Block End | Duration | Staggered? | Example Classes in This Block |
|---------|-------------|-----------|----------|------------|-------------------------------|
| **0** | 5:30 AM | 6:10 AM | 40 min | ✅ Yes | 5:30am |
| **1** | 6:10 AM | 6:50 AM | 40 min | ✅ Yes | 6:10am, 6:30am |
| **2** | 6:50 AM | 7:30 AM | 40 min | ✅ Yes | 6:50am, 7:10am |
| **3** | 7:30 AM | 8:10 AM | 40 min | ✅ Yes | 7:30am, 7:50am |
| **4** | 8:10 AM | 8:50 AM | 40 min | ✅ Yes | 8:10am, 8:30am |
| **5** | 8:50 AM | 9:30 AM | 40 min | ✅ Yes | 8:50am, 9:10am |
| **6** | 9:30 AM | 10:10 AM | 40 min | ✅ Yes | 9:30am, 10:10am |
| **7** | 10:10 AM | 10:50 AM | 40 min | ✅ Yes | 10:10am, 10:50am |
| **8** | 10:50 AM | 11:30 AM | 40 min | ✅ Yes | 10:50am, 11:30am |
| **9** | 11:30 AM | 12:10 PM | 40 min | ✅ Yes | 11:30am, 11:50am |
| **10** | 12:10 PM | 12:50 PM | 40 min | ✅ Yes | 12:10pm, 12:30pm |
| **11** | 12:50 PM | 1:30 PM | 40 min | ✅ Yes | 12:50pm, 1:10pm |
| **12** | 1:30 PM | 2:10 PM | 40 min | ❌ **No** | 1:30pm, 2:10pm (use full max) |
| **13** | 2:10 PM | 2:50 PM | 40 min | ❌ **No** | 2:10pm (use full max) |
| **14** | 2:50 PM | 3:30 PM | 40 min | ❌ **No** | *(typically no classes)* |
| **15** | 3:30 PM | 4:10 PM | 40 min | ❌ **No** | 3:55pm (use full max) |
| **16** | 4:10 PM | 4:50 PM | 40 min | ❌ **No** | 4:35pm (use full max) |
| **17** | 4:50 PM | 5:30 PM | 40 min | ❌ **No** | 5:15pm (use full max) |
| **18** | 5:30 PM | 6:10 PM | 40 min | ❌ **No** | 5:55pm (use full max) |
| **19** | 6:10 PM | 6:50 PM | 40 min | ❌ **No** | 6:35pm (use full max) |
| **20** | 6:50 PM | 7:30 PM | 40 min | ❌ **No** | *(typically no classes)* |
| **21** | 7:30 PM | 8:10 PM | 40 min | ❌ **No** | *(typically no classes)* |
| **22** | 8:10 PM | 8:50 PM | 40 min | ❌ **No** | *(typically no classes)* |
| **23** | 8:50 PM | 9:30 PM | 40 min | ❌ **No** | *(typically no classes)* |

---

## Key Staggered Class Scenarios

### Morning Peak (6:10am - 6:50am Block)
**Block 1: 6:10 AM - 6:50 AM**

Classes that would be grouped together:
- ✅ 6:10am class
- ✅ 6:30am class

**Example:**
- 6:10am: 3 coaches = 6 spots
- 6:30am: 1 coach = 2 spots
- **Total:** 8 spots sharing 10 max capacity
- **Allocation:**
  - 6:10am: (6/8) × 10 = **7.5 spots**
  - 6:30am: (2/8) × 10 = **2.5 spots**

### Early Morning (6:50am - 7:30am Block)
**Block 2: 6:50 AM - 7:30 AM**

Classes that would be grouped together:
- ✅ 6:50am class
- ✅ 7:10am class

### Mid-Morning (7:30am - 8:10am Block)
**Block 3: 7:30 AM - 8:10 AM**

Classes that would be grouped together:
- ✅ 7:30am class
- ✅ 7:50am class

### Late Morning (8:10am - 8:50am Block)
**Block 4: 8:10 AM - 8:50 AM**

Classes that would be grouped together:
- ✅ 8:10am class
- ✅ 8:30am class

### Afternoon Peak (12:10pm - 12:50pm Block)
**Block 10: 12:10 PM - 12:50 PM**

Classes that would be grouped together:
- ✅ 12:10pm class
- ✅ 12:30pm class

### Evening Peak (5:30pm - 6:10pm Block)
**Block 18: 5:30 PM - 6:10 PM**

Classes that would be grouped together:
- ✅ 5:55pm class
- *(and any other classes starting between 5:30-6:10pm)*

---

## SQL Implementation

### Method: Fixed 40-Minute Blocks from 5:30am (Blocks 0-11 Only)

```sql
WITH time_blocks AS (
    SELECT 
        *,
        -- Calculate which 40-minute block this class falls into
        -- Starting from 5:30am, each block is 40 minutes
        FLOOR(
            EXTRACT(EPOCH FROM (session_start - '05:30:00'::time)) / 2400  -- 2400 seconds = 40 minutes
        )::int as block_number,
        '05:30:00'::time + 
        (FLOOR(
            EXTRACT(EPOCH FROM (session_start - '05:30:00'::time)) / 2400
        )::int * INTERVAL '40 minutes') as block_start
    FROM view_data_attendance_26_mvp_csu
    WHERE gym = 'BLIGH'
),
concurrent_capacity AS (
    SELECT 
        *,
        -- Sum of all capacities in the same block
        SUM(class_instance_capacity) OVER (
            PARTITION BY session_date, gym, block_start
        ) as total_block_capacity,
        -- Count of classes in the same block
        COUNT(*) OVER (
            PARTITION BY session_date, gym, block_start
        ) as classes_in_block
    FROM time_blocks
)
SELECT 
    *,
    CASE 
        -- Only apply staggered logic to blocks 0-11 (5:30am - 1:30pm)
        -- Blocks 12+ (after 1:30pm) use full max capacity
        WHEN gym = 'BLIGH' 
         AND block_number < 12  -- Only blocks 0-11
         AND classes_in_block > 1 
         AND total_block_capacity > max_class_type_capacity THEN
            -- Proportional allocation (staggered)
            (class_instance_capacity::numeric / total_block_capacity::numeric) * max_class_type_capacity
        ELSE
            -- Use full max capacity (not staggered, or block 12+)
            max_class_type_capacity
    END as adjusted_max_capacity
FROM concurrent_capacity
```

### Alternative: Simpler Bucket Calculation (Blocks 0-11 Only)

```sql
-- Simpler approach: Calculate block number, then convert back to time
WITH block_calculation AS (
    SELECT 
        *,
        -- Calculate which block number (0, 1, 2, 3, ...)
        FLOOR(
            EXTRACT(EPOCH FROM (session_start - '05:30:00'::time)) / 2400
        )::int as block_number
    FROM view_data_attendance_26_mvp_csu
    WHERE gym = 'BLIGH'
),
concurrent_capacity AS (
    SELECT 
        *,
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
    CASE 
        -- Only apply staggered logic to blocks 0-11 (5:30am - 1:30pm)
        WHEN gym = 'BLIGH' 
         AND block_number < 12  -- Only blocks 0-11
         AND classes_in_block > 1 
         AND total_block_capacity > max_class_type_capacity THEN
            (class_instance_capacity::numeric / total_block_capacity::numeric) * max_class_type_capacity
        ELSE
            -- Use full max capacity (not staggered, or block 12+)
            max_class_type_capacity
    END as adjusted_max_capacity
FROM concurrent_capacity
```

---

## Visual Timeline

```
5:30am ──────────────────────────────────────────────────────────────── 10:00pm

Block 0:  [5:30──────6:10]
Block 1:           [6:10──────6:50]  ← 6:10am + 6:30am grouped
Block 2:                     [6:50──────7:30]  ← 6:50am + 7:10am grouped
Block 3:                               [7:30──────8:10]  ← 7:30am + 7:50am grouped
Block 4:                                         [8:10──────8:50]  ← 8:10am + 8:30am grouped
Block 5:                                                   [8:50──────9:30]  ← 8:50am + 9:10am grouped
Block 6:                                                             [9:30──────10:10]
Block 7:                                                                       [10:10──────10:50]
Block 8:                                                                                 [10:50──────11:30]
Block 9:                                                                                           [11:30──────12:10]
Block 10:                                                                                                    [12:10──────12:50]  ← 12:10pm + 12:30pm grouped
Block 11:                                                                                                              [12:50──────1:30]  ← 12:50pm + 1:10pm grouped
Block 12:                                                                                                                        [1:30──────2:10]
Block 13:                                                                                                                                  [2:10──────2:50]
Block 14:                                                                                                                                            [2:50──────3:30]
Block 15:                                                                                                                                                      [3:30──────4:10]  ← 3:55pm
Block 16:                                                                                                                                                                [4:10──────4:50]  ← 4:35pm
Block 17:                                                                                                                                                                          [4:50──────5:30]
Block 18:                                                                                                                                                                                    [5:30──────6:10]  ← 5:55pm
Block 19:                                                                                                                                                                                              [6:10──────6:50]  ← 6:35pm
```

---

## Key Points

### ✅ Staggered Capacity Rules

**Blocks 0-11 (5:30am - 1:30pm):**
- ✅ Apply staggered capacity calculation
- ✅ Group classes in same block, split proportionally if total exceeds max
- ✅ Example: 6:10am + 6:30am in Block 1 → split 10 max capacity proportionally

**Blocks 12+ (After 1:30pm):**
- ❌ **No staggered calculation**
- ✅ Use full maximum capacity for each class
- ✅ Example: 3:55pm class → gets full 10 max capacity (no sharing)

### ✅ Advantages of 40-Minute Blocks

1. **Covers Common Staggered Times (Morning/Afternoon Peak)**
   - 6:10am + 6:30am → Same block ✅ (Block 1)
   - 6:50am + 7:10am → Same block ✅ (Block 2)
   - 7:30am + 7:50am → Same block ✅ (Block 3)
   - 8:10am + 8:30am → Same block ✅ (Block 4)
   - 12:10pm + 12:30pm → Same block ✅ (Block 10)

2. **Simple Logic**
   - Easy to calculate which block a class belongs to
   - No complex time window definitions needed
   - Predictable grouping

3. **Good Coverage**
   - 40 minutes is long enough to catch staggered classes
   - Not so long that unrelated classes get grouped
   - Only applies during peak hours (morning/early afternoon)

### ⚠️ Edge Cases

1. **Classes at Block Boundaries**
   - 6:10am class → Block 1
   - 6:50am class → Block 2
   - **These are NOT grouped** (different blocks)
   - If you have 6:10am and 6:50am staggered, they won't be grouped

2. **Very Close Classes**
   - 6:09am and 6:11am → Different blocks (6:09 in Block 0, 6:11 in Block 1)
   - **Solution:** Use sliding window approach instead

---

## Recommendation

**40-minute blocks (Blocks 0-11 only) work well because:**
- ✅ Covers all morning and early afternoon staggered classes
- ✅ Evening classes (after 1:30pm) use full max capacity (simpler)
- ✅ Gaps in evening schedule are large enough that staggered logic isn't needed
- ✅ Simple, predictable grouping during peak hours

**Implementation:**
- Apply staggered calculation: **Blocks 0-11** (5:30am - 1:30pm)
- Use full max capacity: **Blocks 12+** (after 1:30pm)

---

## Next Steps

1. **Test with your actual class times** - Do most staggered classes fall within the same 40-minute block?
2. **Check edge cases** - Are there any staggered classes that would be split across blocks?
3. **Implement in daily view** - Use the SQL provided above
4. **Monitor results** - Verify the grouping makes sense for your schedule

