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
npm run dev             # Vite dev server at http://localhost:5173
npm run build           # TypeScript check + Vite build
npm run preview         # Preview production build

npm run test            # Full suite: typecheck + prettier + lint + vitest + build
npm run vitest          # Unit tests only
npm run vitest:watch    # Watch mode
npm run typecheck       # tsc --noEmit
npm run lint            # ESLint + StyleLint
npm run prettier:write  # Auto-fix formatting

npm run storybook       # Component explorer at port 6006
```

## Architecture

### Backend (`src/`)

- **`index.ts`**: Express app entry. Configures CORS, session cookies, Passport Google OAuth, mounts all routes. Contains `allowedEmails` hardcoded allowlist.
- **`routes/`**: One file per domain — `projectRoutes`, `contactRoutes`, `taskRoutes`, `noteRoutes`, `designFileRoutes`, `gmailRoutes`, `xeroRoutes`, `authRoutes`, `settingsRoutes`, `userRoutes`, `projectContactRoutes`, `portalRoutes`.
- **`routes/xeroClient.ts`**: Shared Xero token handling used across routes.
- **`routes/portalRoutes.ts`**: Public token-gated portal routes (no auth middleware) — validate token, PDF proxy, request-mfa, verify-mfa, approve, feedback CRUD, reissue. Mounted at `/portal` (no `/api` prefix).
- **`routes/projectContactRoutes.ts`**: CRUD for per-project contacts (`/api/projects/:id/contacts`).
- **`utils/ensureAuthenticated.ts`**: Auth middleware applied to all protected routes.
- **`utils/ensureAdmin.ts`**: Admin-only middleware — returns 403 if `req.user.isAdmin` is falsy.
- **`utils/sendEmail.ts`**: Gmail API email sending via `studio@vil.nz` shared inbox. RFC 2047 subject encoding, base64 MIME body, auto-labels sent messages with `VisualOS/JOB-{projectId}` (MFA code emails excluded). Contains 4 HTML templates: `approvalRequestEmailHtml`, `newTokenEmailHtml`, `mfaCodeEmailHtml`, `approvalConfirmedEmailHtml`.
- **`utils/portalAudit.ts`**: `logPortalEvent()` — writes structured events to `PortalAuditLog` (token_accessed, mfa_sent, mfa_verified, approval_submitted, etc). Failures are non-fatal.
- **`prisma/schema.prisma`**: Source of truth for data models.

Authentication is session-based (express-session + **connect-pg-simple** PostgreSQL session store — sessions persist across backend rebuilds/restarts). Google OAuth tokens are stored on the `User` model for Gmail/Drive access. Xero has a separate OAuth flow stored on the same `User` model.

**Drive token endpoint:** `GET /auth/drive-token` exchanges the stored `google_refresh_token` for a short-lived access token using the googleapis `OAuth2Client`. Frontend calls this via `getDriveAccessToken()` utility (`src/utils/driveToken.ts`) before opening the Google Picker — silently refreshes without user interaction.

The Xero webhook endpoint (`POST /api/webhooks/xero`) uses HMAC-SHA256 verification — no session auth.

### Frontend (`src/`)

- **`main.tsx` → `App.tsx` → `Shell.page.tsx`**: App entry chain. `App.tsx` splits into two top-level routes: `/portal/:token` (no AppShell, no auth — uses `Portal.page.tsx`) and `/*` (full app inside `ShellPage`). `Shell.page.tsx` owns the `AppShell` layout, navigation, and all authenticated routes.
- **`pages/`**: Full-page route components. `Dashboard.page.tsx` is the Kanban board; `Project.tsx` is the multi-tab project detail view; `Portal.page.tsx` is the public client portal.
- **`components/`**: Reusable UI — `Tasks/TaskList`, `Tasks/TaskModal`, `Project/Tabs/DesignTab`, `Project/Tabs/EmailTab`, `PhoneMessageModal`, `DrivePickerButton`, `Project/ProjectContactsPanel` (add/edit/delete per-project contacts).
- **`components/Portal/`**: Portal-specific components — `PortalLayout`, `DesignViewer` (PDF iframe proxy), `ApprovalActions` (Approve/Request Changes buttons + T&Cs checkbox), `MFAConfirmationModal` (6-digit PinInput), `FeedbackEntryList` (editable or read-only), `FeedbackEntryForm`, `TokenExpiredScreen`.
- **`types/`**: Shared TypeScript interfaces (`IProject`, `ITask`, `IDesignFile`, `Contact`, `SystemSettings`, `IProjectContact`, `IDesignApproval`, `IDesignApprovalToken`, `IFeedbackItem`).
- **`utils/notifications.ts`**: Wrapper around Mantine notifications for toast messages.
- **`utils/driveToken.ts`**: `getDriveAccessToken()` — calls `GET /auth/drive-token` to exchange the stored refresh token for a short-lived Drive access token.
- **`hooks/useAuthStatus.ts`**: Checks login state on app load.

State management is local `useState`/`useEffect` per component — no global state yet (planned via React Context). HTTP calls use Axios with `withCredentials: true` for cookie auth.

### Data Models (Prisma)

Key models: `User`, `Project`, `Contact`, `Task`, `Note`, `DesignFile`, `SystemSettings`, `ProjectContact`, `DesignApprovalToken`, `DesignApproval`, `PortalAuditLog`.
- `User` has `isAdmin Boolean @default(false)` — `bren@vil.nz` and `bev@vil.nz` are seeded as admins via migration
- `Project` has a FK to `Contact` (Xero contact), plus optional `driveFolderId`/`driveFolderName`
- `Project.status` valid values: `New`, `Design`, `Quote`, `With Customer`, `Production`, `Install`, `Invoice`, `Archived` — Kanban shows all except `New`
- `Task` can be standalone or linked to `Project` or `DesignFile`; phone message tasks set `isPhoneMessage=true` and carry `callerName`, `callerPhone`, `callerEmail`, `takenAt`; client feedback tasks have `source='client_feedback'`, `portalTokenId`, and start as `status='draft'` until the client submits (then promoted to `pending`); all task list queries exclude `draft` status
- `Contact` has sub-relations: `ContactAddress`, `ContactPhone`, `ContactPerson`; `driveFolderId`/`driveFolderName` store the linked Brand Assets Drive folder
- `DesignFile` tracks Google Drive files with version history and approval status (`pending_review`, `changes_requested`, `approved`); `approvalSentAt` and `sentByUserId` set when approval email is sent
- `SystemSettings` — single-row singleton (`id=1`), stores `baseDriveFolderId`/`baseDriveFolderName` (top-level job folder) and `termsUrl` (T&Cs link shown in client portal approval flow)
- `ProjectContact` — per-project contacts (name, email, isPrimary); used to send design approval emails
- `DesignApprovalToken` — 96-char hex token (14-day expiry) sent to each contact for portal access; one per contact per design file send
- `DesignApproval` — MFA-verified approval record per token; stores 6-digit code, expiry, verified status, and final `status` (`approved`/`changes_requested`)
- `PortalAuditLog` — structured event log for all portal activity (token_accessed, mfa_sent, mfa_verified, approval_submitted, etc) with IP and user agent
- Xero sync only fetches `INPROGRESS` and `CLOSED` — Xero Projects API does not accept `DRAFT` as a states filter

### External Integrations

| Integration | Purpose | Auth |
|---|---|---|
| Google OAuth | Login + Gmail + Drive scopes | Passport `passport-google-oauth20` |
| Xero API | Projects, contacts, invoices | `xero-node` SDK, tokens on User |
| Gmail API | Thread viewer in project Email tab | googleapis, shared `studio@vil.nz` account |
| Google Drive | Design file uploads + file picker | googleapis |
| Google Picker API | Drive folder picker in project Details tab, Brand Assets tab, Settings page | Backend `/auth/drive-token` exchanges stored refresh token; `VITE_GOOGLE_API_KEY` (browser key) required |
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
- **Write tests for every new function/route.** Backend: add a test in a `*.test.ts` file alongside the route. Frontend: add a Vitest unit test for any new utility or hook.

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
- **Express API** on port 3001
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
