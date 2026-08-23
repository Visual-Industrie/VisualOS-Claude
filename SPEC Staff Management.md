SPEC: Staff Management
Feature: Staff Profiles & System Linking
Priority: High — foundational for Shop Floor View and Scheduling
Status: Ready to build
Overview
A staff management section in VisualOS that allows you to create and manage staff member profiles, linking each person to their Xero identity and their vil.nz Google Calendar. This is foundational infrastructure required by the Shop Floor View, scheduling, and future reporting features.
UI
A staff list page under Settings (or People). Each staff member has a profile with:

Name — display name used throughout VisualOS
Email — their vil.nz address
Display colour — colour picker, used in calendar and scheduling views
Xero User ID — linked via dropdown populated from Xero API (list of Xero organisation users)
Google Calendar — dropdown populated from the vil.nz Google Workspace calendars list (one calendar per staff member under the shared account)
Active/Inactive toggle — soft disable without deleting

Data Model
staff_members
- id (UUID)
- name (string)
- email (string)
- display_colour (string — hex)
- xero_user_id (string, nullable)
- google_calendar_id (string, nullable) — e.g. firstname@vil.nz
- is_active (boolean, default true)
- created_at (timestamp)
Notes

Calendar integration is already live — just need the dropdown to list available calendars from the vil.nz workspace account
Xero user list available via Xero API (GET /Users)
No auth changes needed — this is admin-only config