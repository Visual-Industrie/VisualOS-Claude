# VisualOS — Financial Overview Page (Invoicing vs Budget)

**Status:** 🟢 Ready to build
**Priority:** High
**Labels:** `feature`, `financial`, `admin-only`, `reporting`, `xero`, `budget`, `fix`, `tech-debt`
**Last Updated:** July 2026

---

## 1. Goal

An admin-only `/financial-overview` page that answers one question at a glance:

> *"How much have we invoiced this month, and does it cover what it costs to run the business this month?"*

It compares **invoiced contribution** (what each job actually earned us, after stripping material and expense **costs** but leaving labour in) against the **monthly budgeted expenditure** from the Budget page — rendered as two gauges:

- **Coverage** — how much of this month's overheads the month's invoicing has covered so far. 100% = break-even for the month; over 100% = real profit.
- **Pace** — the same figure scaled by how far through the month we are. Tells us if we're ahead of or behind the run-rate we need to hit break-even.

> **Companion change (§14):** this spec also fixes a standing oversight — the Financial tab's per-row **Est/Act** and **Charge** toggles for staff time, materials, and expenses are currently held only in React state and reset on every reload. §14 persists all six. The Overview itself reads revenue from `xeroTotal` so it's insulated either way, but the tab you build invoices on is running on unsaved state, so it's worth fixing in the same pass.

---

## 2. The metric

### 2.1 Why labour stays in

Wages (NETT) and PAYE + KiwiSaver are line items in the **Budget** (`BudgetItem`), i.e. they're already counted as a monthly outflow. If we *also* subtracted staff cost from each project we'd charge labour twice. So a project's contribution keeps the labour it billed the customer and only strips the direct costs that are **not** in the budget: materials and expenses.

> ⚠️ **Do not reuse the Financial tab's "Profit" line.** That figure subtracts staff cost. This page deliberately does not — reusing it reintroduces the double-count.

### 2.2 Definitions

For an invoice `i` on project `P`:

```
invoiceContribution(i) = i.xeroTotal − allocatedDirectCost(i)
```

Where `xeroTotal` is the pre-tax invoice total (already includes labour charge-out + marked-up materials/expenses − discount, straight from Xero), and `allocatedDirectCost` is the project's material + expense **cost** apportioned to this invoice (see §2.4).

```
monthlyContribution(m)  = Σ invoiceContribution(i)  for all i whose invoice date falls in month m
budgetMonthlyTotal      = Σ normaliseToMonthly(BudgetItem)          (see §6.4)
elapsedFraction(m)      = dayOfMonth / daysInMonth  (current month) | 1 (past) | 0 (future)

coverage(m) = monthlyContribution(m) / budgetMonthlyTotal
pace(m)     = monthlyContribution(m) / (budgetMonthlyTotal × elapsedFraction(m))
```

Both ratios can exceed 100% (a strong month covers more than the budget). The gauges must render overshoot gracefully.

### 2.3 Worked example (Bren's numbers)

Budget = $10,000/mo, it's the 15th of a 30-day month, month's contribution so far = $4,000.

- `coverage = 4000 / 10000 = 40%`
- `elapsedFraction = 15/30 = 0.5`
- `pace = 4000 / (10000 × 0.5) = 4000 / 5000 = 80%`

*(80% — the earlier "90%" was an arithmetic slip.)*

### 2.4 Multiple invoices per project (pro-rating)

A project can carry several invoices (deposit + final, progress claims). We split the project's total direct cost across its invoices in proportion to each invoice's value, so each invoice lands in its own month carrying its fair share of cost:

```
T            = Σ inv.xeroTotal  for all inv on the project where xeroTotal is not null
directCost   = materialsCost(P) + expensesCost(P)
allocatedDirectCost(i) = T > 0 ? (i.xeroTotal / T) × directCost : 0
```

- **Single-invoice job (the norm):** share = 1 → `contribution = xeroTotal − directCost`. Unchanged.
- **T ≤ 0** (all totals null/zero): allocate 0 cost, flag the project as needing a total refresh.
- Invoices with `xeroTotal == null` are **excluded** from the sum and **flagged** (see §7.4) — never silently treated as 0.

### 2.5 Cost resolution (from VisualOS, not Xero)

```
materialsCost(P) = Σ over P.projectMaterials:
    qty      = pm.actualQty ?? pm.estimatedQty ?? 0
    unitCost = pm.costPriceOverride ?? pm.material?.purchasePricePerUnit ?? 0
    qty × unitCost

expensesCost(P)  = Σ over P.projectExpenses:
    pe.actualCost ?? pe.estimatedCost ?? 0
```

No dependency on the Financial tab's Est/Act *charge* toggle — we use actual-else-estimate for cost, which is well-defined regardless of how the row is billed.

> **Interaction with the persisted toggles (§14):** the Overview cost side stays **actual-else-estimate** on purpose, independent of the new `financialUseActual` and `financialChargeable` fields. `financialUseActual` governs the *billing basis* on the tab, not what we actually spent; and an *unchargeable* row (absorbed, not billed to the customer) still cost us money, so it still reduces contribution. Contribution should reflect true margin, so the cost side counts every material/expense at its real cost regardless of how it was billed.

**GST assumption:** all figures are pre-tax / GST-exclusive on both sides (`xeroTotal` is pre-tax; material/expense costs entered ex-GST; budget items ex-GST). Sanity-check any GST-inclusive budget lines (e.g. rent) when reconciling — not a blocker.

---

## 3. Data sources (all local — no Xero calls at aggregation time)

| Need | Source |
|---|---|
| Invoiced revenue | `ProjectInvoice.xeroTotal` (pre-tax) |
| Invoice date | `ProjectInvoice.xeroInvoiceDate` **(new — §4.1)** |
| Materials cost | `ProjectMaterial` (`costPriceOverride` / `material.purchasePricePerUnit`, `actualQty`/`estimatedQty`) |
| Expenses cost | `ProjectExpense` (`actualCost`/`estimatedCost`) |
| Monthly budget | `BudgetItem` (`frequency`, `amount`) — live-normalised |
| "Expects an invoice" stages | `TaxonomyItem` `project_stage` `meta.expectsInvoice` **(new — §4.2)** |

Because we linked each invoice to its project locally when it was created/linked, the old Xero Projects↔invoice join limitation never bites here.

---

## 4. Schema changes

### 4.1 `ProjectInvoice.xeroInvoiceDate`

```prisma
model ProjectInvoice {
  id                Int       @id @default(autoincrement())
  projectId         Int
  project           Project   @relation(fields: [projectId], references: [id], onDelete: Cascade)
  xeroInvoiceId     String
  xeroInvoiceNumber String?
  xeroInvoiceUrl    String
  xeroTotal         Decimal?
  xeroInvoiceDate   DateTime?  // NEW — Xero invoice.Date (date-only), the bucketing date
  createdAt         DateTime  @default(now())
}
```

- Populated from the Xero invoice's own `Date` field on create/link (§6.5) and via one-off backfill (§6.6).
- Nullable so existing rows migrate cleanly; the backfill fills them.

### 4.2 `project_stage` `meta.expectsInvoice`

No column change — extends the existing `meta` JSON on `TaxonomyItem` (type `project_stage`). Migration seeds `expectsInvoice: true` on the stages that should have an invoice by the time they're reached:

```sql
UPDATE "TaxonomyItem"
SET meta = COALESCE(meta, '{}'::jsonb) || '{"expectsInvoice": true}'::jsonb
WHERE type = 'project_stage' AND name IN ('Invoice', 'Completed');
```

> Confirm the exact stage `name`s in production before running (they're user-editable). Adjust the `IN (...)` list if renamed.

Used only by the "No invoice" badge (§7.5). The invoicing calculation itself keys purely off the presence of `ProjectInvoice` rows, so no `isInvoiced` flag is needed.

---

## 5. TypeScript types

**Backend** (`src/types/financialOverview.ts` or inline in the route):

```ts
interface InvoiceContribution {
  invoiceId: number;
  projectId: number;
  projectName: string;
  xeroInvoiceNumber: string | null;
  xeroInvoiceUrl: string;
  invoiceDate: string;          // YYYY-MM-DD
  xeroTotal: number;
  allocatedDirectCost: number;
  contribution: number;
}

interface FinancialOverviewResponse {
  month: string;                // 'YYYY-MM'
  budgetMonthlyTotal: number;
  monthlyContribution: number;
  coverage: number;             // 0..>1
  elapsedFraction: number;      // 0..1
  pace: number | null;          // null for future months
  invoiceCount: number;
  contributions: InvoiceContribution[];   // drill-down list
  gaps: {
    invoicesMissingTotal: { invoiceId: number; projectId: number; projectName: string }[];
    projectsMissingInvoice: { projectId: number; projectName: string; status: string }[];
  };
}
```

**Frontend** (`src/types/financialOverview.ts`): mirror the above as `IFinancialOverview`, `IInvoiceContribution`.

---

## 6. Backend

### 6.1 Shared util — `src/utils/financialOverview.ts`

Pure functions, individually unit-tested:

- `materialsCost(project): number`
- `expensesCost(project): number`
- `invoiceContributions(project): InvoiceContribution[]` — applies §2.4 pro-rating across the project's invoices.
- `normaliseBudgetToMonthly(items): number` — §6.4.
- `elapsedFraction(month, now): number` — §6.3.

Keep all money maths in a single module so the route is thin and the logic is testable in isolation.

### 6.2 Route — `GET /api/financial-overview?month=YYYY-MM`

- `ensureAuthenticated` + `ensureAdmin`.
- `month` defaults to the current **Pacific/Auckland** month.
- Query plan (avoid N+1):
  1. Find `ProjectInvoice` rows whose `xeroInvoiceDate` falls in `month` (+ the null-total ones for the gap report).
  2. Collect their `projectId`s; load those projects **once** with `{ invoices, projectMaterials: { include: material }, projectExpenses }`. (Siblings in other months are needed for the pro-rating denominator `T`.)
  3. Compute `invoiceContributions(project)`, keep only invoices dated in `month`, sum → `monthlyContribution`.
  4. `budgetMonthlyTotal` from all `BudgetItem` via `normaliseBudgetToMonthly`.
  5. `elapsedFraction`, `coverage`, `pace`.
  6. Build `gaps` (§7.4/§7.5).

### 6.3 Elapsed fraction (Auckland)

```
nowAK   = dayjs().tz('Pacific/Auckland')
target  = dayjs.tz(`${month}-01`, 'Pacific/Auckland')
if target is a past month      → 1
if target is a future month    → 0   (pace = null; hide the pace gauge)
else (current month)           → nowAK.date() / nowAK.daysInMonth()
```

Use `dayjs` with `utc` + `timezone` plugins (dayjs is already in the stack).

### 6.4 Budget normalisation

Match the Budget page's own footer exactly (52 weeks / 12 months) so the numbers agree:

```
weekly  → amount × 52 / 12
monthly → amount
annual  → amount / 12
```

### 6.5 Capture invoice date on create/link (`invoiceRoutes.ts`)

- `POST /api/projects/:id/invoice` (create): set `xeroInvoiceDate` from the created invoice's `Date` (today).
- "Link existing" flow: fetch the linked invoice and set `xeroInvoiceDate` from its real `Date` (may be well in the past).
- Populate `xeroTotal` at the same time if not already set.

### 6.6 One-off backfill (temp admin endpoint)

Per the NAS constraint, run as a temporary admin-only endpoint, not CLI:

- `POST /api/admin/backfill-invoice-dates` (`ensureAdmin`).
- For each `ProjectInvoice` with null `xeroInvoiceDate`: fetch the Xero invoice by `xeroInvoiceId`, set `xeroInvoiceDate` from `.Date`, and set `xeroTotal` if null.
- Voided/deleted-in-Xero invoices: leave null, collect in the response so they surface in the gaps report.
- Return `{ updated, skipped, failures[] }`. Remove the endpoint once run (note in the PR).

---

## 7. Frontend

### 7.1 Page — `src/pages/FinancialOverview.page.tsx` (`/financial-overview`, admin only)

- Sidebar link (admin-gated, same pattern as other admin routes).
- Month picker (Mantine `MonthPickerInput`), default = current Auckland month; back/forward arrows.
- Header stat row: Budget (monthly), Invoiced contribution (this month), Coverage %, Pace %.
- Two gauges (§7.2).
- Drill-down list of the month's invoices (§7.3).
- Gaps panel (§7.4).

### 7.2 Gauges (Recharts `RadialBarChart`)

- **Coverage gauge** — `monthlyContribution / budgetMonthlyTotal`. Centre label: the %, subtitle `$contribution of $budget`.
- **Pace gauge** — `monthlyContribution / (budget × elapsedFraction)`. Subtitle: `on track for $target by day N`.
- Overshoot: cap the arc at 100% but show the true numeric % in the centre; add a distinct "over 100%" colour so beating budget reads as a win, not a full/empty ambiguity.
- Colour thresholds (suggestion): pace < 90% red, 90–100% amber, ≥ 100% teal (reuse the Financial tab's teal/red profit palette for consistency).
- Hide the pace gauge for future months (`pace === null`).

### 7.3 Drill-down list

Table of `contributions[]`: project (link), invoice number (→ `xeroInvoiceUrl`), invoice date, `xeroTotal`, allocated cost, contribution. Lets you see what drove the month.

### 7.4 Gaps: invoices missing a total

Small callout listing `gaps.invoicesMissingTotal` with a "Refresh totals" affordance (reuse the existing ↻ refresh) so nulls get filled and stop undercounting.

### 7.5 "No invoice" badge on Home (`Home.page.tsx`)

- Orange badge reading **No invoice**, placed after the project name (same position/treatment as the CLOSED badge).
- Shows when: the project's current `status` is a `project_stage` with `meta.expectsInvoice === true` **and** the project has zero `ProjectInvoice` rows.
- Requires the Home list query/response to include an invoice count (or boolean `hasInvoice`) per project — add a lightweight `_count: { invoices: true }` to the existing Home projects query rather than a second round-trip.

### 7.6 Types

Add `IFinancialOverview` / `IInvoiceContribution` to `src/types/financialOverview.ts`; extend the Home project type with `hasInvoice`/invoice count.

---

## 8. Tests

### 8.1 Backend (`*.test.ts`, vitest — see backlog #12 re: backend vitest setup; if not yet wired, this is the forcing function)

- `materialsCost` / `expensesCost`: override vs linked-material price; freeform (no material, no override) = 0; actual-vs-estimate qty/cost fallback.
- `invoiceContributions`:
  - single invoice → `xeroTotal − directCost`.
  - two invoices → cost split by value, shares sum to `directCost`, each contribution correct.
  - `T ≤ 0` guard → 0 allocated cost, no divide-by-zero.
  - null `xeroTotal` invoice excluded from `T` and from output, surfaced as a gap.
- `normaliseBudgetToMonthly`: weekly×52/12, annual/12, monthly as-is; mixed set matches the Budget page footer.
- `elapsedFraction`: mid-month, day 1, last day, past month = 1, future month = 0.
- **Month bucketing / tz**: an invoice dated on a month boundary lands in the correct NZ month (the classic UTC-midnight edge that bit timesheets before).
- `coverage` / `pace` incl. > 100%.

### 8.2 Frontend (vitest)

- Gauge component: renders the %, handles overshoot > 100%, applies colour thresholds, hides pace when null.
- "No invoice" badge: shows for an `expectsInvoice` stage with no invoice; hidden otherwise.
- Month picker defaults to the current Auckland month.

---

## 9. Edge cases

- Project in an `expectsInvoice` stage with materials/expenses but no invoice → no contribution counted (correct); flagged for the badge + gaps.
- Invoice dated before `ProjectStatusLog`/invoicing existed but with a real `xeroInvoiceDate` after backfill → still buckets correctly (we key off the Xero date, not status history).
- Discount / manual override already baked into `xeroTotal` → nothing extra to do; never re-apply discount.
- `budgetMonthlyTotal === 0` (empty budget) → guard coverage/pace against divide-by-zero; show "set a budget" empty state.

---

## 10. Implementation order

1. Schema: add `ProjectInvoice.xeroInvoiceDate`; migration for the field + the `expectsInvoice` seed. (`prisma migrate` — session-table drift workaround per CLAUDE.md.)
2. `src/utils/financialOverview.ts` + full backend unit tests (§8.1).
3. Capture `xeroInvoiceDate` (+ `xeroTotal`) on create/link in `invoiceRoutes.ts`.
4. `GET /api/financial-overview` route (admin-gated) + route test.
5. One-off backfill endpoint (§6.6); run in prod; note removal.
6. Home query: add invoice count; "No invoice" badge on `Home.page.tsx` + test.
7. `FinancialOverview.page.tsx`: month picker, stat row, two gauges, drill-down, gaps + tests.
8. Sidebar link (admin only); types.
9. `expectsInvoice` toggle in the Lists panel (`TaxonomyPanel`) so it's user-managed (optional but consistent — a checkbox alongside `showInKanban`/`closesXero`).
10. **Companion (§14):** add the six `financial*` fields (migration with defaults — no data backfill); wire PATCH persistence on Task / ProjectMaterial / ProjectExpense; switch the Financial tab rows from local state to persisted values (save-on-toggle) + tests.
11. `ReleasesPanel.tsx` entry; add the feature to `backlog.md` → `complete.md`.

---

## 11. Acceptance criteria

- [ ] `/financial-overview` is admin-only (403 for non-admins; link hidden).
- [ ] Coverage gauge = month's contribution ÷ monthly budget; renders > 100% clearly.
- [ ] Pace gauge = contribution ÷ (budget × day-fraction); hidden for future months.
- [ ] Contribution uses `xeroTotal` − pro-rated (materials + expenses) cost, labour untouched.
- [ ] Multiple invoices bucket independently by their own Xero date with cost split by value.
- [ ] Monthly budget matches the Budget page footer to the cent.
- [ ] Month boundaries + elapsed fraction computed in Pacific/Auckland.
- [ ] `xeroInvoiceDate` captured on create/link; existing invoices backfilled.
- [ ] Orange "No invoice" badge on Home for `expectsInvoice`-stage projects with no invoice.
- [ ] Invoices with null totals surfaced (not counted as 0).
- [ ] **§14:** the six `financial*` toggle fields persist; Financial tab Est/Act + Charge selections survive reload; migration defaults leave existing projects rendering identically.
- [ ] Backend + frontend unit tests per §8 and §14, all green.
- [ ] `ReleasesPanel.tsx` updated in the same PR; feature branch; **PR opened, not merged**.

---

## 12. Open questions

None blocking. Two low-stakes confirmations for when you build:

1. Exact production `name`s of the invoice-expecting stages (for the `expectsInvoice` seed) — assumed `Invoice` and `Completed`.
2. Whether you want the `expectsInvoice` toggle exposed in Settings → Lists now (step 9) or just seeded and left implicit for v1.

*Resolved:* toggle fields prefixed `financial*` on all three models; `financialUseActual` defaults `false` (Est), `financialChargeable` defaults `true`; persisted save-on-toggle.

---

## 13. Notes

- **Multi-tenancy (#32):** `BudgetItem`, `ProjectInvoice`, and the overview query all gain an `organisationId` scope when the org layer lands — keep the aggregation parameterised by org-able queries, don't assume a single global budget.
- **Possible follow-on (not in scope):** a monthly-trend bar chart (contribution vs budget line over the last N months) drops straight onto this endpoint by looping the month param — easy phase 2.
- Feature branch (`feature/FinancialOverview`), tests for every new util/route, releases entry in-PR, do not merge without explicit sign-off — per CLAUDE.md.

---

## 14. Companion change — persist the Financial tab Est/Act + Charge toggles

### 14.1 Problem

The Financial tab shows two independent switches on every staff-time, material, and expense row:

- **Use (Est / Act)** — which figure feeds the charge-out basis.
- **Charge** (checkbox) — whether the row counts toward revenue at all.

Neither is persisted today. `Task`, `ProjectMaterial`, and `ProjectExpense` store the est/actual *values* but not the *selections*, so both switches live only in React state and reset to defaults on reload. Any server-side reader has to re-guess what was actually billed. This spec's Overview dodges it by reading `xeroTotal`, but the tab you build invoices on is running on unsaved state — fix it here.

### 14.2 Schema — six new fields (all `financial*` prefixed, consistent across the three models)

```prisma
// Task
financialUseActual   Boolean @default(false)   // false = Est basis, true = Act
financialChargeable  Boolean @default(true)    // Charge checkbox

// ProjectMaterial
financialUseActual   Boolean @default(false)
financialChargeable  Boolean @default(true)

// ProjectExpense
financialUseActual   Boolean @default(false)
financialChargeable  Boolean @default(true)
```

- Defaults match the current UI (Est basis, chargeable), so **existing rows render identically** after migration — no data backfill needed, the column defaults do the work.
- One migration for all six. `prisma migrate` with the session-table drift workaround per CLAUDE.md.

### 14.3 Backend — PATCH persistence (save-on-toggle)

Extend the existing partial-update routes to accept the two booleans; no new endpoints:

- **Staff time:** `PATCH /tasks/:id` (already partial) → accept `financialUseActual`, `financialChargeable`.
- **Materials:** the project-material update route (the one already saving `salePriceOverride` / `costPriceOverride` on blur) → accept both.
- **Expenses:** the project-expense update route → accept both.

Partial updates only — never clobber sibling fields. Each toggle PATCHes on change (optimistic UI), matching the tab's existing on-blur autosave feel.

### 14.4 Frontend — read persisted values, drop local toggle state

- The Financial tab staff/material/expense row components read `financialUseActual` / `financialChargeable` from the fetched row instead of `useState`.
- Toggling fires the PATCH and updates optimistically; on error, revert + `notify.error` (same pattern as the price-on-blur save).
- The tab's charge-out calc and the "Charge" revenue filter derive from the persisted values, so the invoice you build is reproducible across reloads.
- Extend the frontend row types (`ITask`, and the material/expense row interfaces used by the Financial tab) with the two booleans.

### 14.5 Tests

**Backend:**
- `PATCH` persists `financialUseActual` and `financialChargeable` independently on each of Task / ProjectMaterial / ProjectExpense.
- Partial update of a toggle leaves other fields untouched.
- Migration defaults: a freshly created row is `financialUseActual = false`, `financialChargeable = true`.

**Frontend:**
- Toggling Use or Charge calls the PATCH with the right payload.
- Row renders from the persisted prop, not local state (re-render from fetched data preserves the selection — the reload-safety guarantee).
- Charge-out subtotal responds to `financialUseActual`; an unchargeable row is excluded from the revenue subtotal but still shown.

### 14.6 Acceptance criteria

- [ ] Six `financial*` fields added with UI-matching defaults; single migration; existing projects visually unchanged.
- [ ] Est/Act and Charge selections persist per row on Task / ProjectMaterial / ProjectExpense and survive reload.
- [ ] Toggles save on change (optimistic, with revert on error).
- [ ] Backend + frontend tests per §14.5 green.
- [ ] `ReleasesPanel.tsx` entry (plain-English: "Est/Act and Charge choices on the Financial tab now save").
