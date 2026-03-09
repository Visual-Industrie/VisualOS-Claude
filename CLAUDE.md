# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

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
yarn vitest          # Unit tests only (85 tests across 11 files)
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

- **`index.ts`**: Express app entry. Configures CORS, session cookies, Passport Google OAuth, mounts all routes. Contains `allowedEmails` hardcoded allowlist.
- **`routes/`**: One file per domain:
  - `projectRoutes` — CRUD + Drive folder creation + DELETE (cascading)
  - `contactRoutes` — Xero contact sync + webhook
  - `taskRoutes` — task CRUD
  - `noteRoutes` — project notes
  - `designFileRoutes` — Drive-linked design files + approval send
  - `gmailRoutes` — Gmail thread list/view + search + label-thread
  - `xeroRoutes` — Xero OAuth + project/contact sync
  - `authRoutes` — Google OAuth, session status, Drive token exchange
  - `settingsRoutes` — SystemSettings CRUD (admin-gated for writes)
  - `userRoutes` — current user info
  - `projectContactRoutes` — per-project contacts (`/api/projects/:id/contacts`)
  - `portalRoutes` — public token-gated portal (no auth middleware) — validate token, PDF proxy, request-mfa, verify-mfa, approve, feedback CRUD, reissue. Mounted at `/portal` (no `/api` prefix).
  - `featureRequestRoutes` — staff feature request submissions (`GET`/`POST` any user, `PATCH` admin only)
  - `surveyRoutes` — site survey + photo upload to Drive
  - `deliverableRoutes` — deliverables CRUD
  - `completionPhotoRoutes` — completion photo upload to Drive
  - `calendarRoutes` — Google Calendar list + GCal events overlay (`/api/calendar/calendars`, `/api/calendar/events`); per-project schedule CRUD (`/api/projects/:id/schedule`); all-projects schedule (`/api/schedule`); syncs events to Google Calendar via `studio@vil.nz`
  - `materialRoutes` — materials/products
- **`routes/xeroClient.ts`**: Shared Xero token handling used across routes.
- **`utils/ensureAuthenticated.ts`**: Auth middleware applied to all protected routes.
- **`utils/ensureAdmin.ts`**: Admin-only middleware — returns 403 if `req.user.isAdmin` is falsy.
- **`utils/sendEmail.ts`**: Gmail API email sending via `studio@vil.nz` shared inbox. RFC 2047 subject encoding, base64 MIME body, auto-labels sent messages with `VisualOS/JOB-{projectId}` (MFA code emails excluded). Fetches editable body/subject from `EmailTemplate` DB records; falls back to hardcoded HTML if template not found.
- **`utils/portalAudit.ts`**: `logPortalEvent()` — writes structured events to `PortalAuditLog` (token_accessed, mfa_sent, mfa_verified, approval_submitted, etc). Failures are non-fatal.
- **`prisma/schema.prisma`**: Source of truth for data models.

Authentication is session-based (express-session + **connect-pg-simple** PostgreSQL session store — sessions persist across backend rebuilds/restarts). Google OAuth tokens are stored on the `User` model for Gmail/Drive access. Xero has a separate OAuth flow stored on the same `User` model.

**Drive token endpoint:** `GET /auth/drive-token` exchanges the stored `google_refresh_token` for a short-lived access token using the googleapis `OAuth2Client`. Frontend calls this via `getDriveAccessToken()` utility (`src/utils/driveToken.ts`) before opening the Google Picker — silently refreshes without user interaction.

The Xero webhook endpoint (`POST /api/webhooks/xero`) uses HMAC-SHA256 verification — no session auth.

### Frontend (`src/`)

- **`main.tsx` → `App.tsx` → `Shell.page.tsx`**: App entry chain. `App.tsx` splits into two top-level routes: `/portal/:token` (no AppShell, no auth — uses `Portal.page.tsx`) and `/*` (full app inside `ShellPage`). `Shell.page.tsx` owns the `AppShell` layout, navigation, all authenticated routes, and renders `FeatureRequestModal` + `PhoneMessageModal` globally.
- **`pages/`**: Full-page route components:
  - `Home.page.tsx` — searchable/filterable/sortable project list table; rows stack on mobile; X button on each row to permanently delete a project; search is NOT persisted (other filters are); search field has X clear button + ESC-to-clear
  - `Dashboard.page.tsx` — Kanban board (drag-and-drop status columns)
  - `Project.tsx` — multi-tab project detail view; header has "View in Xero" + "Delete project" buttons right-aligned
  - `Portal.page.tsx` — public client portal (no auth)
  - `CalendarPage.tsx` — master calendar view (all projects); GCal events overlay (public holidays, personal events) deduplicated against VisualOS `googleEventId`; all-day events parsed as local time; clicking VisualOS event shows detail modal with "Open project schedule" link; clicking GCal event opens in Google Calendar; calendar filter dropdown; colour key
  - `Settings.tsx` — tabbed settings page: General, Admin (admin only), Email Templates (admin only), Backlog (all users), Releases (all users)
- **`components/`**: Reusable UI:
  - `Tasks/TaskList`, `Tasks/TaskModal`, `Tasks/TaskItem`
  - `Project/Tabs/DesignTab` — design file upload + approval send
  - `Project/Tabs/EmailTab` — Gmail thread viewer + find & link emails panel
  - `Project/Tabs/SurveyTab` — site survey with address autocomplete, notes, and photo capture (camera via `getUserMedia` or file upload); 3-phase modal: source picker → live camera → preview + notes
  - `Project/Tabs/ScheduleTab` — per-project schedule using React Big Calendar; create/edit/delete events; syncs to Google Calendar; GCal overlay (deduped by `googleEventId`); colour key; upcoming + unscheduled lists
  - `Project/Tabs/CompletionPhotosTab` — completion photo grid + lightbox; same 3-phase camera/upload modal as SurveyTab
  - `Project/Tabs/CalendarTab`, `DeliverablesTab`
  - `Project/DriveFolderPicker` — Drive folder picker + auto-create folder hierarchy
  - `Project/ProjectContactsPanel` — per-project contacts (add/edit/delete)
  - `Project/DeleteProjectModal` — permanent delete confirmation modal
  - `PhoneMessageModal` — take-a-message form
  - `DrivePickerButton` — Google Picker integration
  - `FeatureRequestModal` — staff idea/feedback submission (auto-captures user + page URL)
  - `Settings/BacklogPanel` — feature request list; admins can archive and edit inline
  - `Settings/ReleasesPanel` — static changelog grouped by date
  - `Settings/EmailTemplatesPanel` — admin editable email templates with shortcode preview
- **`components/Portal/`**: Portal-specific — `PortalLayout`, `DesignViewer` (PDF iframe proxy), `ApprovalActions` (T&Cs checkbox + Approve/Request Changes), `MFAConfirmationModal` (6-digit PinInput), `FeedbackEntryList`, `FeedbackEntryForm`, `TokenExpiredScreen`.
- **`types/`**: Shared TypeScript interfaces (`IProject`, `ITask`, `IDesignFile`, `Contact`, `SystemSettings`, `IProjectContact`, `IDesignApproval`, `IDesignApprovalToken`, `IFeedbackItem`).
- **`utils/notifications.ts`**: Wrapper around Mantine notifications for toast messages.
- **`utils/driveToken.ts`**: `getDriveAccessToken()` — calls `GET /auth/drive-token` to exchange the stored refresh token for a short-lived Drive access token.
- **`hooks/useAuthStatus.ts`**: Checks login state on app load.

State management is local `useState`/`useEffect` per component — no global state yet (planned via React Context). HTTP calls use Axios with `withCredentials: true` for cookie auth.

### Data Models (Prisma)

Key models: `User`, `Project`, `Contact`, `Task`, `Note`, `DesignFile`, `SystemSettings`, `EmailTemplate`, `ProjectContact`, `DesignApprovalToken`, `DesignApproval`, `PortalAuditLog`, `FeatureRequest`, `SiteSurvey`, `SurveyPhoto`, `CompletionPhoto`, `Deliverable`, `Material`, `BrandAssets`.

- `User` has `isAdmin Boolean @default(false)` — `bren@vil.nz` and `bev@vil.nz` are seeded as admins via migration
- `Project` has a FK to `Contact` (Xero contact), plus optional `driveFolderId`/`driveFolderName`
- `Project.status` valid values: `New`, `Design`, `Quote`, `With Customer`, `Production`, `Install`, `Invoice`, `Archived`, `On Hold`, `Completed`, `Abandoned` — Kanban shows all except `New`, `Completed`, and `Abandoned`. Setting `Completed` or `Abandoned` also closes the project in Xero.
- `Task` can be standalone or linked to `Project` or `DesignFile`; schedule events set `eventType` (design/print/laminate/cut-apply/install/meeting/quote/invoice), `eventCalendarId`, `startTime`, `endTime`, `duration`, `googleEventId`; phone message tasks set `isPhoneMessage=true` and carry `callerName`, `callerPhone`, `callerEmail`, `takenAt`; client feedback tasks have `source='client_feedback'`, `portalTokenId`, and start as `status='draft'` until the client submits (then promoted to `pending`); all task list queries exclude `draft` status
- `Contact` has sub-relations: `ContactAddress`, `ContactPhone`, `ContactPerson`; `driveFolderId`/`driveFolderName` store the linked Brand Assets Drive folder
- `DesignFile` tracks Google Drive files with version history and approval status (`pending_review`, `changes_requested`, `approved`); `approvalSentAt` and `sentByUserId` set when approval email is sent
- `SystemSettings` — single-row singleton (`id=1`), stores `baseDriveFolderId`/`baseDriveFolderName` (top-level job folder), `templateDriveFileId`/`templateDriveFileName` (AI template file to copy on folder creation), and `termsUrl` (T&Cs link shown in client portal approval flow)
- `EmailTemplate` — editable email templates keyed by slug (e.g. `approval_request`); body supports shortcodes like `[clientName]`, `[version]`, `[portalLink]`
- `ProjectContact` — per-project contacts (name, email, isPrimary); used to send design approval emails
- `DesignApprovalToken` — 96-char hex token (14-day expiry) sent to each contact for portal access; one per contact per design file send
- `DesignApproval` — MFA-verified approval record per token; stores 6-digit code, expiry, verified status, and final `status` (`approved`/`changes_requested`)
- `PortalAuditLog` — structured event log for all portal activity (token_accessed, mfa_sent, mfa_verified, approval_submitted, etc) with IP and user agent
- `FeatureRequest` — staff-submitted feature requests; fields: `description`, `pageUrl`, `userId`, `userName` (denormalised), `status` (`open`/`archived`)
- `SiteSurvey` — one per project; stores address, lat/lng, and notes; has `SurveyPhoto` children (stored in Google Drive)
- `Deliverable` — physical sign component linked to a project; tracks type, dimensions, material, laminate, and optional cutting diagram Drive file
- `Material` — substrate/vinyl/laminate catalogue; optionally linked to Xero items
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

## Development Standards

- **Always work on a feature branch.** Never commit directly to `main` (backend) or `master` (frontend). Branch from `main`/`master` using Gitflow naming: `feature/FeatureName`, `fix/BugDescription`, `chore/TaskName`. Example: `feature/CalendarIntegration`.
- **Never merge to the base branch without explicit permission.** Open a PR and wait. Do not merge until the user has tested locally and either says "merge" or explicitly authorises it. When a PR is ready, ask: "Ready to merge?" and wait for confirmation.
- **Always update the releases list when shipping.** Every time a feature or fix is merged, add it to `projects-frontend/src/components/Settings/ReleasesPanel.tsx` — prepend new items to the matching date entry (or create a new entry if the date is new). Write entries in plain English from the user's perspective. Commit this in the same PR as the feature.
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
