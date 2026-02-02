# Sale Capacity Formula Explanation

## What These Formulas Do

These formulas answer a simple question: **"How many new members can we add to this class?"**

They calculate this in two scenarios:
1. **Without changing capacity** - using the space we already have
2. **With capacity expansion** - if we increase to maximum capacity

---

## The Formulas

### Formula 1: Sale Without Change

```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
```

**In plain English:** 
- How many members can we add using the existing capacity?
- Uses the gap between current utilization and our target (90%)

### Formula 2: Sale Add Change

```
sale_add_change = sale_without_change + ((max_capacity - current_capacity) × operational_days / avg_perform_consumption)
```

**In plain English:**
- Members we can add at current capacity **PLUS**
- Members we can add if we expand capacity to the maximum

---

## Key Terms Explained Simply

| Term | What It Means | Example |
|------|---------------|---------|
| **current_capacity** | Spots configured right now | 4 spots per day |
| **max_capacity** | Maximum spots possible | 10 spots per day |
| **utilization_rate** | How full the class is (0.0 to 1.0+) | 0.9 = 90% full |
| **operational_days** | Days the class runs (Mon-Fri) | 5 days |
| **avg_perform_consumption** | Average sessions per member per week | 0.4 (members attend ~2.5 times per week) |
| **lc_buffer** | Target utilization (90%) | 0.9 |
| **average_spots** | Average spots available per day | 6.0 spots |

---

## Real Example: Full Class

### The Situation
- **Gym:** BRIDGE
- **Current capacity:** 6 spots per day
- **Max capacity:** 8 spots per day
- **Days running:** 5 days (Mon-Fri)
- **Attendance:** 27 people attended this week
- **Utilization:** 90% full (27 out of 30 total spots)
- **System Config Values:**
  - **lc_buffer:** 0.9 (target utilization from system_config)
  - **avg_perform_consumption:** 0.4 (average sessions per member per week from system_config)

### Step-by-Step Calculation

#### Step 1: Calculate Utilization
```
utilization_rate = 27 attendees / 30 total spots = 0.90 (90% full)
```

#### Step 2: Calculate Sale Without Change
```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
                    = ((0.9 - 0.9) × 6.0) / 0.4
                    = (0.0 × 6.0) / 0.4
                    = 0 members
```

**Where the numbers come from:**
- `lc_buffer = 0.9` (from system_config - target utilization)
- `utilization_rate = 0.9` (calculated: 27/30)
- `average_spots = 6.0` (current capacity per day)
- `avg_perform_consumption = 0.4` (from system_config - average sessions per member)

**Result:** 0 members can be added at current capacity (already at 90% target)

#### Step 3: Calculate Sale Add Change
```
capacity_increase = max_capacity - current_capacity
                  = 8 - 6 = 2 spots per day

additional_spots_per_week = capacity_increase × operational_days
                          = 2 × 5 days = 10 spots

additional_members = additional_spots_per_week / avg_perform_consumption
                   = 10 / 0.4 = 25 members

sale_add_change = sale_without_change + additional_members
                = 0 + 25 = 25 members
```

**Where the numbers come from:**
- `max_capacity = 8` (maximum spots possible for BRIDGE PERFORM)
- `current_capacity = 6` (current configured spots)
- `operational_days = 5` (Mon-Fri)
- `avg_perform_consumption = 0.4` (from system_config)

**Result:** If we expand from 6 to 8 spots per day, we can add **25 new members**

---

## Real Example: Empty Class

### The Situation
- **Gym:** BLIGH
- **Current capacity:** 4 spots per day
- **Max capacity:** 10 spots per day
- **Days running:** 5 days (Mon-Fri)
- **Attendance:** 2 people attended this week
- **Utilization:** 10% full (2 out of 20 total spots)
- **System Config Values:**
  - **lc_buffer:** 0.9 (target utilization from system_config)
  - **avg_perform_consumption:** 0.4 (average sessions per member per week from system_config)

### Step-by-Step Calculation

#### Step 1: Calculate Utilization
```
utilization_rate = 2 attendees / 20 total spots = 0.10 (10% full)
```

#### Step 2: Calculate Sale Without Change
```
sale_without_change = ((lc_buffer - utilization_rate) × average_spots) / avg_perform_consumption
                    = ((0.9 - 0.1) × 4.0) / 0.4
                    = (0.8 × 4.0) / 0.4
                    = 3.2 / 0.4
                    = 8 members
```

**Where the numbers come from:**
- `lc_buffer = 0.9` (from system_config - target utilization)
- `utilization_rate = 0.1` (calculated: 2/20)
- `average_spots = 4.0` (current capacity per day)
- `avg_perform_consumption = 0.4` (from system_config - average sessions per member)

**Result:** 8 members can be added at current capacity before reaching 90% utilization

#### Step 3: Calculate Sale Add Change
```
capacity_increase = max_capacity - current_capacity
                  = 10 - 4 = 6 spots per day

additional_spots_per_week = capacity_increase × operational_days
                          = 6 × 5 days = 30 spots

additional_members = additional_spots_per_week / avg_perform_consumption
                   = 30 / 0.4 = 75 members

sale_add_change = sale_without_change + additional_members
                = 8 + 75 = 83 members
```

**Where the numbers come from:**
- `max_capacity = 10` (maximum spots possible for BLIGH PERFORM)
- `current_capacity = 4` (current configured spots)
- `operational_days = 5` (Mon-Fri)
- `avg_perform_consumption = 0.4` (from system_config)

**Result:** 
- **8 members** can be added at current capacity (4 spots/day)
- **75 additional members** if we expand to max capacity (10 spots/day)
- **Total potential: 83 members**

---

## Quick Reference

### What Each Number Tells You

| Metric | What It Means | When to Use It |
|--------|---------------|----------------|
| **sale_without_change** | Members you can add right now | "Can we sell more without hiring more coaches?" |
| **sale_add_change** | Total members if you expand | "What's the full potential if we max out capacity?" |

### The Math Behind It

**Why divide by 0.4?**
- Members attend an average of 2.5 sessions per week
- 1 / 2.5 = 0.4 consumption rate
- So: 10 spots ÷ 0.4 = 25 members (each member uses ~0.4 spots per week)

**Why multiply by operational_days?**
- Capacity is per day, but we calculate weekly
- 2 spots/day × 5 days = 10 spots/week

---

## Configuration Settings

These values are stored in the `system_config` table:

- **lc_buffer:** 0.9 (target 90% utilization)
- **avg_perform_consumption:** 0.4 (average sessions per member per week)
- **Max capacities for PERFORM classes:**
  - BLIGH: 10 spots/day
  - BRIDGE: 8 spots/day
  - COLLIN: 6 spots/day

---

## Summary

**The formulas answer:**
- ✅ How many members can we add **without changing anything?**
- ✅ How many members can we add **if we expand capacity?**

**The key insight:**
- `sale_without_change` = use existing space
- `sale_add_change` = existing space + expansion potential

**Remember:**
- Current capacity = what you have now
- Max capacity = what you could have
- The difference = expansion opportunity
