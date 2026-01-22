# Retool Queries for Attendance Views

## Weekly Aggregate View (Primary Table) - view_attendance_weekly_26_mvp_csu

This is the **primary table** for your Retool dashboard. It shows weekly aggregated data with average spots calculation.

### Basic Weekly Query - All Data (Primary Table)
```sql
SELECT 
    week_start,
    session_type,
    session_start,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity,
    total_capacity_for_week,
    average_spots,
    operational_days,
    total_attendance_for_week,
    avg_attendance_per_session,
    utilization_rate,
    sale_without_change,
    sale_add_change
FROM view_attendance_weekly_26_mvp_csu
ORDER BY week_start DESC, session_start;
```

### Filter by Week Range
```sql
SELECT 
    week_start,
    session_type,
    session_start,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity,
    total_capacity_for_week,
    average_spots,
    operational_days,
    total_attendance_for_week,
    avg_attendance_per_session,
    utilization_rate,
    sale_without_change,
    sale_add_change
FROM view_attendance_weekly_26_mvp_csu
WHERE week_start >= {{ dateRangePicker1.start }}
  AND week_start <= {{ dateRangePicker1.end }}
ORDER BY week_start DESC, session_start;
```

### Filter by Specific Session Type
```sql
SELECT 
    week_start,
    session_type,
    session_start,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity,
    total_capacity_for_week,
    average_spots,
    operational_days,
    total_attendance_for_week,
    avg_attendance_per_session,
    utilization_rate,
    sale_without_change,
    sale_add_change
FROM view_attendance_weekly_26_mvp_csu
WHERE session_type = {{ dropdown1.value }}
ORDER BY week_start DESC, session_start;
```

### Recent 8 Weeks
```sql
SELECT 
    week_start,
    session_type,
    session_start,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity,
    total_capacity_for_week,
    average_spots,
    operational_days,
    total_attendance_for_week,
    avg_attendance_per_session,
    utilization_rate,
    sale_without_change,
    sale_add_change
FROM view_attendance_weekly_26_mvp_csu
WHERE week_start >= DATE_TRUNC('week', CURRENT_DATE) - INTERVAL '8 weeks'
ORDER BY week_start DESC, session_start;
```

### Filter by Utilization Rate
```sql
SELECT 
    week_start,
    session_type,
    session_start,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity,
    total_capacity_for_week,
    average_spots,
    operational_days,
    total_attendance_for_week,
    avg_attendance_per_session,
    utilization_rate,
    sale_without_change,
    sale_add_change
FROM view_attendance_weekly_26_mvp_csu
WHERE week_start >= {{ dateRangePicker1.start }}
  AND week_start <= {{ dateRangePicker1.end }}
  AND utilization_rate >= {{ numberInput1.value }}
ORDER BY week_start DESC, session_start;
```

### Drill-Down Query (On Cell Click)
When a user clicks a cell in the weekly table, use this to show daily details:
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    gym,
    class_type,
    coaches,
    members_attended,
    actual_attendance,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE DATE_TRUNC('week', session_date) = {{ table1.selectedRow.data.week_start }}
  AND session_type = {{ table1.selectedRow.data.session_type }}
  AND session_start = {{ table1.selectedRow.data.session_start }}
ORDER BY session_date, session_start;
```

### Drill-Down Query (All Sessions for a Week)
Show all daily sessions for a selected week:
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    gym,
    class_type,
    coaches,
    members_attended,
    actual_attendance,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE DATE_TRUNC('week', session_date) = {{ table1.selectedRow.data.week_start }}
ORDER BY session_date, session_start, session_type;
```

---

## Daily Detail View - view_data_attendance_26_mvp_csu

This is the **detail view** used for drill-down from the weekly aggregate table.

#### Basic Query - All Data
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    gym,
    class_type,
    coaches,
    members_attended,
    actual_attendance,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
ORDER BY session_date DESC, session_start, session_type;
```

### Filter by Date Range
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= {{ dateRangePicker1.start }}
  AND session_date <= {{ dateRangePicker1.end }}
ORDER BY session_date DESC, session_start, session_type;
```

### Filter by Specific Session Type
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_type = {{ dropdown1.value }}
ORDER BY session_date DESC, session_start;
```

### Filter by Date and Session Type
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= {{ dateRangePicker1.start }}
  AND session_date <= {{ dateRangePicker1.end }}
  AND session_type = {{ dropdown1.value }}
ORDER BY session_date DESC, session_start;
```

### Recent 30 Days Only
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY session_date DESC, session_start, session_type;
```

### Today's Sessions Only
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date = CURRENT_DATE
ORDER BY session_start, session_type;
```

### Grouped by Session Type and Start Time (Summary View)
```sql
SELECT 
    session_start,
    session_type,
    STRING_AGG(DISTINCT coaches, '; ') as all_coaches,
    COUNT(DISTINCT session_date) as days_occurred,
    SUM(actual_attendance) as total_attendance,
    AVG(actual_attendance) as avg_attendance
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= {{ dateRangePicker1.start }}
  AND session_date <= {{ dateRangePicker1.end }}
GROUP BY session_start, session_type
ORDER BY session_start, session_type;
```

### Search by Coach Name
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE coaches ILIKE '%{{ textInput1.value }}%'
ORDER BY session_date DESC, session_start;
```

### Search by Member Name
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE members_attended ILIKE '%{{ textInput1.value }}%'
ORDER BY session_date DESC, session_start;
```

### Sessions with High Attendance (Above Threshold)
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= {{ dateRangePicker1.start }}
  AND session_date <= {{ dateRangePicker1.end }}
  AND actual_attendance >= {{ numberInput1.value }}
ORDER BY actual_attendance DESC, session_date DESC;
```

### Sessions with Low Attendance (Below Threshold)
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= {{ dateRangePicker1.start }}
  AND session_date <= {{ dateRangePicker1.end }}
  AND actual_attendance <= {{ numberInput1.value }}
ORDER BY actual_attendance ASC, session_date DESC;
```

### Filter by Specific Time Range
```sql
SELECT 
    session_date,
    session_start,
    session_type,
    coaches,
    members_attended,
    actual_attendance,
    gym,
    class_type,
    class_type_capacity,
    class_instance_capacity,
    max_class_type_capacity
FROM view_data_attendance_26_mvp_csu
WHERE session_date >= {{ dateRangePicker1.start }}
  AND session_date <= {{ dateRangePicker1.end }}
  AND session_start >= {{ timePicker1.value }}::time
  AND session_start <= {{ timePicker2.value }}::time
ORDER BY session_date DESC, session_start;
```

## Retool Setup Instructions:

1. **Create a new Resource in Retool:**
   - Go to Resources → Create new → Database → PostgreSQL (or your Supabase connection)
   - Enter your Supabase connection details

2. **Create a Query:**
   - In your app, add a new Query
   - Select your database resource
   - Copy one of the queries above
   - Adjust the component references (like `{{ dateRangePicker1.start }}`) to match your UI components

3. **Component References:**
   - `{{ dateRangePicker1.start }}` - Reference to a date picker component
   - `{{ dropdown1.value }}` - Reference to a dropdown component
   - `{{ textInput1.value }}` - Reference to a text input component

4. **To Refresh the Materialized Views:**
   If you need to refresh the views from Retool, create separate queries:
   
   **Refresh Weekly Aggregate View:**
   ```sql
   SELECT refresh_view_attendance_weekly_26_mvp_csu();
   ```
   
   **Refresh Daily Detail View:**
   ```sql
   SELECT refresh_view_data_attendance_26_mvp_csu();
   ```
   
   **Refresh Both Views:**
   ```sql
   SELECT refresh_view_attendance_weekly_26_mvp_csu();
   SELECT refresh_view_data_attendance_26_mvp_csu();
   ```
   
   Note: These are admin queries and should be run with appropriate permissions.

## Recommended Retool Setup

### Primary Table Setup
1. Create a Table component in Retool
2. Connect it to the weekly aggregate query (view_attendance_weekly_26_mvp_csu)
3. This will be your main dashboard view showing weekly metrics

### Drill-Down Setup
1. Add an event handler to the Table component (on row click)
2. Create a Modal or Detail View component
3. Connect it to the drill-down query using the selected row's data
4. This will show daily details when a user clicks on a weekly row

### Refresh Schedule
- Set up a scheduled query to refresh both views daily (recommended: 1 AM)
- Or add a manual refresh button that calls both refresh functions
