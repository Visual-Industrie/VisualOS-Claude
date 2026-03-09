# VisualOS Backlog

This document tracks all planned features and enhancements for the VisualOS signage management system.

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
- Project phases: Design → Quote → With Customer → Production → Install → Invoice (New removed from Kanban)
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
- New Project button in sidebar, Home page, and contact detail page (contact pre-fills form)
- Calendar & Schedule system: per-project Schedule tab (React Big Calendar), master Calendar page, GCal events overlay (public holidays, personal events), Google Calendar CRUD sync, deduplication of VisualOS-created events
- Site survey: camera interface for photo capture (`getUserMedia`, rear-facing preferred) + file upload; same camera UI on Completion Photos tab
- Home page: search not persisted across sessions; X clear button + ESC key to clear search field

---

## Backlog Items

### 1. Contacts System
**Status:** ✅ Complete  
**Priority:** High  
**Labels:** `feature`, `xero-api`, `contacts`, `webhook`

**Completed:**
- [x] Prisma schema: `Contact`, `ContactAddress`, `ContactPhone`, `ContactPerson` models
- [x] `xeroClient.ts` shared helper — single source of truth for Xero token handling and refresh
- [x] `POST /api/contacts/sync` — bulk import all Xero contacts with pagination
- [x] `POST /api/webhooks/xero` — HMAC-verified webhook receiver, handles contact CREATE/UPDATE events and invoice stub
- [x] `GET /api/contacts` — paginated, searchable contact list
- [x] `GET /api/contacts/:id` — full contact detail with all relations
- [x] `POST /api/contacts` — create contact in Xero first, then store locally
- [x] Project model: `contactId` FK to local `Contact`, `xeroContactName` retained during migration period
- [x] One-time migration script to link existing projects to Contact records (run as temp API endpoint)
- [x] Frontend: `Contacts.page.tsx` — searchable list with Sync from Xero button and New Contact modal
- [x] Frontend: `ContactDetail.page.tsx` — read-only detail page with Overview, People, Addresses, Projects tabs
- [x] Frontend: "Edit in Xero" deep-link button (`https://go.xero.com/Contacts/View/{xeroContactId}`)
- [x] Frontend: `Shell.page.tsx` updated — Contacts nav link + `/contacts` and `/contacts/:id` routes
- [x] Frontend: `Dashboard.page.tsx` updated — contact name resolved from `project.contact?.name`
- [x] Webhook raw body fix — using `express.json({ verify })` pattern to capture raw body before parsing
- [x] Deployed and verified in production — 572 contacts synced, webhook confirmed working

**Notes:**
- Xero OAuth scopes updated to `accounting.contacts` (write superset, replaces `accounting.contacts.read`)
- Single tenant — `XERO_TENANT_ID` stored in env
- Webhook registered at `https://vis.vil.nz/api/webhooks/xero`
- `xeroContactName` column still present in Project — remove in a future cleanup migration once all references confirmed gone

---

### 2. Docker Dev/Prod Compose Split + GitHub Actions CI/CD
**Status:** ✅ Complete
**Priority:** Low
**Labels:** `enhancement`, `infrastructure`, `nice-to-have`

**Completed:**
- [x] `docker-compose.yml` is now production base (no volume mount, runs `npm start`, migrations on startup)
- [x] `docker-compose.dev.yml` dev overrides: mounts `.:/app`, runs `npm run dev` with hot reload
- [x] Dev command: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up`
- [x] Production command: `sudo docker compose up --build -d`
- [x] Backend GitHub Actions deploy workflow: push to `main` → SSH → git pull → docker rebuild
- [x] Frontend GitHub Actions deploy workflow: push to `master` → Vite build in CI → scp dist/ to NAS
- [x] CLAUDE.md updated with new commands and deployment docs

---

### 3. Calendar Integration with Google Calendar
**Status:** ✅ Complete (core) — GCal overlay + deduplication shipped March 2026
**Priority:** High
**Labels:** `feature`, `calendar`, `google-api`

**Description:**  
Build a comprehensive calendar system integrated with our shared Google Calendar (`studio@vil.nz`) to manage project tasks, schedules, and staff assignments.

#### Staff & Calendars
- Dynamically fetch sub-calendars from `studio@vil.nz` "My Calendars" via API
- Allow selection from available staff calendars when creating events
- Support for `bren@vil.nz`, `bev@vil.nz`, and production staff calendars

#### Event/Task Types
**Events:**
- Design (1-4 hours suggested)
- Production - Print (30min - 2 hours suggested)
- Production - Laminate (30min - 1 hour suggested)
- Production - Cut & Apply (1-3 hours suggested)
- Install (1-8 hours suggested, varies)
- Client Meeting (30min - 1 hour suggested)

**Tasks:** (non-calendar items)
- Quote
- Invoice

#### Database Schema

```prisma
model Task {
  id                Int      @id @default(autoincrement())
  projectId         Int
  project           Project  @relation(fields: [projectId], references: [id])
  type              String   // design/print/laminate/cut-apply/install/meeting/quote/invoice
  assignedCalendarId String  // Google Calendar ID
  assignedTo        String   // Staff name/email
  startTime         DateTime?
  endTime           DateTime?
  duration          Int?     // minutes
  googleEventId     String?  // For sync with Google Calendar
  notes             String?
  status            String   @default("pending") // pending/in-progress/complete
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```

#### UI Components

1. **Schedule Tab (per-project):**
   - Custom calendar view using **React Big Calendar**
   - Styled with Mantine theme
   - Filter to project-specific events
   - Quick-create buttons for different event types
   - Show only relevant staff calendars
   - Add/edit events inline

2. **Master Calendar View:**
   - Embedded Google Calendar iframe (`studio@vil.nz`)
   - Full overview of all projects and staff

3. **Kanban Board:**
   - Filter view by ALL tasks or by specific staff member

#### Technical Implementation
- Use `studio@vil.nz` OAuth tokens (same pattern as Gmail integration)
- React Big Calendar for custom calendar views
- Bi-directional sync between VisualOS database and Google Calendar
- Google Calendar API for CRUD operations

**Completed:**
- [x] Backend: `calendarRoutes.ts` — `/api/calendar/calendars` (list studio@vil.nz calendars), `/api/calendar/events` (GCal overlay by date range), per-calendar failure logging
- [x] Backend: `/api/projects/:id/schedule` — CRUD for schedule events (Tasks with `eventType`); creates/updates/deletes corresponding Google Calendar events
- [x] Backend: `/api/schedule` — all events across all projects (used by master Calendar page)
- [x] Backend: Create/update/delete events in Google Calendar (via `studio@vil.nz` refresh token)
- [x] Database: `Task` extended with `eventType`, `eventCalendarId`, `eventAssignee`, `startTime`, `endTime`, `duration`, `googleEventId`
- [x] Frontend: `ScheduleTab.tsx` — per-project Schedule tab with React Big Calendar; event creation/edit modal; suggested durations per type; calendar colour key; upcoming + unscheduled event lists
- [x] Frontend: `CalendarPage.tsx` — master calendar view with GCal events overlay (public holidays, personal events); calendar filter dropdown; colour-coded by calendar; deduplication (VisualOS-created events not shown twice); all-day events parsed as local time (dayjs); clicking VisualOS event shows detail modal with "Open project schedule" button; clicking GCal event opens in Google Calendar
- [x] Events include VisualOS link in Google Calendar description
- [x] GCal overlay error handling: per-calendar failures logged; frontend errors surfaced via console.warn

**Deferred / not built:**
- [ ] Kanban board filtered by staff/all tasks
- [ ] Bi-directional sync (changes made directly in Google Calendar reflected back in VisualOS)

---

### 4. Google Drive Design Approval System - Phase 2 (Customer-Facing)
**Status:** ✅ Complete
**Priority:** High
**Labels:** `feature`, `google-api`, `customer-facing`, `security`

**Completed:**
- [x] `ProjectContact` model — per-project contacts (name, email, isPrimary); CRUD via `/api/projects/:id/contacts`
- [x] `DesignApprovalToken` model — 96-char hex token, 14-day expiry, one per contact per file send
- [x] `DesignApproval` model — MFA-verified approval record; stores 6-digit code + expiry, final status
- [x] `PortalAuditLog` model — structured event log (token_accessed, mfa_sent, mfa_verified, approval_submitted, feedback_submitted, token_reissued) with IP + user agent
- [x] `Task` extended: `source`, `sourceReferenceId`, `portalTokenId`; feedback tasks created as `draft`, promoted to `pending` on submission; all task queries exclude drafts
- [x] `DesignFile` extended: `approvalSentAt`, `sentByUserId`; `approvalStatus` removed (redundant — use `status` + `approvalSentAt`)
- [x] `SystemSettings` extended: `termsUrl` (seeded with Notion T&C URL); admin can update via Settings page
- [x] `utils/sendEmail.ts` — Gmail API via `studio@vil.nz`; RFC 2047 subject encoding; base64 MIME; auto-labels with `VisualOS/JOB-{id}` (MFA emails excluded); 4 templates
- [x] `utils/portalAudit.ts` — non-fatal structured event logging
- [x] `POST /api/projects/:id/design-files/:fileId/send-approval` — generates tokens, sends emails, falls back to Xero contact if no project contacts
- [x] `GET /api/projects/:id/design-files/:fileId/approvals` — returns tokens with nested approvals
- [x] `GET /portal/:token` — validates token, returns portal data including `termsUrl` and saved feedback items
- [x] `GET /portal/:token/pdf` — streams Drive file using sentBy user's refresh token (no public Drive access)
- [x] `POST /portal/:token/request-mfa` — generates 6-digit code, sends MFA email (no job label)
- [x] `POST /portal/:token/verify-mfa` — validates code + expiry
- [x] `POST /portal/:token/approve` — requires MFA; updates `DesignFile.status`; promotes draft feedback tasks to pending; sends team notification email with link to project design tab
- [x] `POST /portal/:token/feedback` + `PATCH` + `DELETE` — feedback items saved as draft tasks; editable until approval submitted
- [x] `POST /portal/:token/reissue` — issues new token for expired links
- [x] Frontend: `Portal.page.tsx` — three-mode state machine (idle → changes → complete); restores state from server on load
- [x] Frontend: `ApprovalActions` — Approve/Request Changes buttons; T&Cs checkbox (disabled Approve until ticked)
- [x] Frontend: `MFAConfirmationModal` — 6-digit PinInput, verify then approve
- [x] Frontend: `FeedbackEntryList` — inline edit/delete; `readOnly` prop for post-submission display
- [x] Frontend: `FeedbackEntryForm`, `DesignViewer`, `PortalLayout`, `TokenExpiredScreen`
- [x] Frontend: `ProjectContactsPanel` in project Details tab
- [x] Frontend: `DesignTab` updated — Send for Approval button, per-contact approval status badges
- [x] Frontend: portal route outside AppShell in `App.tsx`
- [x] Frontend: admin Settings page — editable T&Cs URL field

---

### 5. Site Survey Tab
**Status:** ✅ Complete (Phase 1)
**Priority:** Medium
**Labels:** `feature`, `survey`, `maps`

**Completed:**
- [x] `SiteSurvey` and `SurveyPhoto` Prisma models + migration
- [x] Backend CRUD routes (`GET`, `PUT` per project, `POST` photo upload, `DELETE` photo)
- [x] Photo upload via multer (20MB limit), stored in `uploads/survey-photos/`, served as static files
- [x] Frontend: Survey tab added to project detail page (`/projects/:id/survey`)
- [x] Address text field + notes textarea, persisted via upsert
- [x] Photo upload modal with width/height (mm) and notes per photo
- [x] Thumbnail grid with lightbox on click, delete button per photo
- [x] TypeScript types: `ISiteSurvey`, `ISurveyPhoto`

**Deferred to Phase 2 (see item 15):**
- Google Maps address autocomplete + "Open in Google Maps" link
- Calendar integration (schedule survey event)
- Mobile optimisation pass

---

### 6. Materials Catalog & AI Recommendation System
**Status:** 🚫 PARKED - On hold until higher priority features complete  
**Priority:** Low  
**Labels:** `feature`, `ai`, `parked`, `future`

**Description:**  
Create a comprehensive materials management system that uses Claude AI to recommend appropriate substrates, vinyls, and laminates based on project requirements. Includes inventory tracking and cutting layout optimization.

#### Business Value
- Speed up material selection for production staff
- Reduce material waste through optimized cutting layouts
- Maintain accurate inventory levels
- Provide cost estimates based on material usage

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

### 7. Kanban Drag & Drop for Project Status
**Status:** ✅ Complete  
**Priority:** Medium  
**Labels:** `enhancement`, `ux`, `kanban`

**Completed:**
- [x] `@hello-pangea/dnd` configured with DragDropContext, Droppable, Draggable
- [x] Optimistic UI updates
- [x] Backend PATCH sync on drop
- [x] Error handling with revert
- [x] Drop zone highlight (`isDraggingOver` background)
- [x] Toast notification on status change

---

### 8. Project Tasks Tab
**Status:** ✅ Complete  
**Priority:** High  
**Labels:** `feature`, `tasks`, `ux`

**Completed:**
- [x] Prisma migration: `dueDate` and `priority` on `Task`
- [x] Backend: `GET /api/users` endpoint
- [x] Backend: task CRUD with new fields
- [x] Frontend: Tasks tab on project detail page
- [x] Frontend: Task list grouped by status
- [x] Frontend: Add/edit task modal (reusable `TaskModal` component)
- [x] Frontend: Assignee dropdown, due date picker, priority badge
- [x] Frontend: Inline status toggle
- [x] Design tab retains lightweight per-version task lists

---

### 9. My Tasks Page
**Status:** ✅ Complete  
**Priority:** Medium  
**Labels:** `feature`, `tasks`, `dashboard`

**Completed:**
- [x] Backend: `GET /api/tasks/mine` returning user's tasks with project info
- [x] Backend: `POST /api/tasks` for standalone tasks (no project required)
- [x] Frontend: `/my-tasks` route and page component
- [x] Frontend: Tasks grouped by due date / priority / project / none
- [x] Frontend: Sort by due date, priority, project
- [x] Frontend: Filter by priority and project
- [x] Frontend: Show/hide completed tasks toggle
- [x] Frontend: Inline complete toggle with optimistic update
- [x] Frontend: Project + customer name link (→ details or design tab)
- [x] Frontend: Standalone "personal" tasks with no project
- [x] Frontend: New task via reusable `TaskModal`
- [x] Sidebar nav "My Tasks" link with incomplete task count badge

---

### 10. Standardise API Routes to RESTful Plural Nouns
**Status:** ✅ Complete
**Priority:** Low
**Labels:** `enhancement`, `api`, `refactoring`

**Completed:**
- [x] `/api/contacts` added as plural from the start
- [x] `/api/projects` and `/api/notes` plural mounts added to `index.ts`
- [x] Frontend updated: `/api/project` → `/api/projects` and `/api/note` → `/api/notes` (`Project.tsx`)
- [x] Singular alias routes (`/api/project`, `/api/note`) removed from `index.ts`

---

### 11. Global App State with React Context
**Status:** 📋 Planned  
**Priority:** Medium  
**Labels:** `enhancement`, `architecture`, `ux`

**Description:**  
Currently some state (e.g. task count badge in sidebar) is fetched independently per component with polling. A shared React context would allow components to reactively share state without redundant API calls or polling.

**Initial candidates for global state:**
- Incomplete task count (so ticking complete in MyTasks instantly updates sidebar badge)
- Logged-in user object (currently re-fetched in multiple places)
- Potentially: active project, notifications queue

**Proposed approach:**
- Create `AppContext` (or separate contexts per domain e.g. `TaskContext`, `UserContext`)
- Wrap app in provider in `App.tsx`
- Expose update functions so child components can push state changes up

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

**Description:**  
Unit test infrastructure for the backend is not yet configured. `contactRoutes.test.ts` has been written but can't run until vitest is installed and configured.

**Steps:**
- `npm install --save-dev vitest`
- Add `"test": "vitest"` to `package.json` scripts
- Run existing `contactRoutes.test.ts` and fix any issues

---

### 13. Fix Vitest ERR_REQUIRE_ESM in Frontend
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `testing`, `infrastructure`, `frontend`

**Description:**
The frontend `npm run test` suite fails at the vitest step with an unhandled `ERR_REQUIRE_ESM` error. `html-encoding-sniffer` (a jsdom dependency) is trying to CJS-`require()` `@exodus/bytes/encoding-lite.js`, which is now ESM-only. This causes vitest to abort before running any tests.

**Error:**
```
Error: require() of ES Module node_modules/@exodus/bytes/encoding-lite.js
from node_modules/html-encoding-sniffer/lib/html-encoding-sniffer.js not supported.
```

**Steps to investigate:**
- Check if pinning `@exodus/bytes` to an older CJS-compatible version resolves it
- Or upgrade `jsdom` / `vitest` to a version where the dependency chain is compatible
- Run `npm run vitest` after fix to confirm tests execute cleanly

---

### 14. Remove `xeroContactName` from Project
**Status:** 📋 Planned — Cleanup  
**Priority:** Low  
**Labels:** `refactoring`, `database`, `cleanup`

**Description:**  
During the contacts migration, `xeroContactName` and the old `xeroContactId` were retained on the `Project` model for safety. Now that contacts are fully linked via `contactId` FK, these columns can be dropped.

**Steps:**
1. Confirm all projects have `contactId` populated (check for nulls)
2. Search codebase for any remaining references to `xeroContactName` or `project.xeroContactId`
3. Run `npx prisma migrate dev --name remove_xero_contact_fields_from_project`
4. Deploy

---

---

### 17. Phone Message Feature
**Status:** ✅ Complete — PRs open for review
**Priority:** Medium
**Labels:** `feature`, `tasks`, `ux`

**Description:**
Quick "Take a Message" button in the sidebar lets any team member capture a phone message from any page without losing their place. The message becomes a Task assigned to the relevant person, optionally linked to a project.

**Completed:**
- [x] Prisma: 5 new fields on `Task` — `isPhoneMessage`, `callerName`, `callerPhone`, `callerEmail`, `takenAt`
- [x] Migration: `add_phone_message_to_task` applied
- [x] Backend: both `POST /tasks` and `POST /projects/:projectId/tasks` accept + persist new fields
- [x] Backend: `GET /xero/projects` Contact include extended with `emailAddress` + `phones` for auto-fill
- [x] Frontend: `PhoneMessageModal` — fetches users + all projects on open, captures `takenAt` at button-press time, auto-fills phone (MOBILE preferred, then DEFAULT) + email from project contact, submits to project or standalone task endpoint
- [x] Frontend: "Take a Message" button in sidebar (between New Project and Sync Projects)
- [x] Frontend: My Tasks shows 📞 icon, caller phone/email below subtitle, `takenAt` timestamp in place of due date
- [x] Frontend: `ITask` + `IProject` types updated

**Deferred to backlog:**
- Email notification to assignee when a phone message is recorded

---

### 18. Persistent Sessions (PostgreSQL Session Store)
**Status:** ✅ Complete
**Priority:** High
**Labels:** `fix`, `infrastructure`, `auth`

**Completed:**
- [x] Replaced in-memory `MemoryStore` with `connect-pg-simple` PostgreSQL session store
- [x] Sessions now survive backend restarts and container rebuilds — users stay logged in
- [x] `session` table created via `createTableIfMissing: true` (Prisma drift resolved with manual migration)
- [x] `pg.Pool` using `DATABASE_URL` wired into session store in `index.ts`

---

### 19. Silent Drive Token Refresh
**Status:** ✅ Complete
**Priority:** High
**Labels:** `feature`, `google-api`, `drive`, `auth`

**Description:**
Previously the Drive folder picker required users to re-authorise via a Google popup on every session. Now a backend endpoint exchanges the stored refresh token for a short-lived access token silently.

**Completed:**
- [x] Backend: `GET /auth/drive-token` — uses `google_refresh_token` from session user, exchanges via googleapis `OAuth2Client`, returns `access_token`
- [x] Frontend: `src/utils/driveToken.ts` — `getDriveAccessToken()` calls the endpoint and returns the token
- [x] All Drive picker components (`DriveFolderPicker`, `DrivePickerButton`) use `getDriveAccessToken()` instead of GIS popup
- [x] Token errors shown as toast notification

---

### 20. Bidirectional Project → Contact Navigation
**Status:** ✅ Complete
**Priority:** Medium
**Labels:** `fix`, `ux`, `contacts`

**Description:**
Project detail page now links back to the associated contact using `project.contactId` and `project.xeroContactName` — the same contact that appears in the contact's Projects tab.

**Completed:**
- [x] `Project.tsx`: contact name displayed as `<Link to="/contacts/:contactId">` using `xeroContactName` as label
- [x] Contact email shown alongside link when available via `project.contact.emailAddress`
- [x] `GET /api/xero/projects/:id` updated to include `Contact` relation and remap it to lowercase `contact` in response

---

### 21. Drive Folder Picker Smart Defaults + Admin Settings
**Status:** ✅ Complete
**Priority:** Medium
**Labels:** `feature`, `google-api`, `drive`, `admin`

**Description:**
The Drive folder picker now opens in a contextually relevant default folder rather than the Drive root. Admins set a system-wide base job folder; contacts have their own folder set via Brand Assets tab; projects default to the client's folder.

**Completed:**
- [x] Prisma: `isAdmin Boolean @default(false)` added to `User`; `bren@vil.nz` + `bev@vil.nz` seeded as admins in migration
- [x] Prisma: `SystemSettings` model added (singleton id=1); `baseDriveFolderId` + `baseDriveFolderName` fields
- [x] Backend: `settingsRoutes.ts` — `GET /api/settings` (all authenticated), `PATCH /api/settings/base-drive-folder` (admin only)
- [x] Backend: `ensureAdmin` middleware added (`utils/ensureAdmin.ts`)
- [x] Frontend: `Settings.tsx` — admin-only section with `DrivePickerButton` to set base job folder; fetches current setting on load
- [x] Frontend: `BrandAssetsTab.tsx` — fetches system settings in parallel, passes `baseDriveFolderId` as `defaultParentFolderId` to the folder picker
- [x] Frontend: `Project.tsx` (`DriveFolderPicker`) — passes `contact.driveFolderId` as `defaultParentFolderId`
- [x] Frontend: `DrivePickerButton` — generic reusable component (extracted from `DriveFolderPicker`); accepts `linkedFolderId`, `linkedFolderName`, `defaultParentFolderId`, `isSaving`, `onFolderSelected`
- [x] `types/user.ts`: `isAdmin` added to `User` interface; `SystemSettings` interface added

**Drive folder hierarchy:**
```
Base Job Folder (set by admin in Settings)
  └── Client Folder (set per-contact in Brand Assets tab)
       └── Project Folder (set per-project in Details tab)
```

---

## Priority Summary

### ✅ Recently Completed

- **GCal Events Overlay — fixes & deduplication** — Backend now logs per-calendar failures (403s etc.) instead of silently dropping them. Frontend: empty `catch {}` replaced with `console.warn`; VisualOS-created events deduplicated (filtered by `googleEventId` so they don't appear twice); all-day events (public holidays) parsed with dayjs as local time instead of UTC midnight.
- **Home page search UX** — Search string no longer persisted in localStorage (select filters still are). Added X clear button in the TextInput right section; ESC key clears the field when focused.
- **Completion Photos — camera interface** — Same 3-phase modal as the Survey tab: source picker (Take photo / Upload file) → live camera view (rear-facing preferred) → preview + notes before uploading.
- **Calendar page — event detail modal** — Clicking a VisualOS event on the master Calendar page shows a detail modal (title, type badge, project, client, time, notes, assignee) with an "Open project schedule" button. GCal-only events still open in Google Calendar.
- **Scheduled Design Approval Emails** — Clock icon on design file cards opens a DateTimePicker to queue the approval email for a future time. `DesignFile.scheduledSendAt` field; server-side 60s poll in `approvalScheduler.ts` fires emails and sets `approvalSentAt`. Cancel X revokes unused tokens. Timezone-aware (NZ local → UTC ISO before sending to backend).
- **Client Portal — Design Approval (Phase 2)** — Token-gated portal for client design review and approval. Per-project contacts, 96-char hex tokens (14-day expiry), PDF proxy via Drive, MFA 6-digit code, feedback items as draft tasks, audit log, Gmail auto-labelling, T&Cs checkbox, team notification email with direct link to project design tab
- **Admin Role + System Settings** — `isAdmin` on User (bren + bev seeded); `SystemSettings` singleton model; admin-only Settings page section with Drive folder picker to set base job folder; `ensureAdmin` middleware
- **Drive Folder Picker Smart Defaults** — project picker opens in client's Drive folder; Brand Assets picker opens in system base folder; `DrivePickerButton` extracted as generic reusable component
- **Silent Drive Token Refresh** — `/auth/drive-token` backend endpoint exchanges stored refresh token; `getDriveAccessToken()` frontend utility used by all picker components; no Google popup required
- **Persistent Sessions** — `connect-pg-simple` PostgreSQL session store replaces MemoryStore; sessions survive backend rebuilds
- **Bidirectional Project → Contact Navigation** — project detail page links to associated contact via `contactId` + `xeroContactName`
- **Phone Message** — Take a Message button in sidebar; modal captures caller details, auto-fills from project contact, records time at button-press; My Tasks shows phone icon + caller details + taken-at time
- **QoL nav + project creation** — removed "New Project" nav link, added outline button in sidebar action stack; removed "New" column from Kanban; added "New Project" button to Home page; "New Project" button on contact detail page pre-selects the contact in the create form
- **Fix: CLOSED/DRAFT project viewing** — `GET /api/xero/projects/:id` was hardcoded to `xeroStatus: 'INPROGRESS'`, blocking CLOSED projects; fixed to `findUnique` with no status filter
- **Fix: Xero sync 500** — Xero Projects API rejects `DRAFT` as a states filter; removed DRAFT fetch from sync, removed Draft filter/badges from UI
- **Drive Folder Picker (initial)** — link a Google Drive folder to any project; Google Picker API, folder name shown as clickable link in Details tab
- **GitHub Actions CI/CD** — push to `main`/`master` auto-deploys to Synology NAS; docker-compose split into prod base + dev override
- **Site Survey Tab Phase 2** — Google Maps Places autocomplete, lat/lng capture, "Open in Google Maps" link
- **Contacts System** — full Xero sync, webhook (HMAC verified, production confirmed), local DB, contacts list + detail pages, 572 contacts synced
- Kanban drag & drop with drop zone highlighting and toast notifications
- Reusable `notify` utility (`@mantine/notifications`)
- My Tasks page (grouping, sorting, filtering, standalone tasks, sidebar badge)
- Email dark mode fix (light mode island for HTML email bodies)
- Design Approval System Phase 1 (internal workflow)
- Project Tasks Tab (TaskList + TaskModal, full CRUD, assignee/due date/priority)
- Standardise API Routes — frontend migrated to plural nouns, singular aliases removed, note creation userId bug fixed
- Site Survey Tab Phase 1 — address, notes, photo upload with dimensions, lightbox

### High Priority (Ready to Build)

1. Calendar Integration
2. Shop Floor Workflow Viewer *(depends on Calendar Integration)*

### Medium Priority

1. Global App State with React Context
2. Field View — Install & Survey Map
3. Delivery & Install Docket — Mobile Field View
4. Client Portal *(depends on Design Approval Phase 2)*
5. Project Templates *(depends on Calendar Integration)*

### Low Priority / Cleanup

1. Business Intelligence Dashboard
2. Vitest Setup (Backend)
3. Fix Vitest ERR_REQUIRE_ESM (Frontend) — jsdom/ESM dependency incompatibility
4. Remove `xeroContactName` from Project (cleanup migration)
5. Materials Catalog (PARKED)

---

### 16. Drive Folder Picker for Projects
**Status:** ✅ Complete
**Priority:** Medium
**Labels:** `feature`, `google-api`, `drive`

**Description:**
Each project can now be linked to a Google Drive folder. The folder ID is stored on the project (shared across all team members). Uses Google Identity Services (GIS) on the frontend to request a short-lived Drive token client-side — avoids touching backend OAuth scopes.

**Completed:**
- [x] Prisma: `driveFolderId` + `driveFolderName` added to `Project` model, migration applied
- [x] Backend: `PATCH /projects/:id` extended to accept `{ driveFolderId, driveFolderName }` independently of status
- [x] Frontend: `DriveFolderPicker` component — loads GIS + gapi scripts, initialises token client lazily on click, opens folder-only Google Picker
- [x] Frontend: `IProject` type updated with optional `driveFolderId` / `driveFolderName`
- [x] Frontend: `VITE_GOOGLE_CLIENT_ID` + `VITE_GOOGLE_API_KEY` added to env files; values in `.env.development.local` and GH Actions secrets
- [x] Details tab: shows "Link Drive Folder" button when unlinked; folder name as clickable Drive link + "Change" button when linked
- [x] GCP: Google Picker API enabled; `http://localhost:5173` + `https://visualos.vil.nz` added to Authorized JavaScript origins

---

### 15. Site Survey Tab — Phase 2 (Google Maps Integration)
**Status:** ✅ Complete — pushed to master
**Priority:** Medium
**Labels:** `feature`, `survey`, `google-api`, `maps`

**Description:**
Enhance the Survey tab address field with Google Maps integration. No in-app map embed — keep it lightweight. Just autocomplete to resolve a clean address + coordinates, and a link to open the location in Google Maps.

**Completed:**
- [x] Google Places Autocomplete on the address field (`@googlemaps/js-api-loader` v2, `setOptions` + `importLibrary` functional API)
- [x] Selecting an autocomplete result populates `address`, `lat`, `lng` — manual typing clears lat/lng
- [x] `lat`/`lng` included in `saveSurvey` PUT payload (backend already supported these fields)
- [x] "Open in Google Maps" link button next to Save (opens `https://maps.google.com/?q=lat,lng` in new tab)
- [x] Falls back gracefully — link hidden when no lat/lng stored
- [x] `VITE_GOOGLE_MAPS_API_KEY` added to `.env.development` (requires Maps JavaScript API + Places API enabled in Google Cloud)
- [x] `@types/google.maps` added as dev dep; `tsconfig.json` updated with `"google.maps"` in types array

---

## Notes

- All Google API integrations use `studio@vil.nz` shared account OAuth tokens
- Database changes require Prisma migrations — use `npx prisma migrate deploy` on production (not `migrate dev`)
- Customer-facing features require public routes with token-based authentication
- All features should maintain mobile responsiveness
- Email notifications go through Gmail API via `studio@vil.nz`
- Production deploy is automated via GitHub Actions — push to `main`/`master` triggers deploy. No manual steps needed.
- Xero webhook key and tenant ID stored in `.env` as `XERO_WEBHOOK_KEY` and `XERO_TENANT_ID`

---

---

### 22. Shop Floor Workflow Viewer
**Status:** 📋 Planned  
**Priority:** High  
**Labels:** `feature`, `shop-floor`, `tablet`, `ux`

**Description:**  
A dedicated tablet-optimised view permanently displayed in the production area. No sidebar, no nav — just what floor staff need. Shows today's production jobs pulled from calendar events, with big tappable status toggles (Pending → In Progress → Done) suited to gloved hands. Auto-refreshes every 60 seconds. Pinned to a shop floor user session (no repeated logins). Landscape-optimised layout with a clear "What's Next" queue — overdue jobs highlighted in red, today's amber, upcoming green. Optionally shows a thumbnail of the linked Drive design file so staff can see exactly what they're producing.

**Notes:**
- Depends on Calendar Integration (#3) being complete first
- Route: `/shopfloor` — fully separate layout, no AppShell
- Session pinned to a dedicated shop floor user account
- Consider large touch targets throughout — this is a gloved-hand environment

---

### 23. Business Intelligence Dashboard
**Status:** 📋 Planned  
**Priority:** Low  
**Labels:** `feature`, `analytics`, `admin`, `reporting`

**Description:**  
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

**Description:**  
A map-based field operations view at `/field` (or `/map`) that plots both active installs and pending surveys on a Google Map, colour-coded by type and status. The key insight: being able to visually identify clusters of jobs in the same area and batch them — "we've got 3 surveys and 2 installs all within 2km of each other this week, let's do them in one run."

Surveys should also appear as calendar events (schedule a survey from the Survey tab, same as scheduling an install). The field view brings both together in one geographic picture.

**Features:**
- Google Map with pins for all projects that have survey lat/lng coordinates
- Separate pin styles for surveys (pending/scheduled) vs installs (scheduled/in-progress)
- Filter by date range, status, and assigned staff member
- Click a pin → project summary card (client name, job type, status, assigned staff) with link to open the full project
- Optionally show drive time / distance between selected pins for route planning

**Notes:**
- Lat/lng already captured in Survey tab — existing data can populate the map immediately
- Depends on Calendar Integration (#3) for survey scheduling
- Mobile-optimised — this is a field tool, used from a phone or tablet in the van
- Uses Google Maps JS API (already enabled in GCP)

---

### 25. Delivery & Install Docket — Mobile Field View
**Status:** 📋 Planned  
**Priority:** Medium  
**Labels:** `feature`, `install`, `mobile`, `field`

**Description:**  
When a job hits Install status, a mobile-optimised "docket" view is available for the installer — not a printed PDF, but a clean phone/tablet screen they reference on-site. Pulls together everything the installer needs in one place without navigating through tabs.

**Contents:**
- Client name, site address with "Open in Maps" link (uses Survey lat/lng)
- Design thumbnail from the linked Drive folder
- Install task checklist (tick off items on-site)
- Notes from the Survey tab (site-specific details the installer needs to know)
- A photo upload button to capture completion photos directly from the camera, saved to the project's Drive folder in an `Install Photos` subfolder
- Timestamped and GPS-tagged completion record

**Notes:**
- Accessible from the project detail page (Install tab or a dedicated button when status = Install)
- No new auth required — uses existing session
- Completion photos saved to Drive extend the existing photo pattern from the Survey tab
- Could eventually trigger a client completion email with photos attached (via Gmail API)

---

### 26. Client Portal
**Status:** 📋 Planned  
**Priority:** Medium  
**Labels:** `feature`, `customer-facing`, `magic-link`, `security`

**Description:**  
A persistent (token-gated, no password) client-facing mini-portal extending the Design Approval Phase 2 magic-link pattern. Clients get a single link tied to their contact record that shows all their active projects in a clean, branded view — no Kanban complexity, just the information relevant to them.

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
- Public routes with token-based middleware (same pattern as Design Approval Phase 2)

**Notes:**
- Design Approval Phase 2 (#4) should be completed first — portal will absorb that flow
- Branding via Brand Assets tab (colours, logo) — existing infrastructure
- Consider a "share portal link" button on the Contact detail page

---

### 27. Project Templates
**Status:** 📋 Planned  
**Priority:** Medium  
**Labels:** `feature`, `templates`, `ux`, `workflow`

**Description:**  
For common recurring job types (vehicle wraps, shop signage, event banners, etc.), save a reusable template that scaffolds out a new project automatically. Huge time saver for bread-and-butter work where the task list, assignees, and calendar event types are predictable.

**Template contents:**
- Default set of tasks with suggested assignees and priorities
- Calendar event types with suggested durations (leverages Calendar Integration types)
- Notes boilerplate (e.g. standard checklist for a full vehicle wrap)
- Default project status on creation

**UI:**
- Templates managed in Settings (admin) or a dedicated `/templates` page
- On "New Project" — optional "Start from template" picker before the form
- Template selection pre-populates tasks, notes, and calendar scaffolding; user can edit before saving

**Notes:**
- Depends on Calendar Integration (#3) for the scheduling scaffolding
- Templates stored as a new `ProjectTemplate` model with child `TemplateTask` records
- No external integrations required — purely internal data

---

**Last Updated:** March 10, 2026 (Completed: Calendar Integration core — Schedule tab, master Calendar page, GCal overlay with deduplication + all-day date fix; camera interface on Completion Photos tab; Home page search UX improvements)