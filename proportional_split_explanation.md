# Proportional Split Calculation - Simple Explanation

## How It Works

The proportional split divides the maximum capacity (10 spots for BLIGH) between staggered classes based on how many coaches each class has. For example, if a 6:10am class has 2 coaches (4 spots capacity) and a 6:30am class has 1 coach (2 spots capacity), they share the 10-spot maximum proportionally: the 6:10am class gets 6.67 spots (66.7% because it has 2/3 of the coaches) and the 6:30am class gets 3.33 spots (33.3% because it has 1/3 of the coaches). The formula is: `(this_class_capacity / total_block_capacity) × max_capacity`, where `this_class_capacity = base_capacity × number_of_coaches`.
