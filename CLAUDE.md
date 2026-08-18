# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ CRITICAL RULES — READ FIRST

- **NEVER merge a PR.** Ever. Under any circumstances. Not even if asked indirectly.
- After opening a PR, STOP. Say "PR is open at [url] — ready to merge?" and wait.
- Merging requires the user to explicitly say "merge" or "yes merge it" in chat.
- `gh pr merge` is FORBIDDEN unless the user has said "merge" in this session.

## Project Overview

**VisualOS** is a signage project management web application for VIL. It integrates with Xero (projects/contacts), Google Workspace (Gmail, Calendar, Drive), and Synology NAS (file storage).

- Backend: `https://vis.vil.nz` (Express + PostgreSQL)
- Frontend: `https://visualos.vil.nz` (React)

## Monorepo Structure

```
projects/
├── projects-backend/   # Express API, port 3001
└── projects-frontend/  # React + Vite SPA, port 5173
```

## Backend Commands (`projects-backend/`)

```bash
npm run dev             # Hot-reload dev server (tsx watch)
npm run build           # Compile TypeScript to dist/
npm start               # Run compiled app

# Database
npx prisma generate     # Regenerate Prisma client after schema changes
npx prisma migrate dev --name <name>   # Create + apply new migration
npx prisma migrate deploy              # Apply migrations in production
npx prisma studio       # DB browser at http://localhost:5556

# Docker (includes PostgreSQL, Express API, Prisma Studio)
# Dev (hot-reload with volume mount):
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
# Production (on NAS — uses compiled npm start, runs migrations on startup):
sudo docker compose up --build -d
```

## Frontend Commands (`projects-frontend/`)

```bash
yarn dev             # Vite dev server at http://localhost:5173
yarn build           # TypeScript check + Vite build
yarn preview         # Preview production build

yarn test            # Full suite: typecheck + prettier + lint + vitest + build
yarn vitest          # Unit tests only (101 tests across 13 files)
yarn vitest:watch    # Watch mode
yarn vitest --reporter=verbose   # Verbose output — shows each test name
yarn typecheck       # tsc --noEmit
yarn lint            # ESLint + StyleLint
yarn prettier:write  # Auto-fix formatting

yarn storybook       # Component explorer at port 6006
```

> Note: frontend uses **yarn**, backend uses **npm**.
> Test environment: **happy-dom** (replaces jsdom — avoids ERR_REQUIRE_ESM from html-encoding-sniffer).
> Test files live alongside source files as `*.test.tsx` / `*.test.ts`. Setup in `vitest.setup.mjs`.

## Architecture

### Backend (`src/`)

- **`index.ts`**: Express app entry. Configures CORS, session cookies, Passport Google OAuth, mounts all routes. The Google verify callback gates login through `checkStaffLogin()` — no hardcoded allowlist.
- **`routes/`**: One file per domain:
  - `projectRoutes` — CRUD + Drive folder creation + DELETE (cascading)
  - `contactRoutes` — Xero contact sync + webhook
  - `taskRoutes` — task CRUD; `PATCH /tasks/:id` accepts `projectId` to reassign or unlink a task from a project
  - `noteRoutes` — project notes
  - `designFileRoutes` — Drive-linked design files + approval send
  - `gmailRoutes` — Gmail thread list/view + search + label-thread
  - `xeroRoutes` — Xero OAuth + project/contact sync
  - `authRoutes` — Google OAuth, session status, Drive token exchange
  - `settingsRoutes` — SystemSettings CRUD; reads gated on `settings.tab.admin.view` / `settings.tab.templates.view`, writes on the matching `.edit`
  - `userRoutes` — current user info
  - `projectContactRoutes` — per-project contacts (`/api/projects/:id/contacts`)
  - `portalRoutes` — public token-gated portal (no auth middleware) — validate token, PDF proxy, request-mfa, verify-mfa, approve, feedback CRUD, reissue. Mounted at `/portal` (no `/api` prefix).
  - `featureRequestRoutes` — staff feature request submissions (`GET`/`POST` any authed user, `PATCH` needs `settings.tab.backlog.edit`)
  - `surveyRoutes` — site survey + photo upload to Drive
  - `deliverableRoutes` — deliverables CRUD
  - `completionPhotoRoutes` — completion photo upload to Drive
  - `calendarRoutes` — Google Calendar list + GCal events overlay (`/api/calendar/calendars`, `/api/calendar/events`); per-project schedule CRUD (`/api/projects/:id/schedule`); all-projects schedule (`/api/schedule`); syncs events to Google Calendar via `studio@vil.nz`
  - `materialRoutes` — materials/products
  - `taxonomyRoutes` — CRUD for `TaxonomyItem` (`/api/taxonomy/:type`); PATCH cascades renames to `Project.status` or `Material.category` in a DB transaction; valid types: `project_stage`, `material_category`, `task_type`
  - `staffRoutes` — CRUD for `StaffMember`. `GET /` is open to any authed user (task pickers, Shop Floor, schedule all need it) but `costPricePerHour` is stripped without `action.staff.view-cost-rates`; writes need `settings.tab.staff.edit`; 409 on duplicate email
  - `timesheetRoutes` — `GET /` (all staff entries, filterable by dateFrom/dateTo/staffMemberId/projectId; needs either `settings.tab.timesheets.view` or `project.tab.timesheets.view`, since it serves both surfaces), `GET /mine` (current user's entries resolved via email → StaffMember), `POST /` (create manual entry), `PATCH /:id` (supports notes field), `DELETE /:id` — both limited to your own entries without `action.timesheet.edit-others`
  - `shopfloorRoutes` — `GET /tasks` (today's tasks for a staff member — active, future, completed; params: staffMemberId, localDate, utcOffsetMinutes); `POST /tasks/:id/start|stop|complete|undo`
  - `verifoneRoutes` — `POST /sync` (`settings.tab.eftpos.edit` — fetches new Verifone EFTPOS reports), `GET /transactions` (`settings.tab.eftpos.view` — paginated transaction list by date range)
  - `vinylRoutes` — vinyl layout calculations per-project at `/api/projects/:projectId/vinyl-calculations` (CRUD)
  - `quoteRequestRoutes` — public (no auth) `POST /api/quote-requests`; origin-locked to `visualindustrie.co.nz`; rate-limited 10/IP/hour; fuzzy contact matching; creates Project + Note + Task + `QuoteRequest` audit record; sends internal notification email; mounted before auth routes
  - `invoiceRoutes` — per-project Xero invoice + quote create/link/unlink/refresh (`/api/projects/:id/invoice`, `/invoices/link`, `/invoices/:invoiceId/refresh`, `/quote`, etc); captures `ProjectInvoice.xeroInvoiceDate` from the Xero invoice `Date` on create/link/refresh
  - `roleRoutes` — `GET /api/permissions` (any authed), roles CRUD, `GET /api/roles/users` (login accounts), `PATCH /api/users/:id/role`. Invalid permission keys are stripped on write, not rejected, so a stale client can't wipe keys it doesn't know about
  - `financialOverviewRoutes` — `GET /api/financial-overview?month=YYYY-MM` (`page.financial-overview`; invoiced contribution vs monthly budget: coverage, pace, per-invoice drill-down, gaps report); `POST /api/admin/backfill-invoice-dates` (one-off — fills `xeroInvoiceDate`/`xeroTotal` on existing invoices from Xero; **remove after running in prod**). All money maths lives in `utils/financialOverview.ts`
- **`routes/xeroClient.ts`**: Shared Xero token handling used across routes.
- **`utils/ensureAuthenticated.ts`**: Auth middleware applied to all protected routes.
- **`utils/staffLoginGate.ts`**: `checkStaffLogin(prisma, email)` — the login gate. Returns `{ allowed }` plus a denial reason (`no_email` / `not_staff` / `inactive`) and the normalised email. `logDeniedLogin()` writes the warn-level audit line. The user-facing message is deliberately vague about which check failed.
- **`utils/sendEmail.ts`**: Gmail API email sending via `studio@vil.nz` shared inbox. RFC 2047 subject encoding, base64 MIME body, auto-labels sent messages with `VisualOS/JOB-{projectId}` (MFA code emails excluded).
- **`utils/emailTemplates.ts`**: `renderTemplate(prisma, key, context)` — fetches `EmailTemplate` from DB and interpolates shortcodes. `TEMPLATE_SHORTCODES` registry defines available shortcodes per template key for the admin editor. Composite shortcodes (`[viewApproveButton]`, `[statusBadge]`) are built automatically from context.
- **`utils/portalAudit.ts`**: `logPortalEvent()` — writes structured events to `PortalAuditLog` (token_accessed, mfa_sent, mfa_verified, approval_submitted, etc). Failures are non-fatal.
- **`utils/financialOverview.ts`**: pure, unit-tested money maths for the Financial Overview — `materialsCost`, `expensesCost`, `invoiceContributions` (pro-rates a project's direct cost across its invoices by value; labour deliberately left in, since wages are a Budget line), `normaliseBudgetToMonthly`, `elapsedFraction`/`aucklandDateString` (Pacific/Auckland via `Intl`, no dayjs), `coverage`, `pace`, and `filterProjectsMissingInvoice` (cutoff filter for the gaps list).
- **`permissions/registry.ts`**: source of truth for permission keys, grouped into `pages` / `project-tabs` / `contact-tabs` / `settings-tabs` / `actions`. Keys are **permanent** — rename the `label`, never the key. Tabs are `.view`/`.edit` pairs; `.edit` without `.view` grants nothing. Unknown keys are ignored on read and stripped on write. Exposed via `GET /api/permissions`.
- **`utils/permissions.ts`**: `rolePermissions(role)` (expands `isSystemAdmin` to every key, filters unregistered ones) and `hasPermission(user, key)`. Passport's `deserializeUser` includes the role, so a check is an array lookup, not a query.
- **`utils/requirePermission.ts`**: `requirePermission(key)` and `requireAnyPermission(...keys)` route guards — 401 unauthenticated, 403 `{ error: 'forbidden', requiredPermission }`.
- **`utils/roleGuards.ts`**: pure lockout rules for role CRUD and assignment (no self-role-change, no removing your own Roles access, last admin protected, built-in roles undeletable, default role undeletable). Returns a message or null; the route turns it into a 409.
- **`utils/sanitiseStaff.ts`**: strips `costPricePerHour` from staff responses unless the caller holds `action.staff.view-cost-rates`. `GET /api/staff` stays open to all authed users (task pickers, Shop Floor, schedule need it), so the field is removed per-caller instead.
- **`utils/timeEntryAccess.ts`**: time entry ownership — edits/deletes are limited to your own entries unless you hold `action.timesheet.edit-others`.
- **`utils/financialToggles.ts`**: `financialTogglePatch(body)` — shared partial-update builder for the persisted Financial tab toggles (`financialUseActual`/`financialChargeable`), used by the Task / ProjectMaterial / ProjectExpense PATCH routes.
- **`prisma/schema.prisma`**: Source of truth for data models.

**Notification email recipient resolution** (`resolveNotificationRecipient` in `projectRoutes.ts`): resolves in order — (1) primary project contact (`isPrimary=true`), (2) any project contact (first added), (3) organisation contact via `project.xeroContactId → Contact.emailAddress`. Use this helper for all job notification emails. Design approval emails use the same pattern inline in `designFileRoutes.ts`.

Authorisation is role-based: `User.roleId` → `Role.permissions` (a flat `string[]` of registry keys). `Role.isSystemAdmin` is an implicit grant of every key, present and future. **The backend is the authority** — frontend hiding is UX only, so every gated route must also be enforced server-side. `SystemSettings.defaultRoleId` is assigned to brand-new accounts on first login; a null role fails every check.

Authentication is session-based (express-session + **connect-pg-simple** PostgreSQL session store — sessions persist across backend rebuilds/restarts). Google OAuth tokens are stored on the `User` model for Gmail/Drive access. Xero has a separate OAuth flow stored on the same `User` model.

**Who may log in** is a database question: the Google verify callback calls `checkStaffLogin()` (`utils/staffLoginGate.ts`), which allows an email only if it has a `StaffMember` record with `isActive=true`. Any domain works, so contractors on a personal address can be onboarded from Settings → Staff; unticking Active blocks the next login. Denials are logged at warn level and redirect to the frontend `/login-failed` page — no `User` row is created for a denied email.

**Drive token endpoint:** `GET /auth/drive-token` exchanges the stored `google_refresh_token` for a short-lived access token using the googleapis `OAuth2Client`. Frontend calls this via `getDriveAccessToken()` utility (`src/utils/driveToken.ts`) before opening the Google Picker — silently refreshes without user interaction.

The Xero webhook endpoint (`POST /api/webhooks/xero`) uses HMAC-SHA256 verification — no session auth.

### Frontend (`src/`)

- **`main.tsx` → `App.tsx` → `Shell.page.tsx`**: App entry chain. `App.tsx` splits into two top-level routes: `/portal/:token` (no AppShell, no auth — uses `Portal.page.tsx`) and `/*` (full app inside `ShellPage`). `Shell.page.tsx` owns the `AppShell` layout, navigation, all authenticated routes, and renders `FeatureRequestModal` + `PhoneMessageModal` globally.
- **`pages/`**: Full-page route components:
  - `Home.page.tsx` — searchable/filterable/sortable project list table; rows stack on mobile; X button on each row to permanently delete a project; search is NOT persisted (other filters are); search field has X clear button + ESC-to-clear
  - `Dashboard.page.tsx` — Kanban board (drag-and-drop status columns)
  - `Project.tsx` — multi-tab project detail view; header has "View in Xero" + "Delete project" buttons right-aligned; Notes tab is hidden (backlog #37 to move inline to Details)
  - `Portal.page.tsx` — public client portal (no auth)
  - `LoginFailed.page.tsx` — where the backend's OAuth failure redirect lands (`/login-failed`, ungated); explains the two staff-gate denial causes and offers a retry
  - `CalendarPage.tsx` — master calendar view (all projects); GCal events overlay (public holidays, personal events) deduplicated against VisualOS `googleEventId`; all-day events parsed as local time; clicking VisualOS event shows detail modal with "Open project schedule" link; clicking GCal event opens in Google Calendar; calendar filter dropdown; colour key
  - `Settings.tsx` — tabbed settings page: General, Admin, Templates, EFTPOS, Staff, Lists, Vehicles, Budget, Timesheets, Mileage, Roles, Backlog, Releases. Tab visibility is driven by `SETTINGS_TABS` filtered on each tab's `settings.tab.*.view` key — no `isAdmin` checks
  - `ShopFloor.page.tsx` — tablet-optimised production floor view at `/shopfloor`; no AppShell/nav; `StaffPickerOverlay` on first visit (selection persisted to localStorage); shows active, upcoming, and completed tasks for the selected staff member; start/stop/complete/undo actions with optimistic updates; task type + project filter chips; auto-refresh every 60s; `UnavailableScreen` if backend unreachable on initial load; design preview modal (Google Drive iframe, full-screen)
  - `MyTimesheet.page.tsx` — staff time entry view at `/my-timesheet`; entries grouped by day with totals; date range filter (DatePickerInput); add/edit/delete entries; manual vs timer badge
  - `AdminTimesheets.page.tsx` — admin time entry view at `/timesheets`; all staff entries in a table; filters for date range and staff member; totals summary per staff member at bottom
  - `FinancialOverview.page.tsx` — page at `/financial-overview`, gated on `page.financial-overview` (sidebar link and route both); month picker (Auckland default) + back/forward arrows; header stat row; two `RatioGauge`s (Coverage, Pace — Pace hidden for future months); per-invoice drill-down; gaps panel (invoices missing a total with one-click refresh, and expectsInvoice-stage projects with no invoice). Reads `GET /api/financial-overview?month=YYYY-MM`
- **`components/`**: Reusable UI:
  - `Tasks/TaskList`, `Tasks/TaskModal` (`showProjectPicker` prop shows project dropdown — used for both add and edit on My Tasks page; edit pre-fills current project), `Tasks/TaskItem` (shows task type badge)
  - `Project/Tabs/DesignTab` — design file upload + approval send
  - `Project/Tabs/EmailTab` — Gmail thread viewer + find & link emails panel
  - `Project/Tabs/SurveyTab` — site survey with address autocomplete, notes, and photo capture (camera via `getUserMedia` or multi-file upload); 3-phase modal: source picker → live camera → preview + notes/dimensions; multi-file: thumbnail grid + sequential upload with progress
  - `Project/Tabs/ScheduleTab` — per-project schedule using React Big Calendar; create/edit/delete events; syncs to Google Calendar; GCal overlay (deduped by `googleEventId`); colour key; upcoming + unscheduled lists
  - `Project/Tabs/CompletionPhotosTab` — completion photo grid + lightbox; same 3-phase camera/upload modal as SurveyTab; multi-file upload supported
  - `Project/Tabs/TimesheetTab` — all time entries for the project across all staff; add/edit/delete manual entries; task picker pre-filtered to project tasks; total duration in header
  - `Project/Tabs/CalendarTab`, `DeliverablesTab`
  - `Project/Tabs/DetailsTab` — description editor, status combobox (taxonomy-driven), Save button, Notify Customer button (enabled/disabled by `canNotifyCustomer` flag on current stage), Drive folder picker, project contacts
  - `Project/DriveFolderPicker` — Drive folder picker + auto-create folder hierarchy
  - `Project/ProjectContactsPanel` — per-project contacts (add/edit/delete)
  - `Project/DeleteProjectModal` — permanent delete confirmation modal
  - `PhoneMessageModal` — take-a-message form
  - `DrivePickerButton` — Google Picker integration
  - `FeatureRequestModal` — staff idea/feedback submission (auto-captures user + page URL)
  - `Settings/BacklogPanel` — feature request list; admins can archive and edit inline
  - `Settings/ReleasesPanel` — static changelog grouped by date
  - `Settings/EmailTemplatesPanel` — admin editable email + page templates; Select dropdown to pick template; shortcode click-to-insert; HTML preview
  - `Settings/StaffPanel` — staff CRUD (name, email, colour, Xero/GCal mapping, active toggle); admin only
  - `Settings/RolesPanel` — role list + permission matrix (one card per group, View/Edit pair per tab, per-group "All", sticky Save/Cancel). Admin renders all-checked and disabled with an explanatory alert. Includes the "Login accounts" list for assigning a role to an account with no staff record. Matrix shaping and toggle rules live in `Settings/rolePermissionMatrix.ts` (pure, unit-tested) — ticking Edit auto-ticks View, unticking View unticks Edit
  - `Settings/TaxonomyPanel` — editable taxonomy sections; `TaxonomySection` is reusable per type; project stages support flags: `showInKanban`, `closesXero`, `sendNotification` (auto-email on status change), `canNotifyCustomer` (enables manual Notify button), `showOnOverview`, `expectsInvoice` (drives the Home "No invoice" badge + Financial Overview gaps list); badge colours from Mantine colour names
- **`components/ShopFloor/`**: Shop floor tablet components — `ShopFloorTaskCard` (task card with start/stop/complete/undo buttons, time-tracking progress bar against `estimatedMinutes`, design preview button, staff badge, task type badge); `ShopFloorHeader` (staff name + colour dot, last-refreshed time, "Switch user" button); `StaffPickerOverlay` (full-screen staff picker on first visit); `TaskTypeFilterBar` (filter chips by task type); `CompletedTaskSection` (collapsible completed tasks with undo).
- **`components/UnavailableScreen`**: Shown on Shop Floor if backend is unreachable on initial load (network error with no response).
- **`components/Portal/`**: Portal-specific — `PortalLayout`, `DesignViewer` (PDF iframe proxy), `ApprovalActions` (shows Approve/Request Changes; Approve expands inline with option picker + T&Cs + confirm button; POSTs directly to `/approve`, no MFA modal), `FeedbackEntryList`, `FeedbackEntryForm` (Textarea — one change per line, each submitted separately), `TokenExpiredScreen`.
- **`types/`**: Shared TypeScript interfaces (`IProject`, `ITask`, `IDesignFile`, `Contact`, `SystemSettings`, `IProjectContact`, `IDesignApproval`, `IDesignApprovalToken`, `IFeedbackItem`, `ITaxonomyItem`).
- **`types/shopfloor.ts`**: `IShopFloorTask`, `IShopFloorResponse`, `IShopFloorStaff`, `ITimeEntry`.
- **`types/timesheet.ts`**: `ITimesheetEntry`, `entryDurationMinutes()`, `formatDuration()`.
- **`types/taxonomy.ts`**: `ITaxonomyItem` interface + `taxonomyLabel(item)` helper (returns `label ?? name`).
- **`contexts/PermissionsContext.tsx`**: `usePermissions()` → `{ can, canAny, permissions, role, user, isLoggedIn, isLoading, refresh }`. Fed from `/auth/status`, which returns the **fully resolved** key list (admins get every key expanded), so nothing special-cases admins. Supersedes `useAuthStatus` as the source of the current user. In dev, `can()` warns about a key the server doesn't know.
- **`components/Permissions/`**: `<Can permission=... anyOf=... fallback=...>` for controls, `<RequirePermission permission=... what=...>` for route elements, `<NoAccessScreen>` for the denied case (a screen, never a redirect — redirects make deep links look broken and loop when home is also unpermitted).
- **`types/permissions.ts`**: `PermissionKey` string-literal union mirroring the backend registry — keep the two in sync when adding a key.
- **`utils/permissionTabs.ts`**: `visibleTabs()` / `resolveActiveTab()` — filter a tab config array by permission and fall back to the first permitted tab (null when none, which the page turns into `<NoAccessScreen>`).
- **`utils/forbiddenInterceptor.ts`**: global axios 403 handler; toasts the middleware's `{ error: 'forbidden' }` shape only, so a route's own 403 messages survive.
- **`hooks/useTaxonomy.ts`**: `useTaxonomy(type, includeArchived?)` — fetches taxonomy items with module-level 5-min cache. Call `invalidateTaxonomyCache(type?)` after writes. Used across Home, Dashboard, Project, Materials, Deliverables.
- **`utils/notifications.ts`**: Wrapper around Mantine notifications for toast messages.
- **`utils/driveToken.ts`**: `getDriveAccessToken()` — calls `GET /auth/drive-token` to exchange the stored refresh token for a short-lived Drive access token.
- **`hooks/useAuthStatus.ts`**: Checks login state on app load.

State management is local `useState`/`useEffect` per component, except for `PermissionsContext` and `TaskContext` (the first slices of the planned global-state work, backlog #11). HTTP calls use Axios with `withCredentials: true` for cookie auth.

### Data Models (Prisma)

Key models: `User`, `Role`, `Project`, `Contact`, `Task`, `TimeEntry`, `Note`, `DesignFile`, `SystemSettings`, `EmailTemplate`, `ProjectContact`, `DesignApprovalToken`, `DesignApproval`, `PortalAuditLog`, `FeatureRequest`, `SiteSurvey`, `SurveyPhoto`, `CompletionPhoto`, `Deliverable`, `Material`, `BrandAssets`, `TaxonomyItem`, `StaffMember`.

- `User` has `roleId` → `Role`. `isAdmin` is legacy: nothing reads it, it's mirrored from `role.isSystemAdmin` on assignment, and it's dropped in backlog #40
- `Role` — named bundle of permission keys (`permissions` JSON `string[]`). `isSystemAdmin` = implicit grant of everything; `isProtected` = seeded, can't be deleted or have `isSystemAdmin` toggled (Admin + Production Staff). Takes an `organisationId` FK when multi-tenancy (#32) lands
- `Project` has a FK to `Contact` (Xero contact), plus optional `driveFolderId`/`driveFolderName`
- `Project.status` is a free-text field driven by `TaxonomyItem` (type=`project_stage`). Valid values and their behaviour come from taxonomy — do not hardcode. PATCH `/api/projects/:id` validates against the live taxonomy list.
- `TaxonomyItem` — generic extensible table (`type`, `name`, `label`, `colour`, `isArchived`, `sortOrder`, `meta` JSON). Types: `project_stage` (meta flags: `showInKanban`, `closesXero`, `sendNotification`, `canNotifyCustomer`, `showOnOverview`, `expectsInvoice`), `material_category`, and `task_type`. Renaming cascades to all downstream records in a transaction.
- `Task` can be standalone or linked to `Project` or `DesignFile`; schedule events set `eventType` (design/print/laminate/cut-apply/install/meeting/quote/invoice), `eventCalendarId`, `startTime`, `endTime`, `duration`, `googleEventId`; phone message tasks set `isPhoneMessage=true` and carry `callerName`, `callerPhone`, `callerEmail`, `takenAt`; client feedback tasks have `source='client_feedback'`, `portalTokenId`, and start as `status='draft'` until the client submits (then promoted to `pending`); all task list queries exclude `draft` status; tasks have `staffMemberId` FK to `StaffMember` — tasks assigned to a staff member appear on their Shop Floor view; `estimatedMinutes` drives the progress bar on Shop Floor task cards; `timeTrackingEnabled` flag; has `timeEntries` relation (one-to-many `TimeEntry`); `financialUseActual` (default false = Est basis) and `financialChargeable` (default true) persist the Financial tab's per-row Est/Act + Charge toggles
- `TimeEntry` — time tracking record linked to a `Task`; fields: `startedAt`, `stoppedAt` (nullable = timer still running), `manuallyAdjustedMinutes`, `isManual` (true for admin-created entries); `GET /timesheets/mine` resolves the logged-in user via email → StaffMember then filters entries by `task.staffMemberId`
- `Contact` has sub-relations: `ContactAddress`, `ContactPhone`, `ContactPerson`; `driveFolderId`/`driveFolderName` store the linked Brand Assets Drive folder
- `DesignFile` tracks Google Drive files with version history and approval status (`pending_review`, `changes_requested`, `approved`); `approvalSentAt` and `sentByUserId` set when approval email is sent; `optionsCount Int @default(1)` — staff sets how many layout options exist (1–10); shown as a `NumberInput` on the design file card
- `SystemSettings` — single-row singleton (`id=1`), stores `baseDriveFolderId`/`baseDriveFolderName` (top-level job folder), `templateDriveFileId`/`templateDriveFileName` (AI template file to copy on folder creation), `termsUrl` (T&Cs link shown in client portal approval flow), `defaultSalesAccountCode`/`defaultInvoiceDays`, and `financialOverviewStartDate` (cutoff — projects that reached their invoice-expecting stage before this NZ date are hidden from the Financial Overview gaps list; seeded to 2026-06-01; null = no cutoff; edited in Settings → Admin → Invoicing)
- `ProjectInvoice` — per-project Xero invoice link; `xeroTotal` (pre-tax) and `xeroInvoiceDate` (Xero invoice `Date`, the month-bucketing date for the Financial Overview); `ProjectMaterial` and `ProjectExpense` also carry the `financialUseActual`/`financialChargeable` toggle fields (see `Task`)
- `EmailTemplate` — editable email templates keyed by slug (`approval_request`, `mfa_code`, `approval_confirmed`, `new_token`, `job_notification`, `design_approval_confirmation`); body supports shortcodes interpolated by `renderTemplate()`
- `ProjectContact` — per-project contacts (name, email, isPrimary); used to send design approval emails
- `DesignApprovalToken` — 96-char hex token (14-day expiry) sent to each contact for portal access; one per contact per design file send
- `DesignApproval` — approval record per token; `POST /portal/:token/approve` now generates a `confirmationToken` (32-byte hex, 24h expiry) and sends `design_approval_confirmation` email instead of MFA; `GET /portal/confirm/:confirmToken` finalises the approval (sets `submittedAt`, updates `DesignFile.status`, promotes draft feedback tasks); fields: `approvedOption Int?` (which option they chose), `confirmationToken String? @unique`, `confirmTokenExpiry DateTime?`
- `PortalAuditLog` — structured event log for all portal activity (token_accessed, mfa_sent, mfa_verified, approval_submitted, etc) with IP and user agent
- `FeatureRequest` — staff-submitted feature requests; fields: `description`, `pageUrl`, `userId`, `userName` (denormalised), `status` (`open`/`archived`)
- `SiteSurvey` — one per project; stores address, lat/lng, and notes; has `SurveyPhoto` children (stored in Google Drive)
- `Deliverable` — physical sign component linked to a project; tracks type, dimensions, material, laminate, and optional cutting diagram Drive file
- `Material` — substrate/vinyl/laminate catalogue; optionally linked to Xero items
- `StaffMember` — UUID id, unique email, name, displayColour, optional `xeroUserId` and `googleCalendarId` mappings, `isActive` flag; email is used to look up the current user's staff record for `/timesheets/mine` and Shop Floor
- `Note.userId` is nullable — system-generated notes (e.g. from quote request intake) omit the userId
- `Contact.source` and `Project.source` — optional string field (`'xero'` | `'quote_request'` | null); set on new web-form leads
- `QuoteRequest` — audit log for every inbound quote form submission; stores all fields plus `matchStrategy` (`'email'` | `'fuzzy_name'` | `'none'`), `contactId`, `projectId`; `src/utils/fuzzyMatch.ts` exports `normalise()` and `tokenOverlapScore()` (70% token overlap threshold)
- Xero sync only fetches `INPROGRESS` and `CLOSED` — Xero Projects API does not accept `DRAFT` as a states filter

### External Integrations

| Integration | Purpose | Auth |
|---|---|---|
| Google OAuth | Login + Gmail + Drive scopes | Passport `passport-google-oauth20` |
| Xero API | Projects, contacts, invoices | `xero-node` SDK, tokens on User |
| Gmail API | Thread viewer + find & link in project Email tab | googleapis, shared `studio@vil.nz` account |
| Google Drive | Design file uploads, survey/completion photos, folder creation | googleapis |
| Google Picker API | Drive folder picker in project Details tab, Brand Assets tab, Settings page | Backend `/auth/drive-token` exchanges stored refresh token; `VITE_GOOGLE_API_KEY` (browser key) required |
| Google Maps / Places API | Site address autocomplete in Survey tab | `@googlemaps/js-api-loader`, Places API (New) |
| Xero Webhook | Real-time contact sync | HMAC-SHA256 verified, no session |

## Environment Variables

**Backend (`.env`):**
```
DATABASE_URL, APP_DATABASE_URL
GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET
SHARED_INBOX_EMAIL, SHARED_INBOX_REFRESH_TOKEN
XERO_CLIENT_ID, XERO_CLIENT_SECRET, XERO_WEBHOOK_KEY, XERO_TENANT_ID
COOKIE_KEY, FRONTEND_URL, BACKEND_URL, CALLBACK_URL
```

**Frontend (`.env.development`):**
```
VITE_API_BASE_URL=http://localhost:3001/api
VITE_GOOGLE_MAPS_API_KEY=   # real key goes in .env.development.local (gitignored)
VITE_GOOGLE_CLIENT_ID=      # same value as backend GOOGLE_CLIENT_ID — safe to expose
VITE_GOOGLE_API_KEY=        # browser API key with Picker API enabled
```

**Frontend (`.env.production` / server):**
```
VITE_API_BASE_URL=https://vis.vil.nz/api
VITE_GOOGLE_MAPS_API_KEY=   # set on build server or in CI
VITE_GOOGLE_CLIENT_ID=      # set via GH Actions secret
VITE_GOOGLE_API_KEY=        # set via GH Actions secret
```

**Google Cloud APIs required:**
- Maps JavaScript API
- Places API (New) — used by the Survey tab address autocomplete
- Google Picker API — used by the Drive folder picker in project Details tab

## Canvas in Mantine fullScreen Modal

- Add `styles={{ content: { display: 'flex', flexDirection: 'column' } }}` on the Modal — without it, `body: { flex: 1 }` has no flex parent and the canvas container collapses to 0px height.
- Read canvas dimensions from `canvas.offsetWidth/offsetHeight` inside render — never from React state, which may be stale when the modal animates in.
- Mobile browsers fire a synthetic `click` ~300ms after a touch. Suppress in click handlers with a `lastTouchEndRef` timestamp check: `if (Date.now() - lastTouchEndRef.current < 500) return`.

## Portal — feedback task promotion

- `draft` feedback tasks must be promoted to `pending` in `POST /portal/:token/approve`, not in `GET /portal/confirm/:confirmToken`. Customers frequently miss the confirmation email; waiting for the click leaves tasks permanently hidden on the design tab.

## Development Standards

- **Run commands individually — never chain with `&&`.** Each command must be a separate `Bash` call so output and errors are visible independently.
- **Please execute tasks one at a time**. Run one command, wait for the output, and then run the next command.
- **Always work on a feature branch.** Never commit directly to `main` (backend) or `master` (frontend). Branch from `main`/`master` using Gitflow naming: `feature/FeatureName`, `fix/BugDescription`, `chore/TaskName`. Example: `feature/CalendarIntegration`.
- **Never merge to the base branch without explicit permission.** Open a PR and wait. Do not merge until the user has tested locally and either says "merge" or explicitly authorises it. When a PR is ready, ask: "Ready to merge?" and wait for confirmation.
- **Restart express_api after every backend PR.** After opening a backend PR (and before merging), always restart the local dev container so the user can test the latest code: `docker restart express_api`. If the PR is frontend-only, no restart is needed.
- **ALWAYS update the releases list — no exceptions.** Every merged feature, fix, or refactor MUST include an entry in `projects-frontend/src/components/Settings/ReleasesPanel.tsx`. This applies to every PR, including small bug fixes. Prepend to the matching date entry or create a new one. Write in plain English from the user's perspective (what changed and why it matters to them). Commit the releases update in the same PR as the change — never as a follow-up. If a PR is raised without a releases entry, it is not complete.
- **Every new page, tab, or sensitive action needs a permission key.** Add it to `projects-backend/src/permissions/registry.ts`, mirror it in `projects-frontend/src/types/permissions.ts`, gate the route with `requirePermission()`, and gate the UI with `can()` / `<Can>`. Tabs get a `.view`/`.edit` pair; pages and actions get a single key. New keys default to **off** for every non-admin role and are implicitly granted to `isSystemAdmin` roles. Never hardcode `isAdmin`.
- **Write tests for every new function/route.** Backend: add a test in a `*.test.ts` file alongside the route. Frontend: add a Vitest unit test for any new utility or hook.
- **Prisma migration drift**: the `session` table (created by `connect-pg-simple`) causes drift warnings with `prisma migrate dev`. Workaround: create the migration SQL manually → apply via `psql` → mark applied with `npx prisma migrate resolve --applied <name>`.

## Installing New npm Packages (Backend)

When adding a new backend dependency, run it both locally and inside the container — the volume mount doesn't share `node_modules`:

```bash
npm install <package>                                       # local (updates package.json / lock)
docker exec express_api npm install                         # dev (local Docker)
sudo /usr/local/bin/docker exec express_api npm install     # production (Synology NAS)
```

## Docker Setup

`projects-backend/` has two compose files:
- **`docker-compose.yml`** — production base: no volume mount, runs `npx prisma migrate deploy && npm start`
- **`docker-compose.dev.yml`** — dev overrides: mounts `.:/app`, runs `npm run dev` with hot reload

Services:
- **PostgreSQL 15** on port 5433
- **Express API** on port 3001 (service name: `express-api`)
- **Prisma Studio** on port 5556

The app uses different `DATABASE_URL` values: `localhost:5433` when running outside Docker, `postgres:5432` (internal hostname) when running inside Docker (`APP_DATABASE_URL`).

## Deployment

Both repos deploy automatically via GitHub Actions on push to `main`/`master`.

**Backend** (`main` branch): SSH → `git pull` → `docker compose up --build -d` (migrations run on container start).

**Frontend** (`master` branch): Build in CI (Vite) → `scp` `dist/` to `/volume1/web/visualos` on NAS.

**GitHub Secrets required (both repos):** `NAS_HOST`, `NAS_USER`, `NAS_SSH_KEY`, `NAS_SSH_PORT`
**Backend only:** `NAS_BACKEND_PATH`
**Frontend only:** `VITE_API_BASE_URL`, `VITE_GOOGLE_MAPS_API_KEY`, `VITE_GOOGLE_CLIENT_ID`, `VITE_GOOGLE_API_KEY`

**NAS details:** `visualindustrie.synology.me`, SSH port `222` (forwards to 22), user `visual`, backend at `/volume1/docker/visualos`, frontend web root at `/volume1/web/visualos`.

**One-time NAS setup (already done):**
1. Generate key on NAS: `ssh-keygen -t ed25519 -C "github-actions"`
2. Add public key to NAS: `cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys`
3. Enable passwordless sudo for docker: `echo "visual ALL=(ALL) NOPASSWD: /usr/local/bin/docker" | sudo tee /etc/sudoers.d/visual-docker`
4. Add `StrictModes no` to `/etc/ssh/sshd_config` (Synology home dir permissions conflict with SSH defaults)
5. Add private key to GitHub Secrets as `NAS_SSH_KEY` on both repos

## Roadmap Notes (as of March 2026)

VisualOS is being designed with multi-tenancy in mind for future productisation. When building new features, keep the following in mind:

- **Org layer coming**: All new models should be designed to accept an `organisationId` FK when multi-tenancy (#32) is implemented. Don't hardcode Visual Industrie assumptions.
- **Integration credentials**: Moving toward per-org credential storage in DB (encrypted). Env vars remain as fallback for single-tenant mode.
- **Email/calendar abstraction**: Gmail and Google Calendar are the current providers, but the intent is to make these swappable per org. Avoid tight coupling to Google-specific APIs where possible.
- **Scaffolding system**: A data-driven project scaffolding wizard (#31) is planned. New deliverable types should be designed with configurable templates in mind.
- **Quote intake**: A public `POST /api/quote-requests` endpoint (#30) will accept form submissions from `visualindustrie.co.nz`. Origin-locked. Contact fuzzy matching against existing records.
