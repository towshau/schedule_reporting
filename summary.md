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
   (Blocks 0-11)                                              (Blocks 12+)
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

**Calculation:**
```
Total block capacity = 6 + 2 = 8 spots
Max capacity = 10 spots

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

## Key Equations

### 1. Block Number
```
block_number = FLOOR((session_start - 05:30:00) / 2400 seconds)
```

### 2. Class Instance Capacity
```
class_instance_capacity = base_capacity × number_of_coaches
```

### 3. Adjusted Max Capacity (Staggered)
```
IF (BLIGH AND block < 12 AND multiple_classes):
    adjusted = (class_capacity / total_block_capacity) × max_capacity
ELSE:
    adjusted = max_capacity
```

### 4. Sale Add Change (Weekly)
```
sale_add_change = sale_without_change + 
                  ((adjusted_max_capacity - current_capacity) × 
                   operational_days / avg_perform_consumption)
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

## Summary

**What:** Staggered classes share maximum capacity proportionally based on coaches

**When:** BLIGH gym, blocks 0-11 (5:30am - 1:30pm), multiple classes

**How:** `(this_class / total_in_block) × max_capacity`

**Result:** Fair allocation that uses full capacity efficiently

