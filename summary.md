# Staggered Capacity Calculation - Summary

## The Problem

When multiple classes run at the same time (staggered), they share the same physical space and maximum capacity.

**Example:**
- 6:10am class + 6:30am class = both in the gym at the same time
- They share the 10-spot maximum capacity for BLIGH gym
- Need to split capacity proportionally based on coaches

---

## The Solution

### Step 1: Group Classes into 40-Minute Blocks

```
5:30am ──────────────────────────────────────────────────────────────── 1:30pm
       │                                                               │
       ▼                                                               ▼
   STAGGERED LOGIC APPLIES                                    FULL MAX CAPACITY
                                             
```

**Block Calculation:**
```
Block Number = FLOOR((session_start - 5:30am) / 40 minutes)
```

**Visual Timeline:**
```
Block 0:  [5:30──────6:10]  ✅ Staggered
Block 1:           [6:10──────6:50]  ✅ Staggered  ← 6:10am + 6:30am grouped
Block 2:                     [6:50──────7:30]  ✅ Staggered  ← 6:50am + 7:10am grouped
...
Block 11:                                                                  [12:50──────1:30]  ✅ Staggered
Block 12:                                                                        [1:30──────2:10]  ❌ Full max
```

---

## The Formula

### Proportional Split

When multiple classes share a block, split capacity based on coaches:

```
adjusted_max_capacity = (class_instance_capacity / total_block_capacity) × max_capacity
```

**Where:**
- `class_instance_capacity` = base_capacity × number_of_coaches
- `total_block_capacity` = sum of all class_instance_capacity in the same block
- `max_capacity` = gym maximum (10 for BLIGH)

---

## Example Calculation

### Scenario: Block 1 (6:10am - 6:50am)

**Classes:**
- 6:10am: 3 coaches → 6 spots capacity
- 6:30am: 1 coach → 2 spots capacity

**Staggered Capacity Calculation:**
```
Total block capacity = 6 + 2 = 8 spots
Max capacity = 10 spots (from system_config: max_bligh = 10)

6:10am adjusted = (6 / 8) × 10 = 7.5 spots  ✅
6:30am adjusted = (2 / 8) × 10 = 2.5 spots  ✅
Total = 10 spots (matches max) ✅
```

**Visual:**
```
┌─────────────────────────────────────┐
│  BLIGH Gym - Max 10 Spots          │
├─────────────────────────────────────┤
│  6:10am class (3 coaches)           │
│  Gets: 7.5 spots (75%)              │
│  ████████████████░░░░              │
├─────────────────────────────────────┤
│  6:30am class (1 coach)             │
│  Gets: 2.5 spots (25%)              │
│  ████░░░░░░░░░░░░░░░░              │
└─────────────────────────────────────┘
```

### Complete Weekly Example 1: 6:10am Class (Moderate Attendance)

**Scenario:**
- **Time:** 6:10am class at BLIGH gym
- **Original capacity:** 6 spots/day (3 coaches × 2 spots/coach)
- **Staggered context:** Shares Block 1 with 6:30am class
  - 6:10am: 3 coaches → 6 spots original capacity
  - 6:30am: 1 coach → 2 spots original capacity
  - Total block capacity: 8 spots
- **Adjusted max capacity:** 7.5 spots (from staggered calculation: (6/8) × 10)
- **Operational days:** 5 (Mon-Fri)
- **Total attendance:** 15 people this week across all 5 days
- **System Config:**
  - `lc_buffer = 0.9`
  - `avg_perform_consumption = 0.4`

**Step-by-Step:**

```
1. Total capacity for week = 6 × 5 = 30 spots
   (Original capacity: 6 spots/day × 5 days)

2. Utilization rate = 15 / 30 = 0.50 (50% full)
   (15 people attended out of 30 total spots available)

3. Average spots = 30 / 5 = 6.0 spots/day
   (Average capacity per day)

4. Sale without change:
   = ((0.9 - 0.50) × 6.0) / 0.4
   = (0.40 × 6.0) / 0.4
   = 2.4 / 0.4
   = 6 members
   (Can add 6 members using existing 6-spot capacity)

5. Sale add change:
   = 6 + ((7.5 - 6) × 5 / 0.4)
   = 6 + (1.5 × 5 / 0.4)
   = 6 + (7.5 / 0.4)
   = 6 + 18.75
   = 24.75 members
   (Can add 6 at current capacity + 18.75 if expand to staggered max of 7.5)
```

**Key Points:**
- Original capacity: **6 spots/day** (3 coaches)
- Adjusted max (staggered): **7.5 spots/day** (75% of 10 max, shared with 6:30am)
- The 6:30am class gets **2.5 spots/day** (25% of 10 max)
- Combined: 7.5 + 2.5 = **10 spots total** (matches BLIGH max)

---

### Complete Weekly Example 2: Nearly Full Staggered Classes

**Scenario:**
- **Time:** 6:15am and 6:30am classes at BLIGH gym (Block 1)
- **Original capacities:**
  - 6:15am: 3 coaches → 6 spots/day
  - 6:30am: 2 coaches → 4 spots/day
  - Total block capacity: 10 spots
- **Adjusted max capacities (staggered):**
  - 6:15am: (6/10) × 10 = **6.0 spots/day**
  - 6:30am: (4/10) × 10 = **4.0 spots/day**
  - Total: 10 spots (matches max)
- **Attendance (all 5 days):**
  - 6:15am: 5 people per day (5/6 = 83% full)
  - 6:30am: 3 people per day (3/4 = 75% full)
- **Operational days:** 5 (Mon-Fri)
- **System Config:**
  - `lc_buffer = 0.9`
  - `avg_perform_consumption = 0.4`

**6:15am Class Calculation:**

```
1. Total capacity for week = 6 × 5 = 30 spots
   (Original capacity: 6 spots/day × 5 days)

2. Total attendance = 5 × 5 = 25 people
   (5 people per day × 5 days)

3. Utilization rate = 25 / 30 = 0.833 (83.3% full)

4. Average spots = 30 / 5 = 6.0 spots/day

5. Sale without change:
   = ((0.9 - 0.833) × 6.0) / 0.4
   = (0.067 × 6.0) / 0.4
   = 0.402 / 0.4
   = 1.005 members
   (Can add ~1 member using existing capacity)

6. Sale add change:
   = 1.005 + ((6.0 - 6) × 5 / 0.4)
   = 1.005 + (0 × 5 / 0.4)
   = 1.005 + 0
   = 1.005 members
   (No expansion possible - already at adjusted max of 6.0)
```

**6:30am Class Calculation:**

```
1. Total capacity for week = 4 × 5 = 20 spots
   (Original capacity: 4 spots/day × 5 days)

2. Total attendance = 3 × 5 = 15 people
   (3 people per day × 5 days)

3. Utilization rate = 15 / 20 = 0.75 (75% full)

4. Average spots = 20 / 5 = 4.0 spots/day

5. Sale without change:
   = ((0.9 - 0.75) × 4.0) / 0.4
   = (0.15 × 4.0) / 0.4
   = 0.6 / 0.4
   = 1.5 members
   (Can add 1.5 members using existing capacity)

6. Sale add change:
   = 1.5 + ((4.0 - 4) × 5 / 0.4)
   = 1.5 + (0 × 5 / 0.4)
   = 1.5 + 0
   = 1.5 members
   (No expansion possible - already at adjusted max of 4.0)
```

**Key Points:**
- Both classes are **nearly full** (83% and 75% utilization)
- Both are already at their **adjusted max capacity** (6.0 and 4.0)
- Combined: 6.0 + 4.0 = **10 spots total** (matches BLIGH max)
- **No expansion room** - the staggered system has allocated all 10 spots
- Can only add members using existing capacity gaps (1-1.5 members each)

---

## When It Applies

### ✅ Staggered Logic (Proportional Split)

**Conditions:**
- Gym = BLIGH
- Block number < 12 (5:30am - 1:30pm)
- Multiple classes in same block

**Result:** Proportional split based on coaches

### ❌ Full Max Capacity

**Conditions:**
- Gym ≠ BLIGH (BRIDGE, COLLIN)
- Block number ≥ 12 (after 1:30pm)
- Single class in block

**Result:** Full maximum capacity

---

## Data Flow

```
┌─────────────────────┐
│  Raw Data            │
│  (member_daily_...)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Daily View         │
│  ─────────────────  │
│  1. Group classes   │
│  2. Calculate blocks│
│  3. Find concurrent │
│  4. Split capacity  │
│                     │
│  OUTPUT:            │
│  adjusted_max_cap   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Weekly View         │
│  ─────────────────  │
│  1. Aggregate days  │
│  2. MAX(adjusted)    │
│  3. Calculate sales │
│                     │
│  OUTPUT:            │
│  sale_add_change    │
└─────────────────────┘
```

---

## System Config Values

These values come from the `system_config` table and are used throughout calculations:

| Config Key | Value | What It Means |
|------------|-------|---------------|
| **lc_buffer** | 0.9 | Target utilization (90% full) - cancellation buffer |
| **avg_perform_consumption** | 0.4 | Average sessions per member per week (members attend ~2.5x/week) |
| **base_capacity** (PERFORM) | 2 | Spots per coach for PERFORM classes |
| **max_bligh** | 10 | Maximum spots per day for BLIGH gym |
| **max_bridge** | 8 | Maximum spots per day for BRIDGE gym |
| **max_collin** | 6 | Maximum spots per day for COLLIN gym |

---

## Key Equations

### 1. Block Number
```
block_number = FLOOR((session_start - 05:30:00) / 2400 seconds)
```
*2400 seconds = 40 minutes*

### 2. Class Instance Capacity
```
class_instance_capacity = base_capacity × number_of_coaches
```
*Example: 2 spots/coach × 3 coaches = 6 spots*

### 3. Utilization Rate
```
utilization_rate = total_attendance / total_capacity_for_week
```
*Example: 27 attendees / 30 spots = 0.90 (90% full)*

### 4. Average Spots
```
average_spots = total_capacity_for_week / operational_days
```
*Example: 30 spots / 5 days = 6.0 spots per day*

### 5. Adjusted Max Capacity (Staggered)
```
IF (BLIGH AND block < 12 AND multiple_classes):
    adjusted = (class_capacity / total_block_capacity) × max_capacity
ELSE:
    adjusted = max_capacity
```

### 6. Sale Without Change
```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
```
*Uses existing capacity to reach 90% utilization*

**Example:**
```
= ((0.9 - 0.1) × 4.0) / 0.4
= (0.8 × 4.0) / 0.4
= 8 members
```

### 7. Sale Add Change (Weekly)
```
sale_add_change = sale_without_change + 
                  ((adjusted_max_capacity - current_capacity) × 
                   operational_days / avg_perform_consumption)
```
*Existing capacity + expansion potential*

**Example:**
```
= 8 + ((10 - 4) × 5 / 0.4)
= 8 + (6 × 5 / 0.4)
= 8 + 75
= 83 members
```

---

## Quick Reference

| Scenario | Gym | Block | Classes | Result |
|----------|-----|-------|---------|--------|
| 6:10am + 6:30am | BLIGH | 1 | 2 | Proportional split |
| 6:50am alone | BLIGH | 2 | 1 | Full max (10) |
| 3:55pm | BLIGH | 15 | 1 | Full max (10) |
| 6:15am | BRIDGE | - | 1 | Full max (8) |

---

---

## Complete Calculation Flow

### Daily View Calculations

```
1. Calculate block_number
   → FLOOR((session_start - 5:30am) / 40 min)

2. Calculate class_instance_capacity
   → base_capacity (2) × number_of_coaches

3. Find concurrent classes
   → SUM(class_instance_capacity) OVER (same block)

4. Calculate adjusted_max_capacity
   → IF staggered: (this_class / total_block) × max_capacity
   → ELSE: max_capacity
```

### Weekly View Calculations

```
1. Aggregate daily data
   → MAX(adjusted_max_capacity)
   → MAX(class_instance_capacity)
   → SUM(actual_attendance)
   → COUNT(DISTINCT session_date) = operational_days

2. Calculate utilization_rate
   → total_attendance / total_capacity_for_week

3. Calculate average_spots
   → total_capacity_for_week / operational_days

4. Calculate sale_without_change
   → ((0.9 - utilization_rate) × average_spots) / 0.4

5. Calculate sale_add_change
   → sale_without_change + 
     ((adjusted_max_capacity - current_capacity) × 
      operational_days / 0.4)
```

---

## Summary

**What:** Staggered classes share maximum capacity proportionally based on coaches

**When:** BLIGH gym, blocks 0-11 (5:30am - 1:30pm), multiple classes

**How:** `(this_class / total_in_block) × max_capacity`

**System Config:**
- `lc_buffer = 0.9` (90% target utilization)
- `avg_perform_consumption = 0.4` (members attend ~2.5x/week)

**Result:** Fair allocation that uses full capacity efficiently

