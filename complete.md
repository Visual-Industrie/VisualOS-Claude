# VisualOS — Completed Features

A record of everything shipped. Items are ordered roughly by completion date (most recent first).

---

### Bug Fixes — Mantine v8 API Changes (March 2026)
**Labels:** `fix`, `frontend`

- `DateTimePicker` onChange in Mantine v8 returns a formatted string (`'YYYY-MM-DD HH:mm:ss'`) not a `Date` object. Fixed in `MyTimesheet.page.tsx`, `AdminTimesheets.page.tsx`, and `ScheduleTab.tsx` by parsing with `new Date(String(val).replace(' ', 'T'))`.
- `DatePickerInput` onChange also returns a string (`'YYYY-MM-DD'`). Fixed in both timesheet pages by parsing with `new Date(String(val))`.
- Both bugs caused silent `TypeError: n.toISOString is not a function` when submitting forms or changing date range filters.
- Tiptap duplicate extension: `@tiptap/starter-kit` now bundles `Link` internally — removed explicit `Link` import from `ProjectDescriptionEditor.tsx` and `Project.tsx`, configured via `StarterKit.configure({ link: {...} })`.

---

### Status Change Spinner + Contextual Toast (March 2026)
**Labels:** `ux`, `frontend`

- Status combobox on the project Details tab shows a loading spinner (disabled) while the PATCH is in-flight.
- On success, a toast announces the new status. If the stage has `closesXero: true`, the toast says "project closed in Xero". If `sendNotification: true`, it says "customer notified". Both flags can appear together.
- Same contextual toast logic applied to Kanban drag-and-drop on the Dashboard page.

---

### Timesheets (March 2026)
**Labels:** `feature`, `backend`, `frontend`

Time tracking for shop floor staff, with admin oversight.

**Backend:**
- `TimeEntry` Prisma model: `taskId` FK, `startedAt`, `stoppedAt` (nullable), `manuallyAdjustedMinutes`, `isManual`
- `GET /api/timesheets` (admin) — all entries filterable by dateFrom/dateTo/staffMemberId/projectId
- `GET /api/timesheets/mine` — current user's entries resolved via email → StaffMember lookup
- `POST /api/timesheets` — create manual entry (taskId, startedAt, stoppedAt)
- `PATCH /api/timesheets/:id` — edit startedAt/stoppedAt/taskId
- `DELETE /api/timesheets/:id`
- Shop floor `start`/`stop` actions create/close `TimeEntry` records automatically

**Frontend:**
- `MyTimesheet.page.tsx` — entries grouped by day; date range filter; manual vs timer badge; add/edit/delete entries; duration preview in edit form
- `AdminTimesheets.page.tsx` — table of all staff entries; filters for date range + staff member; totals per staff member
- `ITimesheetEntry` type + `entryDurationMinutes()` / `formatDuration()` helpers in `types/timesheet.ts`

---

### Shop Floor Workflow Viewer (March 2026)
**Labels:** `feature`, `shop-floor`, `tablet`, `backend`, `frontend`

Tablet-optimised `/shopfloor` view for production floor staff.

**Backend:**
- `GET /api/shopfloor/tasks` — returns active, future, and completed tasks for a staff member on a given local date (params: `staffMemberId`, `localDate`, `utcOffsetMinutes`)
- `POST /api/shopfloor/tasks/:id/start` — creates a `TimeEntry` with `startedAt = now()`
- `POST /api/shopfloor/tasks/:id/stop` — closes the open `TimeEntry` with `stoppedAt = now()`
- `POST /api/shopfloor/tasks/:id/complete` — marks task complete and stops any running timer
- `POST /api/shopfloor/tasks/:id/undo` — reverts task to pending
- Task `estimatedMinutes` field drives progress bar; `timeTrackingEnabled` flag

**Frontend:**
- `ShopFloor.page.tsx` — no AppShell/nav; `StaffPickerOverlay` on first visit (selection persisted to localStorage); active, upcoming, and completed task sections; task type + project filter chips; auto-refresh every 60s; `UnavailableScreen` if backend unreachable on first load
- `ShopFloorTaskCard` — start/stop/complete/undo buttons; time-tracking progress bar vs estimated minutes; staff member badge; design preview button (Google Drive iframe, full-screen modal)
- `ShopFloorHeader`, `StaffPickerOverlay`, `TaskTypeFilterBar`, `CompletedTaskSection`
- Task create/edit form (`TaskModal`) gained staff member field and estimated time field
- Task list and card views show assigned staff member name + colour dot
- Calendar events auto-assign the linked staff member's Google Calendar → tasks appear on their shop floor view
- `types/shopfloor.ts`: `IShopFloorTask`, `IShopFloorResponse`, `IShopFloorStaff`, `ITimeEntry`

---

### Vinyl Calculator (March 2026)
**Labels:** `feature`, `backend`, `frontend`

Per-project vinyl layout calculator with best-fit skyline packing.

**Backend:**
- `GET`/`POST`/`PATCH`/`DELETE` `/api/projects/:projectId/vinyl-calculations`

**Frontend:**
- `VinylCalculatorTab.tsx` — set roll width, bleed, and gap; add pieces (name, W×H, qty, colour); live SVG layout preview; efficiency %; print sizes with bleed; placement coordinate table
- Calculations are named, saved to the project, and support full CRUD
- Packing algorithm auto-rotates pieces for best fit; fixed portrait/landscape orientation bug

---

### Projects List Improvements (March 2026)
**Labels:** `ux`, `frontend`

- Added "Date Added" and "Due Date" columns
- Column settings button — choose which columns to show; preferences saved in localStorage
- Sort direction toggle; Due Date added as sort option
- Stage filter is a compact badge with chip picker; click a stage badge on a project row to change stage inline
- Edit modal: update project name, customer, description, and due date without leaving the list
- Due date shown in project header when set
- Status combobox auto-saves on change — no separate Save button needed

---

### Taxonomy Editor — Editable Project Stages & Material Categories (March 2026)
**Labels:** `feature`, `settings`, `admin`, `backend`, `frontend`

Admin-managed lists for project stages and material categories, replacing all hardcoded values across the app.

**Backend:**
- `TaxonomyItem` Prisma model: `type` discriminator, `name` (stored DB key), `label` (display override), `colour` (Mantine colour name), `isArchived`, `sortOrder`, `meta` JSON (stage-specific flags: `showInKanban`, `closesXero`, `sendNotification`, `canNotifyCustomer`); valid types: `project_stage`, `material_category`, `task_type`
- Migration seeded with all 11 project stages and 10 material categories
- `GET /api/taxonomy/:type` — non-admin users see active items only
- `POST /api/taxonomy/:type` — create (admin only)
- `PATCH /api/taxonomy/:type/:id` — update with cascade rename in a DB transaction: project_stage → `UPDATE Project SET status = newName WHERE status = oldName`; material_category → `UPDATE Material SET category = newName WHERE category = oldName`
- `VALID_STATUSES` and `XERO_CLOSE_STATUSES` in projectRoutes now query taxonomy (no longer hardcoded)

**Frontend:**
- `useTaxonomy(type)` hook with 5-minute module-level cache; `invalidateTaxonomyCache()` called after writes
- New **Lists** admin-only tab in Settings with `TaxonomyPanel` containing reusable `TaxonomySection` components — adding a new taxonomy type requires one extra `<TaxonomySection>` call
- Colour picker shows all 14 Mantine colour names with live badge previews
- Project stages have "Show in Kanban" and "Closes Xero project" toggles
- **Project detail** status dropdown, **Home page** badge colours, **Kanban board** columns, **Materials page**, **MaterialPickerModal**, and **DeliverablesTab** all derive from taxonomy — no deploys needed for label/colour/stage changes

---

### Staff Management (March 2026)
**Labels:** `feature`, `settings`, `admin`, `backend`, `frontend`

Admin-only staff roster. Staff members have a name, email, display colour, optional Xero user mapping, optional Google Calendar mapping, and an active/inactive flag.

**Backend:**
- `StaffMember` Prisma model with UUID id, unique email, and all above fields
- Migration `20260310120000_add_staff_members` (applied via psql; session table drift workaround)
- `GET /api/staff` — list all, active-first then alphabetical (any authed user)
- `GET /api/staff/xero-users` — proxy Xero org users for dropdown (admin only)
- `POST /api/staff` — create (admin only); 409 on duplicate email
- `PATCH /api/staff/:id` — partial update (admin only); 404 on not-found
- `DELETE /api/staff/:id` — delete (admin only); 404 on not-found

**Frontend:**
- `StaffPanel` component: list table with colour swatch, name, email, active/inactive badge, edit and delete action icons
- `StaffModal`: name, email, colour picker, Google Calendar dropdown (fetched from `/api/calendar/calendars`), Xero user dropdown (fetched from `/api/staff/xero-users`), active toggle (edit mode only)
- Delete confirmation modal
- Admin-only "Staff" tab added to the Settings page
- `IStaffMember` and `IXeroUser` types added to `user.ts`
- 16 Vitest tests (total suite: 101 tests across 13 files)

---

### Frontend Unit Tests (March 2026)
**Labels:** `testing`, `infrastructure`, `frontend`

Fixed the vitest ERR_REQUIRE_ESM error (jsdom → html-encoding-sniffer → @exodus/bytes ESM incompatibility) by switching to `happy-dom`. Added 85 unit tests across 11 new test files.

- `vite.config.mjs`: `environment: 'jsdom'` → `environment: 'happy-dom'`; `jsdom` removed, `happy-dom` added
- Tests covering: `notify` utility, `getDriveAccessToken`, `useAuthStatus` hook, `LoginButton`, `LogoutButton`, `TaskItem`, `TaskModal`, `DeleteProjectModal`, `FeatureRequestModal`, `TokenExpiredScreen`, `ReleasesPanel`
- `yarn vitest` / `yarn vitest --reporter=verbose` — runs cleanly, 85 passing

---

### GCal Events Overlay — Fixes & Deduplication (March 2026)
**Labels:** `fix`, `calendar`, `google-api`

- Backend now logs per-calendar failures (403s etc.) instead of silently dropping them
- Frontend: empty `catch {}` replaced with `console.warn`
- VisualOS-created events deduplicated (filtered by `googleEventId` so they don't appear twice on the master calendar)
- All-day events (public holidays) parsed with dayjs as local time instead of UTC midnight

---

### Home Page Search UX (March 2026)
**Labels:** `fix`, `ux`

- Search string no longer persisted in localStorage (select filters still are)
- Added X clear button in the TextInput right section
- ESC key clears the field when focused

---

### Completion Photos — Camera Interface (March 2026)
**Labels:** `feature`, `ux`, `camera`

Same 3-phase modal as the Survey tab: source picker (Take photo / Upload file) → live camera view (rear-facing preferred) → preview + notes before uploading.

---

### Calendar Page — Event Detail Modal (March 2026)
**Labels:** `feature`, `calendar`, `ux`

Clicking a VisualOS event on the master Calendar page shows a detail modal (title, type badge, project, client, time, notes, assignee) with an "Open project schedule" button. GCal-only events still open in Google Calendar.

---

### 3. Calendar Integration with Google Calendar (March 2026)
**Labels:** `feature`, `calendar`, `google-api`

Per-project Schedule tab and master Calendar page with full Google Calendar sync.

**Completed:**
- [x] Backend: `calendarRoutes.ts` — `/api/calendar/calendars` (list studio@vil.nz calendars), `/api/calendar/events` (GCal overlay by date range), per-calendar failure logging
- [x] Backend: `/api/projects/:id/schedule` — CRUD for schedule events (Tasks with `eventType`); creates/updates/deletes corresponding Google Calendar events
- [x] Backend: `/api/schedule` — all events across all projects (used by master Calendar page)
- [x] Backend: Create/update/delete events in Google Calendar (via `studio@vil.nz` refresh token)
- [x] Database: `Task` extended with `eventType`, `eventCalendarId`, `eventAssignee`, `startTime`, `endTime`, `duration`, `googleEventId`
- [x] Frontend: `ScheduleTab.tsx` — per-project Schedule tab with React Big Calendar; event creation/edit modal; suggested durations per type; calendar colour key; upcoming + unscheduled event lists
- [x] Frontend: `CalendarPage.tsx` — master calendar view with GCal events overlay; calendar filter dropdown; colour-coded by calendar; deduplication; all-day events parsed as local time (dayjs); event detail modal; GCal events open in Google Calendar
- [x] Events include VisualOS link in Google Calendar description
- [x] GCal overlay error handling: per-calendar failures logged; frontend errors surfaced via console.warn

**Deferred (not built):**
- Kanban board filtered by staff/all tasks
- Bi-directional sync (changes made directly in Google Calendar reflected back in VisualOS)

---

### Scheduled Design Approval Emails (March 2026)
**Labels:** `feature`, `design-approval`, `scheduler`

- Clock icon on design file cards opens a DateTimePicker to queue the approval email for a future time
- `DesignFile.scheduledSendAt` field; server-side 60s poll in `approvalScheduler.ts` fires emails and sets `approvalSentAt`
- Cancel X revokes unused tokens
- Timezone-aware (NZ local → UTC ISO before sending to backend)

---

### 4. Client Portal — Design Approval (Customer-Facing) (February 2026)
**Labels:** `feature`, `google-api`, `customer-facing`, `security`

Token-gated portal for client design review and approval.

**Completed:**
- [x] `ProjectContact` model — per-project contacts (name, email, isPrimary); CRUD via `/api/projects/:id/contacts`
- [x] `DesignApprovalToken` model — 96-char hex token, 14-day expiry, one per contact per file send
- [x] `DesignApproval` model — MFA-verified approval record; stores 6-digit code + expiry, final status
- [x] `PortalAuditLog` model — structured event log (token_accessed, mfa_sent, mfa_verified, approval_submitted, feedback_submitted, token_reissued) with IP + user agent
- [x] `Task` extended: `source`, `sourceReferenceId`, `portalTokenId`; feedback tasks created as `draft`, promoted to `pending` on submission; all task queries exclude drafts
- [x] `DesignFile` extended: `approvalSentAt`, `sentByUserId`
- [x] `SystemSettings` extended: `termsUrl` (seeded with Notion T&C URL); admin can update via Settings page
- [x] `utils/sendEmail.ts` — Gmail API via `studio@vil.nz`; RFC 2047 subject encoding; base64 MIME; auto-labels with `VisualOS/JOB-{id}` (MFA emails excluded); 4 templates
- [x] `utils/portalAudit.ts` — non-fatal structured event logging
- [x] `POST /api/projects/:id/design-files/:fileId/send-approval` — generates tokens, sends emails, falls back to Xero contact if no project contacts
- [x] `GET /api/projects/:id/design-files/:fileId/approvals` — returns tokens with nested approvals
- [x] `GET /portal/:token` — validates token, returns portal data including `termsUrl` and saved feedback items
- [x] `GET /portal/:token/pdf` — streams Drive file using sentBy user's refresh token (no public Drive access)
- [x] `POST /portal/:token/request-mfa` — generates 6-digit code, sends MFA email (no job label)
- [x] `POST /portal/:token/verify-mfa` — validates code + expiry
- [x] `POST /portal/:token/approve` — requires MFA; updates `DesignFile.status`; promotes draft feedback tasks to pending; sends team notification email
- [x] `POST /portal/:token/feedback` + `PATCH` + `DELETE` — feedback items saved as draft tasks; editable until approval submitted
- [x] `POST /portal/:token/reissue` — issues new token for expired links
- [x] Frontend: `Portal.page.tsx` — three-mode state machine (idle → changes → complete); restores state from server on load
- [x] Frontend: `ApprovalActions`, `MFAConfirmationModal`, `FeedbackEntryList`, `FeedbackEntryForm`, `DesignViewer`, `PortalLayout`, `TokenExpiredScreen`
- [x] Frontend: `ProjectContactsPanel` in project Details tab
- [x] Frontend: `DesignTab` updated — Send for Approval button, per-contact approval status badges
- [x] Frontend: portal route outside AppShell in `App.tsx`
- [x] Frontend: admin Settings page — editable T&Cs URL field

---

### 21. Drive Folder Picker Smart Defaults + Admin Settings
**Labels:** `feature`, `google-api`, `drive`, `admin`

- Prisma: `isAdmin Boolean @default(false)` added to `User`; `bren@vil.nz` + `bev@vil.nz` seeded as admins in migration
- Prisma: `SystemSettings` model added (singleton id=1); `baseDriveFolderId` + `baseDriveFolderName` fields
- Backend: `settingsRoutes.ts` — `GET /api/settings` (all authenticated), `PATCH /api/settings/base-drive-folder` (admin only)
- Backend: `ensureAdmin` middleware added (`utils/ensureAdmin.ts`)
- Frontend: `Settings.tsx` — admin-only section with `DrivePickerButton` to set base job folder
- Frontend: `BrandAssetsTab.tsx` — passes `baseDriveFolderId` as `defaultParentFolderId` to the folder picker
- Frontend: `Project.tsx` (`DriveFolderPicker`) — passes `contact.driveFolderId` as `defaultParentFolderId`
- Frontend: `DrivePickerButton` — generic reusable component; accepts `linkedFolderId`, `linkedFolderName`, `defaultParentFolderId`, `isSaving`, `onFolderSelected`
- `types/user.ts`: `isAdmin` added to `User` interface; `SystemSettings` interface added

Drive folder hierarchy:
```
Base Job Folder (set by admin in Settings)
  └── Client Folder (set per-contact in Brand Assets tab)
       └── Project Folder (set per-project in Details tab)
```

---

### 19. Silent Drive Token Refresh
**Labels:** `feature`, `google-api`, `drive`, `auth`

- Backend: `GET /auth/drive-token` — uses `google_refresh_token` from session user, exchanges via googleapis `OAuth2Client`, returns `access_token`
- Frontend: `src/utils/driveToken.ts` — `getDriveAccessToken()` calls the endpoint and returns the token
- All Drive picker components (`DriveFolderPicker`, `DrivePickerButton`) use `getDriveAccessToken()` instead of GIS popup
- Token errors shown as toast notification

---

### 18. Persistent Sessions (PostgreSQL Session Store)
**Labels:** `fix`, `infrastructure`, `auth`

- Replaced in-memory `MemoryStore` with `connect-pg-simple` PostgreSQL session store
- Sessions now survive backend restarts and container rebuilds — users stay logged in
- `session` table created via `createTableIfMissing: true` (Prisma drift resolved with manual migration)
- `pg.Pool` using `DATABASE_URL` wired into session store in `index.ts`

---

### 17. Phone Message Feature
**Labels:** `feature`, `tasks`, `ux`

- Prisma: 5 new fields on `Task` — `isPhoneMessage`, `callerName`, `callerPhone`, `callerEmail`, `takenAt`
- Migration: `add_phone_message_to_task` applied
- Backend: both `POST /tasks` and `POST /projects/:projectId/tasks` accept + persist new fields
- Backend: `GET /xero/projects` Contact include extended with `emailAddress` + `phones` for auto-fill
- Frontend: `PhoneMessageModal` — fetches users + all projects on open, captures `takenAt` at button-press time, auto-fills phone (MOBILE preferred, then DEFAULT) + email from project contact
- Frontend: "Take a Message" button in sidebar
- Frontend: My Tasks shows 📞 icon, caller phone/email below subtitle, `takenAt` timestamp in place of due date
- Frontend: `ITask` + `IProject` types updated
- Fix (March 2026): "Assigned to" picker switched from `/users` (app login accounts) to `/staff` (active staff members); payload changed from `assignedToId` to `staffMemberId` — consistent with all other task assignment flows

---

### 20. Bidirectional Project → Contact Navigation
**Labels:** `fix`, `ux`, `contacts`

- `Project.tsx`: contact name displayed as `<Link to="/contacts/:contactId">` using `xeroContactName` as label
- Contact email shown alongside link when available via `project.contact.emailAddress`
- `GET /api/xero/projects/:id` updated to include `Contact` relation and remap it to lowercase `contact` in response

---

### 16. Drive Folder Picker for Projects
**Labels:** `feature`, `google-api`, `drive`

- Prisma: `driveFolderId` + `driveFolderName` added to `Project` model, migration applied
- Backend: `PATCH /projects/:id` extended to accept `{ driveFolderId, driveFolderName }` independently of status
- Frontend: `DriveFolderPicker` component — loads GIS + gapi scripts, initialises token client lazily on click, opens folder-only Google Picker
- Frontend: `IProject` type updated with optional `driveFolderId` / `driveFolderName`
- Frontend: `VITE_GOOGLE_CLIENT_ID` + `VITE_GOOGLE_API_KEY` added to env files
- Details tab: shows "Link Drive Folder" button when unlinked; folder name as clickable Drive link + "Change" button when linked
- GCP: Google Picker API enabled; `http://localhost:5173` + `https://visualos.vil.nz` added to Authorized JavaScript origins

---

### 15. Site Survey Tab — Phase 2 (Google Maps Integration)
**Labels:** `feature`, `survey`, `google-api`, `maps`

- Google Places Autocomplete on the address field (`@googlemaps/js-api-loader` v2, functional API)
- Selecting an autocomplete result populates `address`, `lat`, `lng` — manual typing clears lat/lng
- `lat`/`lng` included in `saveSurvey` PUT payload
- "Open in Google Maps" link button next to Save (opens `https://maps.google.com/?q=lat,lng` in new tab)
- Falls back gracefully — link hidden when no lat/lng stored
- `VITE_GOOGLE_MAPS_API_KEY` added to `.env.development`
- `@types/google.maps` added as dev dep; `tsconfig.json` updated with `"google.maps"` in types array

---

### 5. Site Survey Tab — Phase 1
**Labels:** `feature`, `survey`

- `SiteSurvey` and `SurveyPhoto` Prisma models + migration
- Backend CRUD routes (`GET`, `PUT` per project, `POST` photo upload, `DELETE` photo)
- Photo upload via multer (20MB limit), stored in `uploads/survey-photos/`, served as static files
- Frontend: Survey tab added to project detail page (`/projects/:id/survey`)
- Address text field + notes textarea, persisted via upsert
- Photo upload modal with width/height (mm) and notes per photo
- Thumbnail grid with lightbox on click, delete button per photo
- TypeScript types: `ISiteSurvey`, `ISurveyPhoto`

---

### 1. Contacts System
**Labels:** `feature`, `xero-api`, `contacts`, `webhook`

- Prisma schema: `Contact`, `ContactAddress`, `ContactPhone`, `ContactPerson` models
- `xeroClient.ts` shared helper — single source of truth for Xero token handling and refresh
- `POST /api/contacts/sync` — bulk import all Xero contacts with pagination
- `POST /api/webhooks/xero` — HMAC-verified webhook receiver, handles contact CREATE/UPDATE events
- `GET /api/contacts` — paginated, searchable contact list
- `GET /api/contacts/:id` — full contact detail with all relations
- `POST /api/contacts` — create contact in Xero first, then store locally
- Project model: `contactId` FK to local `Contact`
- Frontend: `Contacts.page.tsx` — searchable list with Sync from Xero button and New Contact modal
- Frontend: `ContactDetail.page.tsx` — read-only detail page with Overview, People, Addresses, Projects tabs
- Frontend: "Edit in Xero" deep-link button
- Deployed and verified in production — 572 contacts synced, webhook confirmed working

**Notes:**
- Xero OAuth scopes updated to `accounting.contacts`
- Single tenant — `XERO_TENANT_ID` stored in env
- Webhook registered at `https://vis.vil.nz/api/webhooks/xero`
- `xeroContactName` column still present in Project — see backlog item #14 for cleanup

---

### 9. My Tasks Page
**Labels:** `feature`, `tasks`, `dashboard`

- Backend: `GET /api/tasks/mine` returning user's tasks with project info
- Backend: `POST /api/tasks` for standalone tasks (no project required)
- Frontend: `/my-tasks` route and page component
- Frontend: Tasks grouped by due date / priority / project / none; sort + filter; show/hide completed toggle
- Frontend: Inline complete toggle with optimistic update
- Frontend: Project + customer name link (→ details or design tab)
- Frontend: Standalone "personal" tasks with no project
- Frontend: New task via reusable `TaskModal`
- Sidebar nav "My Tasks" link with incomplete task count badge

---

### 8. Project Tasks Tab
**Labels:** `feature`, `tasks`, `ux`

- Prisma migration: `dueDate` and `priority` on `Task`
- Backend: `GET /api/users` endpoint
- Backend: task CRUD with new fields
- Frontend: Tasks tab on project detail page
- Frontend: Task list grouped by status
- Frontend: Add/edit task modal (reusable `TaskModal` component)
- Frontend: Assignee dropdown, due date picker, priority badge
- Frontend: Inline status toggle
- Design tab retains lightweight per-version task lists

---

### 7. Kanban Drag & Drop for Project Status
**Labels:** `enhancement`, `ux`, `kanban`

- `@hello-pangea/dnd` configured with DragDropContext, Droppable, Draggable
- Optimistic UI updates
- Backend PATCH sync on drop
- Error handling with revert
- Drop zone highlight (`isDraggingOver` background)
- Toast notification on status change

---

### 10. Standardise API Routes to RESTful Plural Nouns
**Labels:** `enhancement`, `api`, `refactoring`

- `/api/contacts` added as plural from the start
- `/api/projects` and `/api/notes` plural mounts added to `index.ts`
- Frontend updated: `/api/project` → `/api/projects` and `/api/note` → `/api/notes`
- Singular alias routes (`/api/project`, `/api/note`) removed from `index.ts`

---

### 2. Docker Dev/Prod Compose Split + GitHub Actions CI/CD
**Labels:** `enhancement`, `infrastructure`

- `docker-compose.yml` is now production base (no volume mount, runs `npm start`, migrations on startup)
- `docker-compose.dev.yml` dev overrides: mounts `.:/app`, runs `npm run dev` with hot reload
- Backend GitHub Actions deploy workflow: push to `main` → SSH → git pull → docker rebuild
- Frontend GitHub Actions deploy workflow: push to `master` → Vite build in CI → scp dist/ to NAS

---

### Miscellaneous Earlier Fixes & QoL
- **Email dark mode fix** — light mode island for HTML email bodies
- **Design Approval System Phase 1** — internal workflow (Drive file picker, version history, approve/request changes, per-version task lists)
- **Fix: CLOSED/DRAFT project viewing** — `GET /api/xero/projects/:id` was hardcoded to `xeroStatus: 'INPROGRESS'`, blocking CLOSED projects; fixed to `findUnique` with no status filter
- **Fix: Xero sync 500** — Xero Projects API rejects `DRAFT` as a states filter; removed DRAFT fetch from sync, removed Draft filter/badges from UI
- **QoL nav + project creation** — removed "New Project" nav link, added outline button in sidebar action stack; removed "New" column from Kanban; added "New Project" button to Home page and contact detail page
- **Reusable `notify` utility** — `@mantine/notifications` wrapper with `success`, `error`, `info` methods
