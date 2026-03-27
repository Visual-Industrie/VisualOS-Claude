# VisualOS Backlog

This document tracks planned and in-progress features. Completed items live in `complete.md`.

---

## Project Overview

**VisualOS** is a signage project management tool that integrates with:
- Xero (project and customer data)
- Google Workspace (Gmail, Calendar, Drive)
- Synology NAS (file storage)

**Tech Stack:**
- Backend: Express + Postgres + Prisma (TypeScript)
- Frontend: React + Mantine UI (Vite)
- Deployment: Docker on Synology NAS (BE: `vis.vil.nz`, FE: `visualos.vil.nz`)
- APIs: Xero, Google (Gmail, Calendar, Drive)

**Current Features:**
- Xero project import + sync (INPROGRESS + CLOSED only — Xero API does not support DRAFT filter)
- Project phases: Design → Quote → With Customer → Production → Install → Invoice (Kanban excludes New, Completed, Abandoned)
- Notes system (rich text editor)
- Email integration (Gmail threads via `studio@vil.nz`)
- Kanban dashboard view with drag & drop (status updates via PATCH, optimistic UI, drop zone highlighting)
- Toast notification system (`@mantine/notifications` + reusable `notify` utility)
- Docker hot-reload for development
- Design approval system (internal): Drive file picker, version history, approve/request changes, per-version and project-level tasks, scheduled approval emails (server-side 60s poll, NZDT/NZST-aware)
- Client portal — design approval (customer-facing): token-gated portal, PDF proxy, MFA-verified approval, change requests with feedback items, per-project contacts, Gmail auto-labelling, T&Cs checkbox, portal audit log
- Task management: reusable TaskList + TaskModal components with assignee, due date, priority, status
- URL-synced project tabs (React Router)
- My Tasks page: grouped/sorted/filtered task view, standalone tasks, sidebar badge with incomplete count
- Contacts system: full Xero sync, webhook, local DB, contacts list + detail pages
- Drive folder picker: link a Google Drive folder to any project (backend drive-token endpoint, stored per-project)
- Drive folder picker smart defaults: project picker opens in client's Drive folder; Brand Assets picker opens in system base folder
- Admin role: `isAdmin` flag on User; bren + bev seeded as admins
- System Settings: single-row `SystemSettings` model stores base Drive folder (admin-only Settings page)
- Persistent sessions: `connect-pg-simple` PostgreSQL session store — logins survive backend restarts/rebuilds
- Silent Drive token refresh: `getDriveAccessToken()` utility + `/auth/drive-token` backend endpoint
- Calendar & Schedule system: per-project Schedule tab (React Big Calendar), master Calendar page, GCal events overlay (public holidays, personal events), Google Calendar CRUD sync, deduplication of VisualOS-created events
- Site survey: camera interface for photo capture (`getUserMedia`, rear-facing preferred) + file upload; same camera UI on Completion Photos tab
- Home page: search not persisted; X clear button + ESC key to clear search field; filters persisted in localStorage
- Frontend unit tests: 101 tests across 13 files (`yarn vitest` / `yarn vitest --reporter=verbose`); `happy-dom` environment
- Staff management: admin-only Staff tab in Settings; StaffMember model with name, email, display colour, Xero user mapping, Google Calendar mapping, active/inactive toggle
- Taxonomy editor: admin Lists tab in Settings; TaxonomyItem model drives project stages, material categories, and task types; cascade rename, archive/restore, Mantine colour picker; Kanban columns, status dropdowns, badge colours, and material category dropdowns all live from taxonomy
- Shop Floor tablet view (`/shopfloor`): staff picker (persisted), today's task cards with start/stop/complete/undo, optimistic updates, time-tracking progress bar, design preview overlay, task type + project filters, auto-refresh every 60s; "Switch user" button in header
- Timesheets: `TimeEntry` model tracks start/stop per task; My Timesheet page shows staff's own time; Admin Timesheets page shows all staff with filters and totals; project Timesheets tab shows all entries per job; manual entry add/edit/delete; notes field on entries; Job column shows customer + project name
- Survey & Completion Photos: multi-file upload supported — thumbnail grid + sequential upload with progress; single-file retains notes/dimensions flow
- Tasks: task type badge shown in all task list views (My Tasks, project lists, Shop Floor); project picker shown when editing tasks on My Tasks page
- Vinyl Calculator: per-project tab for laying out vinyl pieces on a roll; skyline packing with auto-rotation; live layout preview, efficiency %, print sizes with bleed; calculations saved per project
- EFTPOS: admin Settings tab shows Verifone transaction history by date range; manual sync button

---

## Backlog Items

### 11. Global App State with React Context
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `enhancement`, `architecture`, `ux`

Currently some state (e.g. task count badge in sidebar) is fetched independently per component with polling. A shared React context would allow components to reactively share state without redundant API calls or polling.

**Initial candidates for global state:**
- Incomplete task count (so ticking complete in MyTasks instantly updates sidebar badge)
- Logged-in user object (currently re-fetched in multiple places)

**Acceptance Criteria:**
- [ ] `UserContext` providing logged-in user across app
- [ ] `TaskContext` providing incomplete task count, updated optimistically on status change
- [ ] Sidebar badge updates instantly when task is completed in MyTasks page
- [ ] No redundant API calls for shared data

---

### 12. Vitest Setup (Backend)
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `testing`, `infrastructure`

Unit test infrastructure for the backend is not yet configured. `contactRoutes.test.ts` has been written but can't run until vitest is installed and configured.

**Steps:**
- `npm install --save-dev vitest`
- Add `"test": "vitest"` to `package.json` scripts
- Run existing `contactRoutes.test.ts` and fix any issues

---

### 14. Remove `xeroContactName` from Project
**Status:** 📋 Planned — Cleanup
**Priority:** Low
**Labels:** `refactoring`, `database`, `cleanup`

During the contacts migration, `xeroContactName` and the old `xeroContactId` were retained on the `Project` model for safety. Now that contacts are fully linked via `contactId` FK, these columns can be dropped.

**Steps:**
1. Confirm all projects have `contactId` populated (check for nulls)
2. Search codebase for any remaining references to `xeroContactName` or `project.xeroContactId`
3. Run `npx prisma migrate dev --name remove_xero_contact_fields_from_project`
4. Deploy

---

### 22. Shop Floor Workflow Viewer
**Status:** ✅ Shipped — March 2026
**Priority:** High
**Labels:** `feature`, `shop-floor`, `tablet`, `ux`

Tablet-optimised `/shopfloor` view. Staff picker persisted to localStorage. Shows active, upcoming, and completed tasks for the selected staff member. Start/stop/complete/undo with optimistic updates. Time-tracking progress bar. Design preview modal. Auto-refresh every 60s.

---

### 23. Business Intelligence Dashboard
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `feature`, `analytics`, `admin`, `reporting`

Admin-only `/analytics` page surfacing business insights from existing data. Potential metrics: revenue pipeline by project stage (from Xero quote values), staff utilisation from calendar hours, average job cycle time per phase (to spot bottlenecks), top clients by project volume, and overdue installs (jobs in Install with no completed install task). Built with Recharts (already in stack).

**Notes:**
- Admin-only route, guarded by `ensureAdmin` middleware
- All data already exists in VisualOS — no new integrations required
- Low priority until core workflow features are complete

---

### 24. Field View — Install & Survey Map
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `maps`, `install`, `survey`, `calendar`, `mobile`

A map-based field operations view at `/field` that plots both active installs and pending surveys on a Google Map, colour-coded by type and status. Click a pin → project summary card with link to the full project. Batch nearby jobs in one run.

**Features:**
- Google Map with pins for all projects that have survey lat/lng coordinates
- Separate pin styles for surveys (pending/scheduled) vs installs (scheduled/in-progress)
- Filter by date range, status, and assigned staff member
- Click a pin → project summary card (client name, job type, status, assigned staff) with link to open the full project
- Optionally show drive time / distance between selected pins for route planning

**Notes:**
- Lat/lng already captured in Survey tab — existing data can populate the map immediately
- Mobile-optimised — used from a phone or tablet in the van
- Uses Google Maps JS API (already enabled in GCP)

---

### 25. Delivery & Install Docket — Mobile Field View
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `install`, `mobile`, `field`

When a job hits Install status, a mobile-optimised "docket" view is available for the installer. Pulls together everything needed in one place without navigating through tabs.

**Contents:**
- Client name, site address with "Open in Maps" link (uses Survey lat/lng)
- Design thumbnail from the linked Drive folder
- Install task checklist (tick off items on-site)
- Notes from the Survey tab
- Photo upload button to capture completion photos directly from the camera, saved to Drive in an `Install Photos` subfolder

**Notes:**
- Accessible from the project detail page when status = Install
- Could eventually trigger a client completion email with photos attached (via Gmail API)

---

### 26. Client Portal (Full)
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `customer-facing`, `magic-link`, `security`

A persistent (token-gated, no password) client-facing mini-portal extending the Design Approval Phase 2 magic-link pattern. Clients get a single link tied to their contact record that shows all their active projects.

**Portal features:**
- All active projects for the client with current status
- Design file review and approval (Phase 2 already covers this — portal integrates it)
- Scheduled install dates (from calendar)
- Completion photos after install
- Ability to raise a "change request" that creates a Task in VisualOS
- Branded using the client's brand colours from the Brand Assets tab

**Auth model:**
- Magic link tied to `Contact.xeroContactId` — no login, no password
- Token stored on Contact model, with optional expiry or rolling refresh

**Notes:**
- Design Approval Phase 2 is already complete — portal absorbs that flow
- Branding via Brand Assets tab (colours, logo) — existing infrastructure

---

### 27. Project Templates
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `templates`, `ux`, `workflow`

For common recurring job types (vehicle wraps, shop signage, event banners, etc.), save a reusable template that scaffolds out a new project automatically.

**Template contents:**
- Default set of tasks with suggested assignees and priorities
- Calendar event types with suggested durations
- Notes boilerplate (e.g. standard checklist for a full vehicle wrap)
- Default project status on creation

**UI:**
- Templates managed in Settings (admin) or a dedicated `/templates` page
- On "New Project" — optional "Start from template" picker before the form

**Notes:**
- Templates stored as a new `ProjectTemplate` model with child `TemplateTask` records
- No external integrations required — purely internal data

---

### 28. Pickup / Job Complete Notification Emails
**Status:** ✅ Shipped — March 10, 2026
**Priority:** High
**Labels:** `feature`, `email`, `notifications`

Status change triggers an email to the primary project contact via the existing `sendEmail.ts`. Template editable in Settings → Templates. Notification enabled per stage via "Send notification email" toggle in Settings → Lists.

---

### 29. Client Communication Preferences
**Status:** 📋 Planned
**Priority:** High
**Labels:** `feature`, `contacts`, `notifications`

`communicationPreference` enum (`email`/`sms`/`phone`) on `ProjectContact`. Phone preference = staff reminder task. SMS via Twilio deferred.

---

### 30. WordPress Quote Request Form → VisualOS API
**Status:** ✅ Shipped — March 2026
**Priority:** High
**Labels:** `feature`, `api`, `intake`, `contacts`

Public `POST /api/quote-requests`, origin-locked to `visualindustrie.co.nz`. Fuzzy contact match on business name + email. Captures contact person name separately. Creates lead in New status.

---

### 31. Project Scaffolding / Decision Tree Wizard
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `ux`, `workflow`

JSON-driven decision tree per org, wizard UI, template editor, preset industry templates. Spec before build.

---

### 32. Multi-Tenancy / Organisation Layer
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `architecture`, `infrastructure`

`Organisation` model + `organisationId` FK across all tables. Start with single org for Visual Industrie. Design all new features with this in mind.

---

### 33. Admin Integration Settings Page
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `admin`, `settings`

UI for org admins to manage API credentials (Xero, Google, etc.). Credentials encrypted at rest in DB.

---

### 6. Materials Catalog & AI Recommendation System
**Status:** 🚫 PARKED
**Priority:** Low
**Labels:** `feature`, `ai`, `parked`, `future`

Create a comprehensive materials management system that uses Claude AI to recommend appropriate substrates, vinyls, and laminates based on project requirements. Includes inventory tracking and cutting layout optimization.

**Acceptance Criteria:**
- [ ] Database schema implemented
- [ ] Material catalog CRUD interface
- [ ] Claude API integration for recommendations
- [ ] Structured JSON output from Claude
- [ ] Cutting layout algorithm
- [ ] Canvas visualization of cutting layouts
- [ ] Recommendation approval workflow
- [ ] Inventory tracking
- [ ] Cost estimation

---

### 34. Transactional Email Provider
**Status:** 📋 Planned — Future
**Priority:** Low
**Labels:** `infrastructure`, `future`

SendGrid/Mailgun when external customers onboard. Not needed while single-tenant on Gmail API.

---

### 35. Email / Calendar Provider Abstraction
**Status:** 📋 Planned — Future
**Priority:** Low
**Labels:** `architecture`, `future`

Provider-agnostic interface for Gmail/Outlook/iCloud. Required before multi-tenancy goes live.

---

### 36. File Storage Provider Abstraction
**Status:** 📋 Planned — Future
**Priority:** Low
**Labels:** `architecture`, `future`

Provider-agnostic storage layer beyond Google Drive. Required before multi-tenancy goes live.

---

### 37. Move Notes onto Project Details Tab
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `ui`, `cleanup`

The Notes tab has been hidden. Notes should be surfaced inline on the Details tab — likely as a simple list below the description with an inline "Add note" input, so staff don't have to switch tabs to jot something down.

---

## Priority Summary

### High Priority (Ready to Build)
1. **Client Communication Preferences** (#29)

### Medium Priority
1. Global App State with React Context (#11)
2. Field View — Install & Survey Map (#24)
3. Delivery & Install Docket — Mobile Field View (#25)
4. Client Portal — Full (#26)
5. Project Templates (#27)
6. Project Scaffolding / Decision Tree Wizard (#31)
7. Multi-Tenancy / Organisation Layer (#32)
8. Admin Integration Settings Page (#33)

### Low Priority / Cleanup
1. Move Notes onto Project Details Tab (#37)
2. Vitest Setup (Backend) (#12)
3. Remove `xeroContactName` from Project (#14) — cleanup migration
4. Business Intelligence Dashboard (#23)
5. Materials Catalog (#6) — PARKED
6. Transactional Email Provider (#34) — future
7. Email / Calendar Provider Abstraction (#35) — future
8. File Storage Provider Abstraction (#36) — future

---

## Notes

- All Google API integrations use `studio@vil.nz` shared account OAuth tokens
- Database changes require Prisma migrations — use `npx prisma migrate deploy` on production (not `migrate dev`)
- Customer-facing features require public routes with token-based authentication
- All features should maintain mobile responsiveness
- Email notifications go through Gmail API via `studio@vil.nz`
- Production deploy is automated via GitHub Actions — push to `main`/`master` triggers deploy
- Xero webhook key and tenant ID stored in `.env` as `XERO_WEBHOOK_KEY` and `XERO_TENANT_ID`
- VisualOS is being designed with multi-tenancy in mind — new models should be designed to accept an `organisationId` FK when #32 is implemented

---

**Last Updated:** March 27, 2026 — Shipped project Timesheets tab, multi-photo upload, task type badges, design approval confirmation token flow, task edit project picker
