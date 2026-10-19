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
- Roles & permissions (RBAC): admin-managed roles, each holding a set of permission keys from `src/permissions/registry.ts`. Tabs carry `.view`/`.edit` pairs; pages and sensitive actions are single keys. Seeded with Admin (implicit full access, present and future) and Production Staff. Enforced server-side by `requirePermission()`, mirrored in the UI by `usePermissions()`/`<Can>`.
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
- Survey photo annotations: tap-to-draw dimension lines on survey photos; stored as relative JSONB; DPR canvas; pinch-to-zoom; overlay controls (no layout shift)
- Tasks: task type badge shown in all task list views (My Tasks, project lists, Shop Floor); project picker shown when editing tasks on My Tasks page
- Vinyl Calculator: per-project tab for laying out vinyl pieces on a roll; skyline packing with auto-rotation; live layout preview, efficiency %, print sizes with bleed; calculations saved per project
- EFTPOS: admin Settings tab shows Verifone transaction history by date range; manual sync button
- Materials pricing: cost price and sale price on each material (synced from Xero or set manually); both columns shown in the Materials catalogue table and editable in the create/edit modal
- Staff cost price: optional hourly cost rate per staff member (admin only); used in Financial tab cost calculations
- Charge out rates: admin-managed billing rates (Standard $108/hr, Web $128/hr); star marks default; used to calculate staff revenue on Financial tab
- Financial tab (admin only): per-project admin tab showing staff time cost/revenue, materials cost/revenue, other expenses, discount, total to invoice, manual invoice override, and profit (teal/red); per-row Est/Act and Chargeable toggles (persisted per row — survive reload); multi-staff rows per task
- Xero quote & invoice creation from the Financial tab: draft quote/invoice with configurable line structure (combine all or itemise per labour/materials/expenses); link an existing Xero quote/invoice by URL; pre-tax totals with refresh; due date from contact payment terms; PO/Ref passed to Xero
- Financial Overview (admin only, `/financial-overview`): monthly invoiced contribution vs budgeted overheads — Coverage + Pace gauges, per-invoice drill-down, and a gaps panel (invoices missing totals; expectsInvoice-stage projects with no invoice, filtered by a configurable start date). Contribution = invoice total − pro-rated material/expense cost (labour left in, since wages are a Budget line)
- Live timesheet timer on the project Timesheet tab (start/stop, running badge, live duration); hours-or-range entry mode; on-the-fly task creation in the entry modal
- Stage-gated task visibility: task types can be hidden until a project reaches a configured stage
- Mileage tracking: per-trip logging (standalone/project/task-linked), odometer or direct km, vehicles managed in Settings, project Mileage tab, combined My VisualOS page
- Public enquiry form at `/enquire` (creates contact + project + follow-up task; work types in Lists; honeypot spam protection)
- Overview page + Home per-stage stat blocks; project status history log on Details; deleting a project closes it in Xero
- Two-way SMS (live): sms-gate.app gateway on the NAS, threads keyed on phone number, two-day reply attribution, `/messages` inbox, project Messages tab, live sticky toasts over SSE

---

## Backlog Items

### 11. Global App State with React Context
**Status:** 🚧 In progress — foundation built (with #42)
**Priority:** Medium
**Labels:** `enhancement`, `architecture`, `ux`

Currently some state (e.g. task count badge in sidebar) is fetched independently per component with polling. A shared React context would allow components to reactively share state without redundant API calls or polling.

**Foundation (shipped with #42 SMS):**
- `GET /api/events` — Server-Sent Events stream fed by an in-process event bus (`src/realtime/eventBus.ts`). Generic `{ type, payload }` envelope; each event carries an `audience` of permission keys and is filtered per user.
- `RealtimeProvider` (`contexts/RealtimeContext.tsx`) owns the app's single `EventSource`; components subscribe with `useRealtimeEvent(type, handler)`.
- `MessagingContext` is the first slice on top of it (SMS enabled flag + per-user unread badge). `PermissionsContext` (user) and `TaskContext` (task count) already exist — the remaining work is publishing task events so the task badge updates live too.

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

### 26. Client Portal — Read-Only Job History (Phase 1)
**Status:** 📋 Planned — reframed October 2026
**Priority:** Low
**Labels:** `feature`, `customer-facing`, `magic-link`, `portal`

**Reframe:** the original idea was a full interactive portal (all jobs, install dates, photos, self-serve change requests). Many customers already bypass the existing approve button and just reply "approved" by email, so a big interactive portal risks solving a problem they don't feel. The real kernel of value is **"show me all my stuff"**. Start read-only and only build interaction if customers actually use it.

**Phase 1 — read-only history view:**
- One persistent magic link per customer (tied to the `Contact`), same no-password token pattern as design approval
- Lists the customer's projects (active + completed) with current stage, using the stage `label` from taxonomy
- Scheduled install date where one exists on the calendar
- Approved design (PDF via the existing portal proxy) and completion photos once installed
- Branded from the Brand Assets tab where available
- `PortalAuditLog` events for link access, so we can see whether anyone opens it

**Deferred (Phase 2, only if Phase 1 gets used):**
- Self-serve change requests that create a Task
- Anything post-install: that's a remediation / warranty job, not a change request, and shouldn't be on a self-serve button at all

**Notes:**
- Decide before building: who gets the link (primary contact only?), when it's sent (with the job notification email?), and token expiry vs rolling refresh
- Related: #47 (lock portal change requests from Production)

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

### 29. Customer Contact Opt-Outs ("Don't contact via")
**Status:** 📋 Planned — reframed October 2026 (replaces "Client Communication Preferences")
**Priority:** Medium
**Labels:** `feature`, `contacts`, `notifications`, `sms`

**Reframe:** a single "preferred channel" forces customers to pick one when most are happy with any. What we actually need is the exceptions: "don't text me", "don't email me", "don't call me". An opt-out list is also simpler — no preference enum, no fallback logic.

**Model:**
- `doNotContactVia String[] @default([])` — values `email` | `sms` | `phone`
- On `Contact` (organisation level) and `ProjectContact` (person level). Effective opt-outs for a person = union of both.

**Behaviour:**
- Pure util `canContactVia(contact, projectContact, channel)` — single place every send path asks
- Job notification emails (`resolveNotificationRecipient`) and design approval sends skip opted-out recipients and say so in the toast
- SMS sends refuse opted-out numbers; texting STOP (already sets `SmsThread.optedOut`) also adds `sms` to the linked contact's list, so there's one source of truth
- Notify Customer / Text customer buttons disable with a tooltip explaining why
- "Don't call" shown as a badge on the contact, project contacts panel, and phone-message modal
- Multi-select editor on the contact detail page and project contacts panel

**Tests:**
- Backend: `canContactVia` (all combinations), notification recipient resolution with opt-outs, STOP syncing to the contact
- Frontend: disabled button states + tooltip text

**Acceptance Criteria:**
- [ ] `doNotContactVia` on `Contact` + `ProjectContact` + migration
- [ ] `canContactVia` util used by email, approval, and SMS send paths
- [ ] STOP → contact opt-out sync
- [ ] Editor + badges in the UI
- [ ] `ReleasesPanel.tsx` updated

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

### 38. Standard Terms & Conditions on Xero Quotes
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `xero-api`, `financial`, `settings`

Quotes created from the Financial tab (`POST /api/projects/:id/quote` in `invoiceRoutes.ts`) go out without our standard terms & conditions. The T&Cs that appear on quotes created in the Xero web app are a default applied by the Xero UI — they are **not** applied when a quote is created via the API, and Xero's API does **not** expose the organisation's default quote/invoice terms text (the `BrandingThemes` and `Organisation` endpoints don't return it). So we can't pull them from Xero.

**Approach:** store the T&Cs in VisualOS and send them on every quote.

- Add `defaultQuoteTerms String?` to `SystemSettings` (alongside `termsUrl` / `defaultInvoiceDays`).
- Surface it as a textarea in Settings → Admin → Invoicing (admin only) — paste the standard T&Cs once.
- In the quote payload, set `terms` from the setting: `...(settings.defaultQuoteTerms && { terms: settings.defaultQuoteTerms })`. (Xero `Quote.terms`, max 4000 chars.)

**Acceptance Criteria:**
- [ ] `defaultQuoteTerms` field on `SystemSettings` + migration
- [ ] Editable textarea in Settings → Admin → Invoicing
- [ ] `terms` populated on every created Xero quote when the setting is non-empty
- [ ] `ReleasesPanel.tsx` updated

**Notes:** consider applying the same to invoices later. Multi-tenant: this becomes per-org terms under #32.

---

### 40. Remove `User.isAdmin`
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `cleanup`, `migration`, `rbac`

Cleanup left behind by roles & permissions (#39). `User.isAdmin` is superseded by `User.roleId` — nothing reads it any more, and role assignment mirrors it purely so an unmigrated reader can't go stale. Drop the column and the mirroring.

**Acceptance Criteria:**
- [ ] `isAdmin` removed from `schema.prisma` + migration
- [ ] `isAdmin` mirroring removed from `PATCH /api/users/:id/role` in `roleRoutes.ts`
- [ ] `isAdmin` removed from the frontend `User` type
- [ ] No remaining `isAdmin` references (`grep -r isAdmin`)
- [ ] `ReleasesPanel.tsx` updated

---

### 41. Add `setAppId` to the Google Drive Pickers
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `cleanup`, `google-drive`, `oauth`, `tech-debt`

Neither picker component calls `PickerBuilder.setAppId()`, which Google's web-picker guide lists as required — it's the GCP **project number**, and it's what associates a picked file with our app under the `drive.file` scope.

It works today only because we also request the **restricted** `drive.readonly` scope (`projects-backend/src/index.ts:148`), which covers reads regardless of app association. That's leaning on a restricted scope to paper over a missing `setAppId`. Google has been steadily pushing apps toward `drive.file`-only; if `drive.readonly` is ever tightened or trimmed from our consent screen, both pickers break.

**Context:** raised off the back of the September 2026 Google Picker security email (enforcement November 16, 2026). That enforcement itself **does not apply to us** — `prompt=consent` + `trigger_onepick=true` are for the OnePick flow used by *native desktop/mobile* apps. VisualOS is a plain browser SPA (no Capacitor/Cordova/Electron/Tauri, no PWA manifest, no embedded webview) using the supported `gapi.load('picker')` + `PickerBuilder().setOAuthToken()` web path, and our OAuth already sends `prompt=consent` (`index.ts:163`). No deadline work required — this item is the optional hardening only.

**Steps:**
1. Get the GCP project **number** (Cloud Console → project settings — numeric, not the project ID)
2. Expose it to the frontend as `VITE_GOOGLE_APP_ID` (`.env.development.local`, `.env.production`, GH Actions secret on the frontend repo)
3. Add `.setAppId(import.meta.env.VITE_GOOGLE_APP_ID)` to the `PickerBuilder` chain in `DrivePickerButton.tsx` and `DriveFilePickerButton.tsx`
4. Verify folder linking, design file picking, and Drive uploads still work
5. Once confirmed, consider whether `drive.readonly` can be dropped from `googleScopes` (`index.ts:148`) and `STUDIO_SCOPES` (`authRoutes.ts:89`) — dropping a restricted scope simplifies OAuth verification. Re-consent required if so.

**Acceptance Criteria:**
- [ ] `VITE_GOOGLE_APP_ID` wired through dev, prod, and CI
- [ ] `setAppId` on both picker components
- [ ] Folder picker, file picker, and Drive upload verified end to end
- [ ] `ReleasesPanel.tsx` updated

**Notes:** step 5 is a separate decision — don't drop `drive.readonly` in the same PR as `setAppId`.

---

### 42. Two-Way SMS Messaging + Live Notifications
**Status:** ✅ Live — October 2026 (gateway running, `smsEnabled` on, messages sent to customers)
**Priority:** Medium
**Labels:** `feature`, `sms`, `notifications`, `customer-facing`

Full spec: `sms-messaging-spec.md`. sms-gate.app gateway (Android phone + private server on the NAS), HMAC-signed webhook, threads keyed on E.164 number, two-day reply attribution, per-user read state vs team-wide "actioned", SMS inbox at `/messages`, project Messages tab, SSE live toasts (stream at `/api/realtime`). Builds the #11 foundation.

**Follow-ups:**
- "Seen by" avatars from `SmsThreadRead`
- A way to clear `optedOut` if a customer texts START (ties into #29)

---

### 43. Website Chatbot + Inbox FAQ Miner
**Status:** 🚧 In progress. The knowledge pack and the FAQ miner are done (see `complete.md`), and 47 drafted entries are loaded in prod. The website chatbot itself is still to do. See `spec-43-website-chatbot-faq-miner.md`
**Priority:** Medium
**Labels:** `feature`, `ai`, `customer-facing`, `intake`, `gmail-api`, `website`

A customer-facing chatbot on the new website (beta.vil.nz) that answers questions about what we do, grounded in a knowledge pack we control, and hands qualified enquiries into Kit as projects. The knowledge pack is kept current by a weekly job that mines the `studio@vil.nz` inbox for real customer questions.

**✅ Done (Oct 2026)**
- ✅ Knowledge pack: Settings → Chatbot → Knowledge, with categories in Lists (backend #171 / frontend #223)
- ✅ FAQ miner: backfill + weekly scan of studio@, pre-filters, PII strip, Haiku via the Batch API, Suggestions review queue (backend #171 / frontend #223)
- ✅ Permission keys: `settings.tab.chatbot.view/.edit`, `action.faq-miner.run`
- ✅ Unit tests for the PII stripper, mail pre-filter and dedupe
- ✅ First knowledge pack loaded: 47 entries from `knowledge-pack-draft.json` via the one-off import (backend #172). Two are unpublished pending a decision (see `_review.needsDecision` in the draft)

**To do: miner follow-ups**
- [ ] Remove the one-off `POST /api/knowledge/admin/import` endpoint now that it has run in prod
- [ ] Create the Anthropic API key, set a monthly spend limit, add `ANTHROPIC_API_KEY` to the NAS `.env`. The miner stays switched off until this is done
- [ ] Run the first backfill, review the suggestions, then switch on the weekly scan

**To do: website chatbot**
- Public `POST /api/chat` route on Kit — origin-locked and rate-limited like `/api/quote-requests`; API key stays server-side; max turns per chat, max output tokens, monthly spend limit in the Anthropic console
- Claude Haiku with the full knowledge pack in a cached system prompt (no RAG/vector DB while it stays small)
- Guardrails: answer only from the knowledge pack or tool results; say "not sure" and offer handoff otherwise; never commit to prices, dates, or warranty outcomes
- Tools: `submit_enquiry` (structured fields → existing quote-request intake: contact match → project → task, transcript attached); `get_portfolio_examples` optional
- Small JS embed widget for the website
- Transcripts logged in Kit; questions the bot couldn't answer feed the same Suggested FAQs queue
- Chat settings, including a configurable `chatbotName` (default **Kit**)
- Tests: enquiry payload mapping (Vitest). Eval suite: 30–50 real (anonymised) customer questions with expected answers + 15–20 known emails for the classifier; re-run on any knowledge or prompt change

**Notes:**
- Expected running cost is a few dollars a month at our volume (Haiku pricing)
- Knowledge pack could share a source with website blog content
- Out of scope for now: quoting estimator / price guide tool — separate idea, to be backlogged later
- Multi-tenant: knowledge pack becomes per-org under #32


### 44. Stop Mounting `mileageRoutes` at Bare `/api`
**Status:** 📋 Planned
**Priority:** Low
**Labels:** `tech-debt`, `routing`, `backend`

`mileageRoutes` is mounted twice: at `/api/mileage` (correct) and at bare `/api`, only so `/api/projects/:projectId/mileage` works. The bare mount also exposes its catch-all paths across the whole `/api` namespace: `GET /api/`, `POST /api/`, `GET /api/all`, and **`PATCH`/`DELETE /api/:id`**. Any single-segment PATCH or DELETE under `/api` that a router mounted later would serve gets swallowed by mileage instead. It's the same class of bug as the `/api/events` SSE collision (fixed by moving the stream to `/api/realtime`). Nothing is broken today, but it's a trap for the next new route.

**Fix:** move `GET /projects/:projectId/mileage` into its own router (or into `projectRoutes`) mounted at `/api/projects`, and drop `app.use('/api', mileageRoutes)` from `routes/mountRoutes.ts`.

**Acceptance Criteria:**
- [ ] `app.use('/api', mileageRoutes)` removed from `mountRoutes.ts`; the header list of bare-`/api` routers updated
- [ ] `/api/projects/:projectId/mileage` still works (project Mileage tab)
- [ ] `/api/mileage/*` unchanged
- [ ] `mountRoutes.test.ts` asserts `GET /api/all` and `PATCH /api/<anything>` no longer reach mileage, and the project mileage path still does
- [ ] `CLAUDE.md` bare-`/api` list updated
- [ ] `ReleasesPanel.tsx` updated

---

### 45. Live Invoice Status from the Xero Webhook
**Status:** ✅ Shipped — October 19, 2026 (see `complete.md`)
**Priority:** Medium
**Labels:** `xero`, `webhook`, `financial`, `realtime`

The Xero webhook received INVOICE events but only logged them, so invoice status only changed when someone pressed Sync Invoices. Handle INVOICE events: update every linked project's invoice from Xero and refresh open pages live.

**Acceptance Criteria:**
- [x] INVOICE events update status, number, totals and date on every project the invoice is linked to
- [x] Unlinked invoices and bills skipped without a Xero API call
- [x] One shared update builder for the webhook, Sync Invoices and the nightly sync
- [x] `invoice.updated` realtime event; Home, Financial tab and Financial Overview refresh without a reload
- [x] Constant-time webhook signature check
- [x] Nightly backstop sync for events the webhook missed
- [x] Tests: linked, unlinked, missing / 404, other Xero errors, multi-link, sweep, route-level routing
- [x] `ReleasesPanel.tsx` updated

---

### 46. Website FAQ Page Fed from Kit's Knowledge Pack
**Status:** 📋 Specced — see `spec-46-website-faq-from-kit.md`
**Priority:** High
**Labels:** `feature`, `website`, `wordpress`, `api`, `webhook`, `faq`

Deliberately separate from #43 so it can ship now, with no AI or API credits needed. The knowledge pack (`KnowledgeEntry`, shipped under #43) is the single source of truth; the website (beta.vil.nz, WordPress) shows a plain FAQ page from the same published entries.

- Public, read-only `GET /api/public/knowledge` on the existing public router (published entries grouped by category, sanitised HTML, ETag)
- Every knowledge write sends a debounced, HMAC-signed "knowledge updated" ping to WordPress
- Small `kit-faq` WordPress plugin verifies the ping, re-fetches from Kit and caches locally — the page always renders from the cache, so it's fast and survives Kit being down
- `[kit_faq]` shortcode renders the accordion + FAQPage structured data; daily fallback re-fetch and a "Resync now" button
- "Website updated …" status line in Settings → Chatbot → Knowledge

---

### 47. Lock Portal Change Requests Once a Job Hits Production
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `portal`, `design-approval`, `taxonomy`

Once a job is in Production, customers shouldn't be able to self-serve a change request on the approval page. Post-install issues are a remediation job, not a change request.

**Approach — blunt cutoff, tied to a stage flag:**
- New `project_stage` meta flag `locksPortalChanges` (toggle in Settings → Lists). Seed it on Production; it applies at that stage **and every later stage** (by stage `sortOrder`, same comparison as stage-gated tasks)
- When locked, the portal hides **Request changes** and shows "Spotted an issue? Give us a call on …" (studio phone from settings)
- Enforce server-side: feedback POST/PATCH and changes-requested submissions return 409 when the project is locked — not just hidden in the UI
- Approving stays allowed (late approvals still need recording)
- No "printed yet" sub-status for now; add a finer-grained flag later only if pre-print tweaks during Production turn out to be common

**Tests:**
- Backend: pure `isPortalLocked(stage, stages)` util (before / at / after the flagged stage, archived stages, no flag set); route returns 409 when locked
- Frontend: `ApprovalActions` renders the call-us message instead of Request changes when locked

**Acceptance Criteria:**
- [ ] `locksPortalChanges` flag in taxonomy meta + Lists toggle
- [ ] Portal response includes `changesLocked`; UI swaps the button
- [ ] Server-side 409 on locked feedback/change requests
- [ ] `ReleasesPanel.tsx` updated

---

### 48. Action-Triggered Stage Prompts
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `workflow`, `ux`, `taxonomy`

When staff do something that implies a stage change, Kit should notice and offer to move the job on — rather than relying on someone remembering to drag the Kanban card. Example: create an invoice on the Financial tab → "Invoice created — move this project to Completed?" **Always a prompt, never automatic.**

**Triggers (code-defined keys):**
- `invoice.created` — Xero invoice created or linked from the Financial tab
- `quote.created` — Xero quote created or linked
- `notification.email.sent` — Notify Customer / stage notification email sent
- `sms.sent` — text sent from the project Messages tab
- `design.approved` — design approved (portal or internal)

**Rules (data, not code):**
- New `StageTransitionRule` model: `trigger`, `fromStage String?` (null = any stage), `toStage`, `isActive`, `sortOrder`. Takes `organisationId` under #32
- Edited in Settings → Lists (new "Stage prompts" section), gated by the Lists tab's edit key
- Seed: `invoice.created` → Completed
- Pure util `suggestTransition(rules, trigger, currentStage, stages)` — returns the target stage or null. Never suggests the current stage or moving backwards

**Flow:**
- The action's API response includes `suggestedStage` (computed server-side with the util)
- Frontend `StageSuggestionModal`: "Move to **Completed**?" — **Move** / **Not now**. Shows the consequences of the move (e.g. "closes the Xero project", "emails the customer") using the same flags as the contextual status toast
- **Move** uses the existing `PATCH /api/projects/:id`, so notifications, `closesXero`, and the status history log all fire as normal
- Respect the existing stage-change permission — no prompt if the user can't change stages

**Open question:** invoices are created in Xero as drafts. Prompt on create for v1. Since #45 now tracks live invoice status from the Xero webhook, a later `invoice.sent` trigger (status → `AUTHORISED`) could prompt at the moment it's actually sent instead.

**Tests:**
- Backend: `suggestTransition` (any-stage rules, from-stage match, inactive rules, backwards moves, already at target); routes include `suggestedStage`
- Frontend: modal renders consequences, Move calls PATCH, Not now closes without a request

**Acceptance Criteria:**
- [ ] `StageTransitionRule` model + migration + seed
- [ ] Rules editor in Settings → Lists
- [ ] `suggestedStage` returned by invoice, quote, notify, SMS, and approval actions
- [ ] `StageSuggestionModal` wired into each action
- [ ] `ReleasesPanel.tsx` updated

---

## Priority Summary

### High Priority (Ready to Build)
1. Website FAQ from Kit's Knowledge Pack (#46) — no AI credits needed

### Medium Priority
1. Action-Triggered Stage Prompts (#48)
2. Lock Portal Change Requests from Production (#47)
3. Customer Contact Opt-Outs (#29)
4. Website Chatbot + Inbox FAQ Miner (#43): knowledge pack + miner done, chatbot + website widget to do (waiting on API credits)
5. Global App State with React Context (#11)
6. Field View — Install & Survey Map (#24)
7. Delivery & Install Docket — Mobile Field View (#25)
8. Project Templates (#27)
9. Project Scaffolding / Decision Tree Wizard (#31)
10. Multi-Tenancy / Organisation Layer (#32)
11. Admin Integration Settings Page (#33)
12. Standard T&Cs on Xero Quotes (#38)

### Low Priority / Cleanup
1. Client Portal — Read-Only Job History (#26) — reframed; build only if wanted
2. Move Notes onto Project Details Tab (#37)
3. Remove `User.isAdmin` (#40) — cleanup migration after #39
4. Vitest Setup (Backend) (#12)
5. Remove `xeroContactName` from Project (#14) — cleanup migration
6. Business Intelligence Dashboard (#23)
7. Materials Catalog (#6) — PARKED
8. Transactional Email Provider (#34) — future
9. Email / Calendar Provider Abstraction (#35) — future
10. File Storage Provider Abstraction (#36) — future
11. Add `setAppId` to the Drive Pickers (#41) — tech debt
12. Stop mounting `mileageRoutes` at bare `/api` (#44) — tech debt, latent route-shadowing trap

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
- Every new pure util gets a unit test in the same PR — backend and frontend (Vitest)

---

**Last Updated:** October 20, 2026 — #42 SMS marked live (gateway health check and webhook logging already shipped). Reframed #29 as contact opt-outs and #26 as a read-only job history portal. Added #46 (website FAQ fed from the knowledge pack, specced in `spec-46-website-faq-from-kit.md`), #47 (lock portal change requests from Production) and #48 (action-triggered stage prompts).

October 20, 2026 — #43: knowledge pack + FAQ miner shipped, 47 drafted entries imported. Website chatbot still to do.

October 19, 2026 — Added #43 (website chatbot "Kit" + studio@ inbox FAQ miner feeding enquiries into Kit), specced in `spec-43-website-chatbot-faq-miner.md`. Quoting estimator deliberately left out for now.

September 21, 2026 — Added #41 (`setAppId` on the Drive pickers), raised off the Google Picker security email. Confirmed the November 16, 2026 Picker enforcement does not apply to VisualOS (native desktop/mobile apps only).

June 24, 2026 — Shipped Xero quote/invoice creation, other expenses, live timesheet timer, multi-staff financial rows, stage-gated tasks, PO/Ref field, mileage tracking, enquiry form, overview page, status history. Added #38 (standard T&Cs on Xero quotes).
