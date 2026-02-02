# Proportional Split Calculation - Detailed Explanation

## How the Proportional Split Works

The proportional split is based on **`class_instance_capacity`**, which is calculated from the **number of coaches**.

---

## Step-by-Step Calculation

### Step 1: Calculate `class_instance_capacity` (Based on Coaches)

**Formula:**
```
class_instance_capacity = class_type_capacity × number_of_coaches
```

**Where:**
- `class_type_capacity` = base capacity per coach (from system_config, typically 2 for PERFORM)
- `number_of_coaches` = count of coaches assigned to that session

**Example:**
- 6:10am class: 2 coaches → `class_instance_capacity = 2 × 2 = 4 spots`
- 6:30am class: 1 coach → `class_instance_capacity = 2 × 1 = 2 spots`

### Step 2: Calculate Total Block Capacity

**Formula:**
```
total_block_capacity = SUM(class_instance_capacity) for all classes in the same block
```

**Example:**
- Block 1 (6:10am - 6:50am):
  - 6:10am: 4 spots (2 coaches)
  - 6:30am: 2 spots (1 coach)
  - **Total: 6 spots**

### Step 3: Calculate Proportional Split

**Formula:**
```
adjusted_max_capacity = (class_instance_capacity / total_block_capacity) × max_class_type_capacity
```

**Example:**
- Max capacity (BLIGH): 10 spots
- 6:10am: (4 / 6) × 10 = **6.67 spots**
- 6:30am: (2 / 6) × 10 = **3.33 spots**
- **Total: 10 spots** ✅

---

## Complete Example

### Scenario
- **Gym:** BLIGH
- **Block 1:** 6:10am - 6:50am
- **Max capacity:** 10 spots
- **Base capacity per coach:** 2 spots

### Classes in Block 1

**6:10am class:**
- Coaches: 2 coaches
- `class_instance_capacity = 2 × 2 = 4 spots`

**6:30am class:**
- Coaches: 1 coach
- `class_instance_capacity = 2 × 1 = 2 spots`

### Calculation

**Step 1: Total Block Capacity**
```
total_block_capacity = 4 + 2 = 6 spots
```

**Step 2: Proportional Split**
```
6:10am adjusted_max = (4 / 6) × 10 = 6.67 spots
6:30am adjusted_max = (2 / 6) × 10 = 3.33 spots
```

**Result:**
- 6:10am gets **6.67 spots** (proportional to its 2 coaches)
- 6:30am gets **3.33 spots** (proportional to its 1 coach)
- Total: **10 spots** (the max capacity is fully allocated)

---

## Why Based on Coaches?

**The split is proportional to coaches because:**
1. `class_instance_capacity` = coaches × base capacity
2. More coaches = more current capacity = gets more of the max capacity
3. This is fair because classes with more coaches are using more resources

### Example: Different Coach Allocations

**Scenario A: Equal coaches**
- 6:10am: 2 coaches = 4 spots
- 6:30am: 2 coaches = 4 spots
- Total: 8 spots
- Split: 50/50 → Each gets 5 spots

**Scenario B: Unequal coaches**
- 6:10am: 3 coaches = 6 spots
- 6:30am: 1 coach = 2 spots
- Total: 8 spots
- Split: 75/25 → 6:10am gets 7.5 spots, 6:30am gets 2.5 spots

**Scenario C: Very unequal**
- 6:10am: 3 coaches = 6 spots
- 6:30am: 1 coach = 2 spots
- Total: 8 spots (but max is 10)
- Split: 75/25 → 6:10am gets 7.5 spots, 6:30am gets 2.5 spots
- **Note:** Total is 10, which is the max capacity

---

## SQL Formula Breakdown

```sql
-- Proportional allocation formula
(class_instance_capacity::numeric / total_block_capacity::numeric) * max_class_type_capacity
```

**What each part means:**
- `class_instance_capacity` = This class's capacity (coaches × base)
- `total_block_capacity` = Sum of all classes' capacities in the block
- `max_class_type_capacity` = Maximum capacity to split (e.g., 10 for BLIGH)

**The ratio:**
- `class_instance_capacity / total_block_capacity` = This class's share (0.0 to 1.0)
- Multiply by max → This class's portion of the max capacity

---

## Visual Example

```
Block 1: 6:10am - 6:50am
Max Capacity Pool: 10 spots

6:10am class:  ████████░░  (4 spots current, gets 6.67 of 10 max)
6:30am class:  ██░░░░░░░░  (2 spots current, gets 3.33 of 10 max)
              ──────────
Total:         ██████████  (6 spots current, 10 spots max allocated)
```

**Proportion:**
- 6:10am: 4/6 = 66.7% → Gets 66.7% of 10 = 6.67 spots
- 6:30am: 2/6 = 33.3% → Gets 33.3% of 10 = 3.33 spots

---

## Key Points

1. **Based on `class_instance_capacity`** (which = coaches × base capacity)
2. **Proportional to coaches** - More coaches = larger share
3. **Always sums to max capacity** - Total allocation = max_class_type_capacity
4. **Only applies when:**
   - BLIGH gym
   - Block 0-11 (5:30am - 1:30pm)
   - Multiple classes in same block
   - Total block capacity exceeds max

---

## Summary

**The proportional split is:**
- ✅ Based on `class_instance_capacity` (coaches × base capacity)
- ✅ Proportional to the number of coaches
- ✅ Fair allocation based on resource usage
- ✅ Formula: `(this_class_capacity / total_block_capacity) × max_capacity`

**Example:**
- 2 coaches vs 1 coach → 2:1 ratio → 66.7% vs 33.3% of max capacity

