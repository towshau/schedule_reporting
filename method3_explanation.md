# Method 3: Fixed Time Buckets - Detailed Explanation

## How Method 3 Works

Method 3 uses **fixed time buckets** - it rounds class start times down to the nearest bucket interval (15 minutes, 30 minutes, etc.), then groups classes that fall into the same bucket.

---

## 15-Minute Buckets (Original Method 3)

### How It Works

Classes are rounded **DOWN** to the nearest 15-minute mark, then grouped by that bucket.

### Examples

| Actual Start Time | Rounded Down To | Bucket |
|-------------------|-----------------|--------|
| 06:10 | 06:00 | 06:00 bucket |
| 06:15 | 06:15 | 06:15 bucket |
| 06:20 | 06:15 | 06:15 bucket |
| 06:30 | 06:30 | 06:30 bucket |
| 06:44 | 06:30 | 06:30 bucket |
| 06:45 | 06:45 | 06:45 bucket |

### SQL Formula

```sql
DATE_TRUNC('hour', session_start) + 
    (EXTRACT(MINUTE FROM session_start)::int / 15) * INTERVAL '15 minutes'
```

**How it works:**
- `EXTRACT(MINUTE FROM session_start)::int / 15` = integer division (e.g., 10/15 = 0, 20/15 = 1, 44/15 = 2)
- Multiply by 15 minutes = rounds down to nearest 15-min mark

### Example Scenario

**Monday 6:10am and 6:30am classes:**
- 6:10am → rounds to **06:00** bucket
- 6:30am → rounds to **06:30** bucket
- **Result:** Different buckets = **NOT grouped together** ❌

**Monday 6:20am and 6:25am classes:**
- 6:20am → rounds to **06:15** bucket
- 6:25am → rounds to **06:15** bucket
- **Result:** Same bucket = **Grouped together** ✅

---

## 30-Minute Buckets

### How It Works

Same concept, but rounds down to the nearest 30-minute mark.

### Examples

| Actual Start Time | Rounded Down To | Bucket |
|-------------------|-----------------|--------|
| 06:10 | 06:00 | 06:00 bucket |
| 06:15 | 06:00 | 06:00 bucket |
| 06:20 | 06:00 | 06:00 bucket |
| 06:30 | 06:30 | 06:30 bucket |
| 06:44 | 06:30 | 06:30 bucket |
| 06:50 | 06:30 | 06:30 bucket |

### SQL Formula

```sql
DATE_TRUNC('hour', session_start) + 
    (EXTRACT(MINUTE FROM session_start)::int / 30) * INTERVAL '30 minutes'
```

### Example Scenario

**Monday 6:10am and 6:30am classes:**
- 6:10am → rounds to **06:00** bucket
- 6:30am → rounds to **06:30** bucket
- **Result:** Different buckets = **NOT grouped together** ❌

**Monday 6:15am and 6:20am classes:**
- 6:15am → rounds to **06:00** bucket
- 6:20am → rounds to **06:00** bucket
- **Result:** Same bucket = **Grouped together** ✅

---

## Important Limitation

**Method 3 does NOT group classes "within 30 minutes of each other"** - it groups classes that round to the **same bucket**.

### The Problem

If you have:
- 6:10am class
- 6:30am class

With 30-minute buckets:
- 6:10am → **06:00** bucket
- 6:30am → **06:30** bucket
- **They are NOT grouped** (even though they're only 20 minutes apart)

### Why This Happens

Method 3 uses **fixed buckets** that don't overlap. Each class belongs to exactly one bucket based on rounding down.

---

## Alternative: "Within X Minutes" Grouping

If you want classes **within 30 minutes of each other** to be grouped, you need a different approach:

### Method 4: Sliding Window (Within X Minutes)

**Concept:** Group classes that start within X minutes of each other, using the earliest class as the anchor.

### SQL Example

```sql
WITH class_anchors AS (
    SELECT 
        *,
        -- Find the earliest start time within 30 minutes
        MIN(session_start) OVER (
            PARTITION BY session_date, gym
            ORDER BY session_start
            RANGE BETWEEN CURRENT ROW AND INTERVAL '30 minutes' FOLLOWING
        ) as anchor_time
    FROM view_data_attendance_26_mvp_csu
)
SELECT 
    *,
    COUNT(*) OVER (
        PARTITION BY session_date, gym, anchor_time
    ) as classes_in_group
FROM class_anchors
```

### How It Works

1. **Order classes by start time**
2. **For each class:** Look ahead 30 minutes
3. **Group all classes** that start within 30 minutes of the earliest one
4. **Use the earliest start time** as the group identifier

### Example Scenario

**Monday classes:**
- 6:10am → anchor = 6:10am (earliest in next 30 min)
- 6:30am → anchor = 6:10am (within 30 min of 6:10)
- 6:50am → anchor = 6:50am (NOT within 30 min of 6:10, so new group)

**Result:**
- 6:10am and 6:30am → **Grouped together** ✅ (20 minutes apart)
- 6:50am → **Separate group** ✅

---

## Comparison: Buckets vs Sliding Window

| Scenario | 30-Min Buckets | Sliding Window (30 min) |
|----------|----------------|-------------------------|
| 6:10am + 6:30am | ❌ Not grouped (different buckets) | ✅ Grouped (within 30 min) |
| 6:15am + 6:20am | ✅ Grouped (same bucket) | ✅ Grouped (within 30 min) |
| 6:00am + 6:35am | ❌ Not grouped (different buckets) | ❌ Not grouped (35 min apart) |
| 6:10am + 6:30am + 6:50am | ❌ 6:10/6:30 separate, 6:50 separate | ✅ 6:10/6:30 grouped, 6:50 separate |

---

## Recommendation

### Use **30-Minute Buckets** if:
- ✅ You want simple, predictable grouping
- ✅ Classes typically start at :00 or :30
- ✅ You don't need exact "within X minutes" logic

### Use **Sliding Window (Within 30 Min)** if:
- ✅ You want classes within 30 minutes grouped together
- ✅ Classes start at irregular times (6:10, 6:25, 6:40)
- ✅ You need more flexible grouping

---

## Implementation Example: Sliding Window

```sql
WITH time_groups AS (
    SELECT 
        *,
        -- Find anchor time (earliest class within 30 min window)
        MIN(session_start) OVER (
            PARTITION BY session_date, gym
            ORDER BY session_start
            RANGE BETWEEN CURRENT ROW AND INTERVAL '30 minutes' FOLLOWING
        ) as group_anchor,
        -- Count classes in the same group
        COUNT(*) OVER (
            PARTITION BY session_date, gym,
            MIN(session_start) OVER (
                PARTITION BY session_date, gym
                ORDER BY session_start
                RANGE BETWEEN CURRENT ROW AND INTERVAL '30 minutes' FOLLOWING
            )
        ) as classes_in_group
    FROM view_data_attendance_26_mvp_csu
    WHERE gym = 'BLIGH'
)
SELECT 
    *,
    CASE 
        WHEN classes_in_group > 1 THEN
            -- Staggered: proportional split
            (class_instance_capacity::numeric / 
             SUM(class_instance_capacity) OVER (
                 PARTITION BY session_date, gym, group_anchor
             )::numeric) * max_class_type_capacity
        ELSE
            -- Not staggered: full max
            max_class_type_capacity
    END as adjusted_max_capacity
FROM time_groups
```

---

## Summary

**Method 3 (Buckets):**
- Groups classes that round to the **same bucket**
- 6:10am and 6:30am → **NOT grouped** (different buckets)
- Simple but may miss classes that are close together

**Sliding Window (Within X Minutes):**
- Groups classes that start **within X minutes of each other**
- 6:10am and 6:30am → **Grouped** (20 minutes apart)
- More flexible but slightly more complex

**For staggered classes, I recommend the Sliding Window approach** because it better captures classes that actually run at similar times, even if they don't round to the same bucket.

