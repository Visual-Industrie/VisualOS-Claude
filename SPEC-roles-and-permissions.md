# SPEC — Roles & Permissions (RBAC)

**Status:** Draft for build
**Backlog item:** #39
**Scope:** Backend (`projects-backend`) + Frontend (`projects-frontend`)
**Supersedes:** `User.isAdmin` boolean + `ensureAdmin` middleware

---

## 1. Goal

Replace the single `isAdmin` boolean with a proper role-based access control layer:

- Admin-managed **Roles**, each holding a set of **permission keys**.
- Permissions are granular: one key per page, per tab, and per sensitive action.
- Ship with two seeded roles — **Admin** and **Production Staff**.
- Bren and Bev get Admin (they are the current `isAdmin` users).
- New roles can be created, edited, and deleted from Settings.
- **Every new major feature adds a permission key** so it can be switched on/off per role.

Backend is the authority. Frontend hiding is UX only — every gated route must also be enforced server-side.

---

## 2. Key design decisions

| Decision | Choice | Why |
|---|---|---|
| Where does the role live? | `roleId` on **`User`** (the auth principal), not `StaffMember` | Permission checks run against `req.user`. Staff without a login can't have permissions. Role is still *edited* from the Staff tab via the linked user account. |
| How are permissions stored? | `Role.permissions Json` — a flat `string[]` of keys | Matches the existing `TaxonomyItem.meta` pattern. Atomic update, no join-table churn when the registry grows. Registry (code) is the source of truth for what keys mean. |
| Does Admin get new permissions automatically? | Yes — `Role.isSystemAdmin = true` short-circuits every check to `true` | New features never silently lock Bren out. Admin's checkboxes render all-checked and disabled. |
| Default for a brand-new permission key on non-admin roles | **Off** | Fail closed. Granting is a deliberate act. |
| Where is the Roles UI? | New **Settings → Roles** tab (admin-gated), plus a **Role** column + picker on the Staff tab | Staff tab stays about people; Roles tab is about the permission matrix. |
| Unknown keys stored on a role | Ignored on read, stripped on write | Lets us delete a feature without a data migration. |

---

## 3. Permission registry

### 3.1 Source of truth

`projects-backend/src/permissions/registry.ts` exports:

```ts
export interface PermissionDef {
  key: string;          // stable, never renamed once shipped
  label: string;        // checkbox label in the Roles UI
  description?: string; // helper text under the checkbox
  group: PermissionGroup;
}

export type PermissionGroup =
  | 'pages'
  | 'project-tabs'
  | 'contact-tabs'
  | 'settings-tabs'
  | 'actions';

export const PERMISSION_GROUPS: { key: PermissionGroup; label: string; order: number }[] = [...];
export const PERMISSIONS: PermissionDef[] = [...];
export const PERMISSION_KEYS: string[] = PERMISSIONS.map((p) => p.key);
export const isValidPermission = (key: string) => PERMISSION_KEYS.includes(key);
```

Exposed to the frontend via `GET /api/permissions` so the Roles UI renders its checkboxes from the server — no duplicated list to keep in sync.

Naming convention: `<group>.<subject>[.<qualifier>]`, lowercase kebab within segments. **Keys are permanent.** If a feature is renamed, keep the key and change the `label`.

### 3.2 Group: Pages (`page.*`)

Controls sidebar visibility and route access.

| Key | Label | Notes |
|---|---|---|
| `page.home` | Projects list | Home route |
| `page.dashboard` | Kanban board | |
| `page.overview` | Overview | Stage stat tiles |
| `page.calendar` | Calendar | Master calendar |
| `page.contacts` | Contacts | List + detail |
| `page.materials` | Materials catalogue | |
| `page.my-tasks` | My Tasks | |
| `page.my-visualos` | My VisualOS | Own timesheet + mileage |
| `page.log-mileage` | Log Mileage | |
| `page.shop-floor` | Shop Floor | `/shopfloor` |
| `page.financial-overview` | Financial Overview | Currently admin-only |
| `page.settings` | Settings | Gate on the page itself; individual tabs gated separately |

Public routes (`/portal/:token`, `/enquire`) are exempt — no session, no permission check.

### 3.3 Group: Project tabs (`project.tab.*`)

| Key | Label |
|---|---|
| `project.tab.details` | Details |
| `project.tab.design` | Design |
| `project.tab.email` | Email |
| `project.tab.survey` | Survey |
| `project.tab.schedule` | Schedule |
| `project.tab.tasks` | Tasks |
| `project.tab.deliverables` | Deliverables |
| `project.tab.materials` | Materials |
| `project.tab.vinyl-calculator` | Vinyl Calculator |
| `project.tab.completion-photos` | Completion Photos |
| `project.tab.timesheets` | Timesheets |
| `project.tab.mileage` | Mileage |
| `project.tab.financial` | Financial |
| `project.tab.notes` | Notes | *(tab currently hidden — key reserved, see backlog #37)* |

### 3.4 Group: Contact tabs (`contact.tab.*`)

`contact.tab.overview`, `contact.tab.people`, `contact.tab.addresses`, `contact.tab.projects`, `contact.tab.brand-assets`.

### 3.5 Group: Settings tabs (`settings.tab.*`)

One per tab currently rendered in `Settings.tsx`:

`general`, `admin`, `templates`, `eftpos`, `staff`, `lists`, `vehicles`, `budget`, `timesheets`, `mileage`, `backlog`, `releases`, plus the new `roles`.

> `settings.tab.roles` is the lockout-sensitive one — see §7.3.

### 3.6 Group: Actions (`action.*`)

Sensitive capabilities that aren't a whole page or tab. These are the ones that must strip data server-side, not just hide UI.

| Key | Label | Enforcement |
|---|---|---|
| `action.project.create` | Create projects | `POST /api/projects` |
| `action.project.delete` | Delete projects | `DELETE /api/projects/:id` (also closes the Xero project) |
| `action.project.change-stage` | Change project stage | `PATCH /api/projects/:id` status field, Kanban drag |
| `action.contact.create` | Create contacts | `POST /api/contacts` |
| `action.contact.sync-xero` | Sync contacts from Xero | `POST /api/contacts/sync` |
| `action.material.sync-xero` | Sync materials from Xero | |
| `action.invoice.manage` | Create / link Xero invoices | all of `invoiceRoutes` invoice endpoints |
| `action.quote.manage` | Create / link Xero quotes | `POST /api/projects/:id/quote` |
| `action.timesheet.view-all` | View all staff timesheets | `GET /api/timesheets` (unfiltered) |
| `action.timesheet.edit-others` | Edit other staff's time entries | `PATCH`/`DELETE /api/timesheets/:id` where the entry isn't theirs |
| `action.mileage.view-all` | View all staff mileage | |
| `action.staff.manage` | Add / edit / delete staff | `POST`/`PATCH`/`DELETE /api/staff` |
| `action.staff.view-cost-rates` | View staff cost rates | **Strips `costPricePerHour` from `GET /api/staff` responses when absent** |
| `action.roles.manage` | Create / edit / delete roles | `roleRoutes` writes |
| `action.design.send-approval` | Send designs for client approval | `POST .../send-approval` |
| `action.email.send` | Send emails to customers | Notify Customer button, job notification emails |
| `action.settings.write` | Change system settings | `PATCH /api/settings/*` |

### 3.7 Adding a permission later

Add the entry to `PERMISSIONS` in `registry.ts` and use it. Nothing else. It appears in the Roles UI automatically, defaults to off for every non-admin role, and is implicitly granted to `isSystemAdmin` roles.

**Add to `CLAUDE.md` → Development Standards:**

> **Every new page, tab, or sensitive action needs a permission key.** Add it to `src/permissions/registry.ts`, gate the route with `requirePermission()`, and gate the UI with `can()` / `<Can>`. New keys default to off for non-admin roles.

---

## 4. Data model

```prisma
model Role {
  id            Int      @id @default(autoincrement())
  name          String   @unique
  description   String?
  // true = implicit grant of every permission, present and future.
  isSystemAdmin Boolean  @default(false)
  // true = cannot be deleted or renamed (the seeded Admin + Production Staff).
  isProtected   Boolean  @default(false)
  permissions   Json     @default("[]") // string[] of permission keys
  sortOrder     Int      @default(0)

  users User[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

`User` gains:

```prisma
  roleId Int?
  role   Role? @relation(fields: [roleId], references: [id], onDelete: SetNull)
```

`SystemSettings` gains:

```prisma
  // Role assigned to a user on first successful login. Null = no role (no access).
  defaultRoleId Int?
```

`User.isAdmin` **stays for now** (see §8 rollout) and is removed in a follow-up cleanup.

> Multi-tenancy (#32): `Role` takes an `organisationId` FK when that lands — roles are per-org.

### 4.1 Migration + seed

The `session` table causes `prisma migrate dev` drift (see CLAUDE.md). Write the migration SQL manually → apply via `psql` → `npx prisma migrate resolve --applied <name>`.

Seed inside the same migration:

1. **Admin** — `isSystemAdmin: true`, `isProtected: true`, `permissions: []` (empty is fine; the flag grants everything), `sortOrder: 0`.
2. **Production Staff** — `isProtected: true`, `sortOrder: 1`, permissions per §5.
3. `UPDATE "User" SET "roleId" = <admin.id> WHERE "isAdmin" = true;`
4. `UPDATE "User" SET "roleId" = <production.id> WHERE "roleId" IS NULL;`
5. `UPDATE "SystemSettings" SET "defaultRoleId" = <production.id> WHERE id = 1;`

---

## 5. Seeded role: Production Staff

Starting set — tune in the UI afterwards, no code change needed.

**Pages:** `page.home`, `page.dashboard`, `page.calendar`, `page.my-tasks`, `page.my-visualos`, `page.log-mileage`, `page.shop-floor`, `page.settings`

**Project tabs:** `details`, `design`, `survey`, `schedule`, `tasks`, `deliverables`, `materials`, `vinyl-calculator`, `completion-photos`, `mileage`

**Contact tabs:** `overview`, `people`, `addresses`, `projects`

**Settings tabs:** `backlog`, `releases`

**Actions:** `action.project.change-stage`

Deliberately excluded: everything financial (`project.tab.financial`, `project.tab.timesheets`, `page.financial-overview`, `action.invoice.manage`, `action.quote.manage`, `action.staff.view-cost-rates`), all admin settings tabs, project deletion, contact/material Xero syncs, and customer-facing email sends.

---

## 6. Backend

### 6.1 Permission resolution

`src/utils/permissions.ts`:

```ts
export function rolePermissions(role: Role | null | undefined): string[] {
  if (!role) return [];
  if (role.isSystemAdmin) return PERMISSION_KEYS;
  const raw = Array.isArray(role.permissions) ? (role.permissions as string[]) : [];
  return raw.filter(isValidPermission);
}

export function hasPermission(user: SessionUser | undefined, key: string): boolean {
  if (!user) return false;
  if (user.role?.isSystemAdmin) return true;
  return rolePermissions(user.role).includes(key);
}
```

Passport's `deserializeUser` must `include: { role: true }` so the role travels with `req.user` on every request. No extra query per permission check.

### 6.2 Middleware

`src/utils/requirePermission.ts`:

```ts
export const requirePermission =
  (key: string): RequestHandler =>
  (req, res, next) => {
    if (!req.user) return res.status(401).json({ error: 'unauthenticated' });
    if (!hasPermission(req.user as SessionUser, key)) {
      return res.status(403).json({ error: 'forbidden', requiredPermission: key });
    }
    return next();
  };
```

`ensureAdmin` becomes a deprecated shim during rollout:

```ts
/** @deprecated use requirePermission(<specific key>) */
export const ensureAdmin = requirePermission('settings.tab.admin');
```

### 6.3 Route mapping

Replace every current `ensureAdmin` usage:

| Route file | Endpoint | Permission |
|---|---|---|
| `financialOverviewRoutes` | `GET /api/financial-overview` | `page.financial-overview` |
| `financialOverviewRoutes` | `POST /api/admin/backfill-invoice-dates` | `settings.tab.admin` |
| `settingsRoutes` | `PATCH /api/settings/*` | `action.settings.write` |
| `staffRoutes` | `GET /api/staff` | *(any authed — but strip cost rates, see §6.4)* |
| `staffRoutes` | `GET /xero-users`, `POST`, `PATCH`, `DELETE` | `action.staff.manage` |
| `taxonomyRoutes` | `POST`/`PATCH`/`DELETE` | `settings.tab.lists` |
| `verifoneRoutes` | `POST /sync`, `GET /transactions` | `settings.tab.eftpos` |
| `timesheetRoutes` | `GET /` (all staff) | `action.timesheet.view-all` |
| `timesheetRoutes` | `PATCH /:id`, `DELETE /:id` on someone else's entry | `action.timesheet.edit-others` |
| `invoiceRoutes` | invoice create/link/unlink/refresh | `action.invoice.manage` |
| `invoiceRoutes` | quote create/link/unlink/refresh | `action.quote.manage` |
| `projectRoutes` | `DELETE /:id` | `action.project.delete` |
| `projectRoutes` | `POST /` | `action.project.create` |
| `contactRoutes` | `POST /sync` | `action.contact.sync-xero` |
| `materialRoutes` | Xero sync | `action.material.sync-xero` |
| `designFileRoutes` | `POST .../send-approval` | `action.design.send-approval` |
| budget / vehicles / charge-out-rate routes | all writes | `settings.tab.budget` / `settings.tab.vehicles` / `settings.tab.admin` |

Project-tab and contact-tab permissions gate the *data endpoints* behind them where a separate endpoint exists — e.g. the project Financial tab's data comes from routes that must require `project.tab.financial`. Hiding the tab alone is not enough.

### 6.4 Field-level stripping

`GET /api/staff` is used by task assignment pickers, so it stays open to all authed users — but `costPricePerHour` must be `undefined` in the response unless the caller has `action.staff.view-cost-rates`. Do this in a small `sanitiseStaff(staff, user)` helper, unit-tested.

Same principle for any endpoint that mixes sensitive and non-sensitive fields.

### 6.5 New routes — `src/routes/roleRoutes.ts`

| Method | Path | Permission | Behaviour |
|---|---|---|---|
| `GET` | `/api/permissions` | any authed | `{ groups, permissions }` from the registry |
| `GET` | `/api/roles` | `settings.tab.roles` | roles + `userCount` per role |
| `POST` | `/api/roles` | `action.roles.manage` | Zod: `name` (unique, 1–50), `description?`, `permissions: string[]` |
| `PATCH` | `/api/roles/:id` | `action.roles.manage` | partial; guards in §7.3 |
| `DELETE` | `/api/roles/:id` | `action.roles.manage` | 409 if `isProtected` or `userCount > 0` |
| `PATCH` | `/api/users/:id/role` | `action.roles.manage` | assign a role to a user; guards in §7.3 |

Invalid permission keys are **stripped** on write (not a 400) so a stale client can't wipe unknown keys, and the response returns the stored set.

### 6.6 Auth status payload

Extend `GET /auth/status` (and `/api/users/me` if it duplicates):

```jsonc
{
  "user": { "id": 1, "email": "bren@vil.nz", "name": "Bren", "isAdmin": true },
  "role": { "id": 1, "name": "Admin", "isSystemAdmin": true },
  "permissions": ["page.home", "page.dashboard", "..."]  // fully resolved
}
```

Admin returns the full expanded key list, not an empty array — the frontend then needs no special-casing.

---

## 7. Frontend

### 7.1 Context + hook

New `src/context/PermissionsContext.tsx` — provider mounted in `App.tsx` above `ShellPage`, fed from `/auth/status`.

```ts
const { can, canAny, permissions, role, isLoading, refresh } = usePermissions();
```

- `can(key: PermissionKey): boolean`
- In dev (`import.meta.env.DEV`), `can()` `console.warn`s if `key` isn't in the server-provided list — catches typos immediately.
- `PermissionKey` is a string-literal union in `src/types/permissions.ts`, maintained alongside the backend registry.

> This is the first slice of backlog **#11 (global state via React Context)** — the user object moves into this provider too. Note it in the PR description.

### 7.2 Components

- **`<Can permission="..." fallback={null}>`** — conditional render wrapper.
- **`<RequirePermission permission="...">`** — route element wrapper; renders `<NoAccessScreen />` (not a redirect loop) when denied.
- **`<NoAccessScreen />`** — friendly "You don't have access to this page — talk to Bren if you need it" with a link home.

### 7.3 Wiring

- **`Shell.page.tsx`** — nav items become a config array with a `permission` field; filter before render. Wrap each `<Route element>` in `<RequirePermission>`. A user landing on `/` with no `page.home` gets redirected to their first permitted page.
- **`Project.tsx`** — extract the tab list to a `PROJECT_TABS` config array (`{ value, label, permission }`), filter by `can()`. If the URL tab isn't permitted, redirect to the first permitted tab.
- **`Settings.tsx`** — same pattern with `SETTINGS_TABS`.
- **`ContactDetail.page.tsx`** — same pattern.
- **Axios interceptor** — on a 403 with `error: 'forbidden'`, toast "You don't have permission to do that" via the existing `notify` utility. Backstops any missed UI gate.

### 7.4 Settings → Roles tab (`components/Settings/RolesPanel.tsx`)

Layout: role list on the left, permission matrix on the right (stacks on mobile).

- Role list: name, description, user count, edit/delete icons. `+ Add role` button.
- Permission matrix: one `<Accordion>` (or `<Card>`) section per group from `GET /api/permissions`, each section a column of `<Checkbox>`es with a section-level "select all" indeterminate checkbox.
- Selecting the Admin role: all checkboxes checked and `disabled`, with an alert — *"Admin has full access to everything, including features added in future."*
- Dirty state → sticky Save / Cancel. Save `PATCH`es the whole `permissions` array.
- Delete confirmation modal; blocked with an explanatory message if the role is protected or in use.

### 7.5 Staff tab additions

Per the current table (Name / Email / User account / Cost/hr / Status):

- New **Role** column between *User account* and *Cost/hr*, showing a role badge or `—`.
- `StaffModal` gains a **Role** `<Select>`, only enabled when a user account is linked; helper text "Roles apply to the linked login account" when it isn't.
- Cost/hr column hidden entirely without `action.staff.view-cost-rates` (and absent from the payload anyway).

Also on the Roles tab: a small **"Login accounts"** section listing every `User`, their email, and their role, so an account with no staff record can still be assigned. Read-only list + role picker.

### 7.6 Lockout guards (enforced backend-side, mirrored in UI)

1. A user cannot change **their own** role → 409, picker disabled on your own row.
2. A user cannot remove `settings.tab.roles` / `action.roles.manage` from a role they themselves hold → 409.
3. The last user holding an `isSystemAdmin` role cannot be moved off it → 409 `"At least one admin is required"`.
4. `isProtected` roles cannot be deleted or have `isSystemAdmin` toggled; they *can* be renamed and (for Production Staff) have permissions edited.

---

## 8. Rollout

**PR 1 — backend foundation.** Registry, `Role` model + migration + seed, `roleId` on `User`, permission utils, `requirePermission`, `roleRoutes`, `/api/permissions`, `deserializeUser` include, extended `/auth/status`. `ensureAdmin` shim left in place so nothing breaks. Tests. Releases entry.

**PR 2 — route migration.** Swap every `ensureAdmin` for a specific `requirePermission`, add gates to the previously ungated routes in §6.3, add `sanitiseStaff`. Tests. Releases entry.

**PR 3 — frontend.** Context, hook, `<Can>`, `<RequirePermission>`, `NoAccessScreen`, nav/tab filtering, axios 403 interceptor. Tests. Releases entry.

**PR 4 — Roles UI.** `RolesPanel`, Settings tab, Staff tab Role column + modal picker, login accounts list. Tests. Releases entry.

**PR 5 — cleanup (separate backlog item).** Drop `User.isAdmin`, delete `ensureAdmin`, remove the shim.

Restart `express_api` after each backend PR (`docker restart express_api`).

---

## 9. Tests

### Backend (`*.test.ts` alongside source)

- `permissions/registry.test.ts` — keys are unique; keys match `^[a-z0-9-]+(\.[a-z0-9-]+)+$`; every key has a non-empty label and a valid group.
- `utils/permissions.test.ts` — `hasPermission` true for any key when `isSystemAdmin`; false for a null role; false for an unknown key; unknown stored keys filtered out by `rolePermissions`.
- `utils/requirePermission.test.ts` — 401 unauthenticated; 403 body shape `{ error, requiredPermission }`; `next()` on success.
- `routes/roleRoutes.test.ts` — create/read/update/delete happy paths; 409 deleting a protected role; 409 deleting a role in use; invalid keys stripped on write; self-role-change blocked; last-admin guard; duplicate name → 409.
- `utils/sanitiseStaff.test.ts` — `costPricePerHour` present with the permission, absent without it.
- One route-level regression test per newly gated endpoint asserting 403 for a permissionless user (parameterise over a table of `[method, path, permission]`).

### Frontend (Vitest + happy-dom)

- `context/PermissionsContext.test.tsx` — `can()` truth table; loading state; `refresh()` re-fetches.
- `components/Can.test.tsx` — renders children when permitted, fallback when not.
- `components/RequirePermission.test.tsx` — renders `NoAccessScreen` when denied.
- `Settings.test.tsx` / `Project.test.tsx` — tab arrays filter correctly; unpermitted URL tab redirects to the first permitted tab.
- `Settings/RolesPanel.test.tsx` — checkbox toggle updates dirty state; Admin role renders all-checked and disabled; delete blocked for protected roles; group "select all" sets every child.
- `Shell.test.tsx` — nav renders only permitted links.

---

## 10. Acceptance criteria

- [ ] `Role` model, `User.roleId`, `SystemSettings.defaultRoleId` migrated and seeded
- [ ] Admin + Production Staff roles exist; Bren and Bev hold Admin
- [ ] `GET /api/permissions` returns the grouped registry
- [ ] Roles CRUD works from Settings → Roles, including creating a third role
- [ ] Every permission key renders as a checkbox, grouped by area
- [ ] Admin's checkboxes are all-checked and disabled, with an explanatory alert
- [ ] Role assignable from the Staff tab (linked user accounts) and the Roles tab login list
- [ ] Sidebar links, project tabs, contact tabs, and settings tabs all hide without their permission
- [ ] Direct URL access to an unpermitted page shows `NoAccessScreen`, not a blank page or a crash
- [ ] Every previously `ensureAdmin` route returns 403 for a Production Staff user
- [ ] `costPricePerHour` absent from `GET /api/staff` without `action.staff.view-cost-rates`
- [ ] All four lockout guards return 409 with a readable message
- [ ] Backend + frontend tests written and passing (`npm test`, `yarn test`)
- [ ] `ReleasesPanel.tsx` updated in **every** PR
- [ ] `CLAUDE.md` updated: permission registry, `requirePermission`, and the new-feature-needs-a-key standard
- [ ] `backlog.md` #39 marked shipped; cleanup item added for removing `isAdmin`

---

## 11. Open questions

1. **Corry, Terry, Vlad have no linked user account.** They can't log in today. Do they need logins (and therefore roles), or are they shop-floor-only via someone else's session?
2. **Production Staff and the Kanban/Dashboard** — included above. Should they also see `page.contacts`?
3. **Timesheets:** Production Staff can see their own via My VisualOS. Should the *project* Timesheets tab be visible to them (all staff's hours on that job) or admin-only?
4. **Read vs write split** — this spec gates visibility, with a handful of write-specific action keys. If you want `view` / `edit` pairs on tabs (e.g. see the Financial tab read-only), say so now; it roughly doubles the key count.
