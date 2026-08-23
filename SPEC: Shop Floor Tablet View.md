SPEC: Shop Floor Tablet View
Feature: Shop Floor Workflow & Timekeeping Interface
Priority: High — depends on Staff Management above
Status: Ready to build
Overview
A touch-optimised tablet interface for shop floor staff to view their day's tasks, track time against each one, and mark work done. Time entries batch-sync to Xero Projects nightly at 10 PM. Uses Mantine components throughout for heavy lifting.
UI Flow
On load: Fetch all tasks where due_date = today AND assigned_to = current staff member. Also include tasks with due_date = null assigned to that staff member (standing tasks with no due date). Sort High → Medium → Low priority.
Filter bar at top: Filter by task type (Laminate, Apply & Trim, Weeding, Print, etc.) to support batching similar work. Sort toggle available — default is priority order, manually reorderable.
Task cards — each card shows:

Customer name
Project name (nullable — omit if standalone task)
Priority badge (High / Medium / Low)
Task type label
Estimated vs actual time bar chart (Mantine — if time_tracking_enabled = true)
Start / Stop / Done buttons (if time_tracking_enabled = true)
Done checkbox only (if time_tracking_enabled = false)
Link to open full project (if project-linked)

Time bar (Mantine Progress):

Gray segment = estimated time
Black segment = actual accumulated time
Red segment = overflow if actual exceeds estimated
Numeric display of accumulated time alongside bar

Actions:

Start — logs started_at timestamp
Stop — logs stopped_at, pauses accumulation
Done — marks complete, card animates to completed section at bottom
Undo — restores completed card to active stack (fat-finger protection)

Completed section at bottom of screen — stays visible for the day, retained in VisualOS indefinitely for historical reporting.
Data Model
Tasks table additions:
FieldTypeNotesproject_idUUID (nullable)Null for standalone taskscustomer_namestringDenormalised for displayproject_namestring (nullable)Denormalised for displayassigned_toUUIDFK → staff_membersdue_datedate (nullable)Null = show every daypriorityenumhigh / medium / lowtask_typestringLaminate, Apply & Trim, Weeding, etc.estimated_minutesinteger (nullable)Manual now; template-driven latertime_tracking_enabledbooleanToggles timer UIstatusenumpending / in_progress / donexero_syncedbooleanDefault falsexero_sync_attimestamp (nullable)When sync ranxero_time_entry_idstring (nullable)Xero Projects time entry ID — deduplication key
Time entry log table (new):
FieldTypeNotesidUUIDtask_idUUIDFK → tasksstarted_attimestampstopped_attimestamp (nullable)Null if still runningmanually_adjusted_minutesinteger (nullable)Override total if staff corrects
Actual time = sum of all (stopped_at - started_at) pairs. If manually_adjusted_minutes is set, use that instead.
End-of-Day Xero Sync
Trigger: Automated cron at 10:00 PM nightly + manual sync button on interface.
Logic:

Fetch all tasks where xero_synced = false AND time_tracking_enabled = true AND due_date = today
Calculate total minutes from time entry log (or manual override)
Round up to nearest 15 minutes
POST time entry to Xero Projects API against linked project
Store returned Xero time entry ID → xero_time_entry_id
Set xero_synced = true, xero_sync_at = now()

Deduplication: xero_synced flag prevents double-syncing. xero_time_entry_id provides audit link back to Xero. If a re-sync is ever needed, use the stored ID to PATCH rather than POST.
Future

Task templates auto-scaffold estimated times by job type (e.g. ACM sign → Laminate 30 min, Apply & Trim 30 min)
Supervisor live view — see all staff timers running across the floor in real time