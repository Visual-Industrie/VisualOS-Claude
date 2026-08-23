# Timesheet Views - Admin and Staff

## 1. Admin Timesheet Page

### Overview
A dedicated admin page showing all time entries across all staff, filterable by date range, staff member, and job.

### Location
New page in the admin section: /admin/timesheets

### Display
Show a table of time entries with the following columns:
- Staff member name
- Job name and number
- Task name
- Start time
- End time
- Duration (calculated)
- Entry type (timer-based or manual)

### Filters
- Date range picker (default to current week)
- Staff member dropdown
- Job search/dropdown

### Totals
Show total hours per staff member for the selected date range at the bottom or in a summary row.

### Acceptance Criteria
- Admin can view all time entries across all staff.
- Admin can filter by date range, staff member, and job.
- Duration is calculated and displayed for each entry.
- Manual entries are visually distinguished from timer-based entries.
- Total hours per staff member are shown for the filtered period.

---

## 2. Staff Timesheet View

### Overview
A personal timesheet view where staff can see, edit, add, and delete their own time entries.

### Location
Accessible from the tablet view and/or main nav: /timesheet or /my-timesheet

### Display
Show a list of the current staff member's time entries grouped by day, with:
- Job name and number
- Task name
- Start time
- End time
- Duration (calculated)
- Entry type (timer-based or manual)

### Editing
Staff can edit any of their own time entries:
- Adjust start time
- Adjust end time
- Reassign to a different task
- Delete an entry

### Adding Missing Entries
Staff can manually add a time entry by selecting:
- Job
- Task
- Start time
- End time

### Acceptance Criteria
- Staff can view their own time entries grouped by day.
- Staff can edit start time, end time, and task on any of their entries.
- Staff can manually add a missing entry by selecting job, task, start and end time.
- Staff can delete an entry.
- Duration is recalculated automatically when start or end time is changed.
- Manual entries are visually distinguished from timer-based entries.
- Changes take effect immediately with no approval step required.
