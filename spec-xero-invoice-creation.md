# Spec: Xero Invoice Creation from Financial Tab

**Feature:** Create a Xero draft invoice directly from the VisualOS Financial tab  
**Status:** 📋 Ready to Build  
**Priority:** High  
**Labels:** `feature`, `xero-api`, `financial`, `backend`, `frontend`

---

## Overview

When an admin views the Financial tab on a project, they can click **Create Invoice** to generate a draft invoice in Xero. A modal lets them choose how to structure the invoice lines (single line, labour + materials split, individual task lines, individual material lines, or fully itemised). On success, the invoice number and a deep-link to Xero are stored on the project and surfaced in the UI.

---

## User Stories

- As an admin, I can create a Xero draft invoice from the Financial tab without leaving VisualOS.
- I can choose how labour and materials are grouped as invoice line items.
- I'm warned before creation if any pricing information is missing (labour rates, material sale prices).
- After creation, I can click **View Invoice in Xero** to open the invoice directly.
- The Create Invoice button remains available after creation (re-creates / overwrites if needed — Xero draft can be manually deleted there).

---

## UI Changes

### Financial Tab — Header Area

Add two buttons right-aligned in the Financial tab header, alongside any existing controls:

```
[ Create Invoice ]   [ View Invoice in Xero ]  ← disabled until invoice exists
```

- **Create Invoice** — always enabled for admins; opens the invoice creation modal.
- **View Invoice in Xero** — disabled (greyed out) until `project.xeroInvoiceId` is set; opens `https://go.xero.com/AccountsReceivable/Edit.aspx?InvoiceID={xeroInvoiceId}` in a new tab.
- Both buttons sit inside the existing admin Financial tab guard (non-admins never see this tab).

### Invoice Creation Modal

`<InvoiceCreationModal>` — opened by Create Invoice button.

**Title:** "Create Xero Invoice"

#### Step 1 — Line Item Structure (radio group)

Present five options with short descriptions:

| Value | Label | Description |
|---|---|---|
| `single` | Single line item | One line: job description. Total = calculated invoice total. |
| `labour_materials` | Labour + Materials (2 lines) | Two lines: one for all labour, one for all materials. |
| `labour_itemised` | Itemised labour, combined materials | One line per Task (using task title as description), single materials line. |
| `materials_itemised` | Combined labour, itemised materials | Single labour line, one line per material used on the project. |
| `fully_itemised` | Fully itemised | One line per Task + one line per material. |

Default selection: `single`.

#### Step 2 — Validation / Warnings Panel

Before allowing invoice creation, the modal checks for missing pricing data and displays inline warnings (yellow `Alert` with list of issues). The **Create** button is disabled while any blocking issues exist. Issues are split into:

**Blocking (must fix before creating):**
- Contact has no Xero contact linked (project has no `contactId` / associated Xero contact)
- Selected structure includes labour lines but no charge-out rate is set for the project

**Warnings (show but allow creation — line omitted or uses $0):**
- One or more tasks have no `estimatedMinutes` (labour line will use 0 hours)
- One or more materials have no sale price set (line will use $0)
- No completed time entries exist (labour hours = 0)

Display warnings immediately when the modal opens (pre-check on mount). Re-validate when the user changes the line structure option.

#### Step 3 — Confirmation / Create Button

Below the warnings panel:

- **Description override** — optional single `TextInput` field labelled "Invoice description / reference". Pre-filled with the project name + customer name (e.g. `"Shop signage — Acme Ltd"`). Used as the Description on single-line invoices and as a `Reference` field on all invoices.
- **Due date** — `DatePickerInput` labelled "Due date (optional)". Defaults to 30 days from today.
- **[ Cancel ]** and **[ Create Invoice ]** buttons. Create is disabled while warnings are blocking or while the API call is in-flight (show spinner).

---

## Data Model Changes

### `Project` model — new fields

```prisma
xeroInvoiceId     String?   // Xero InvoiceID (UUID)
xeroInvoiceNumber String?   // Human-readable: INV-XXXX
xeroInvoiceUrl    String?   // Deep link to Xero invoice edit page
```

Migration name: `add_xero_invoice_fields_to_project`

Update `IProject` TypeScript interface in `projects-frontend/src/types/` to add these three optional string fields.

---

## Backend

### New Route File: `invoiceRoutes.ts`

Mount at: `POST /api/projects/:id/invoice`

#### `POST /api/projects/:id/invoice`

- Requires `ensureAuthenticated` + `ensureAdmin`
- Body:

```typescript
{
  structure: 'single' | 'labour_materials' | 'labour_itemised' | 'materials_itemised' | 'fully_itemised';
  description?: string;   // invoice reference / description override
  dueDate?: string;       // ISO date string, e.g. "2026-06-30"
}
```

**Processing steps:**

1. Load the project with relations:
   - `contact` (for `xeroContactId`)
   - `tasks` where `eventType IS NULL` (exclude calendar events; include completed tasks with time entries) and `isPhoneMessage = false` and `status != 'draft'`
   - `tasks.timeEntries` (to sum actual hours)
   - `projectFinancials` (for `chargeOutRateId`, `discountType`, `discountValue`)
   - materials via the project's material lines (however these are stored — see Financial tab implementation)

2. Load the active charge-out rate from `ProjectFinancials.chargeOutRateId` → `ChargeOutRate` model.

3. Build line items per the requested `structure` (see Line Item Logic below).

4. Call Xero Accounting API: `POST /api.xro/2.0/Invoices` via `xero-node` SDK.

5. On success: `PATCH project SET xeroInvoiceId, xeroInvoiceNumber, xeroInvoiceUrl WHERE id = projectId`.

6. Return `{ invoiceId, invoiceNumber, invoiceUrl }` with HTTP 201.

7. On Xero error: return 502 with the Xero error body for the frontend to surface.

#### Xero Invoice Payload Shape

```typescript
{
  Type: 'ACCREC',
  Contact: { ContactID: project.contact.xeroContactId },
  Status: 'DRAFT',
  Reference: description ?? `${project.name} — ${project.contact.name}`,
  DueDate: dueDate ?? (today + 30 days),   // formatted as "YYYY-MM-DD"
  LineAmountTypes: 'EXCLUSIVE',             // prices ex-GST; Xero applies tax
  LineItems: [ ...built per structure ],
}
```

Each `LineItem`:

```typescript
{
  Description: string,
  Quantity: number,        // hours for labour, qty for materials
  UnitAmount: number,      // hourly rate or sale price per unit
  AccountCode: string,     // use a configurable default — see note below
  TaxType: 'OUTPUT2',      // NZ GST on income
}
```

> **Account Code note:** Store a `defaultSalesAccountCode` in `SystemSettings` (new field, default `'200'` which is the standard Xero sales account). Admin can update it in Settings → General. Use it for all lines. A future enhancement could allow per-line overrides.

#### Line Item Logic

**Labour calculation:**

- Source: sum of `TimeEntry.stoppedAt - TimeEntry.startedAt` (in hours, 2 dp) per task, falling back to `Task.estimatedMinutes / 60` if no time entries exist for that task.
- Rate: `ChargeOutRate.ratePerHour` for the project's selected rate.
- Labour line description: task title (itemised) or `"Labour"` (combined).

**Materials calculation:**

- Source: project material lines with quantity and `Material.salePrice`.
- Materials line description: `Material.description ?? Material.name` (itemised) or `"Materials"` (combined).
- Skip materials with `salePrice = null` and log a warning — do not create a $0 line.

**Structure → line item mapping:**

| Structure | Lines produced |
|---|---|
| `single` | 1 line: description override, qty=1, unit=Financial tab "Total to invoice" (or calculated total) |
| `labour_materials` | 1 labour line (total hours × rate) + 1 materials line (sum of material sale prices × qty) |
| `labour_itemised` | 1 line per Task + 1 combined materials line |
| `materials_itemised` | 1 combined labour line + 1 line per material |
| `fully_itemised` | 1 line per Task + 1 line per material |

For `single` structure, use `ProjectFinancials.invoiceOverride` if set, otherwise the calculated Financial tab total (staff revenue + materials revenue − discount).

#### Validation endpoint (optional — alternatively validated client-side)

If preferred, expose `GET /api/projects/:id/invoice/validate` which returns the warnings list without creating anything. This keeps the modal snappy without round-tripping on every option change. The frontend can call this once on modal open.

Alternatively, the frontend can compute warnings locally from data already loaded on the Financial tab — preferred to avoid an extra endpoint.

---

## Frontend

### New Component: `InvoiceCreationModal.tsx`

Location: `projects-frontend/src/components/Project/Tabs/Financial/InvoiceCreationModal.tsx`

Props:
```typescript
interface InvoiceCreationModalProps {
  opened: boolean;
  onClose: () => void;
  project: IProject;
  financialData: FinancialTabData;  // tasks, materials, time entries, chargeOutRate — already loaded by FinancialTab
  onSuccess: (result: { invoiceId: string; invoiceNumber: string; invoiceUrl: string }) => void;
}
```

**State:**
- `structure` — radio value, default `'single'`
- `description` — string, pre-filled
- `dueDate` — Date | null, default today + 30 days
- `warnings` — `{ blocking: string[]; advisory: string[] }` — computed on mount and on structure change
- `loading` — boolean

**Validation logic (pure function, testable):**

```typescript
export function validateInvoiceData(
  project: IProject,
  financialData: FinancialTabData,
  structure: InvoiceStructure
): { blocking: string[]; advisory: string[] }
```

Rules:
- Blocking: `!project.contact?.xeroContactId` → "Project has no linked Xero contact"
- Blocking: structure includes labour + no charge-out rate set → "No charge-out rate selected for this project"
- Advisory: any task included in labour lines has no time entries AND no `estimatedMinutes` → "X task(s) have no hours recorded — they will contribute 0 hours"
- Advisory: any material line has `salePrice = null` → "X material(s) have no sale price — they will be skipped"
- Advisory: total labour hours = 0 → "No labour hours found — labour line(s) will be $0"

**On submit:**
- `POST /api/projects/:projectId/invoice` with body `{ structure, description, dueDate }`
- On success: call `onSuccess(result)`, close modal, show success toast: `"Invoice INV-XXXX created in Xero"`
- On error: show error toast with Xero's error message if available, keep modal open

### Changes to `FinancialTab.tsx`

1. Add local state: `invoiceId`, `invoiceNumber`, `invoiceUrl` — initialised from `project.xeroInvoiceId` etc. on mount.
2. Add `invoiceModalOpen` boolean state.
3. Render in the tab header (admin guard already wraps the whole tab):

```tsx
<Group justify="flex-end" mb="md">
  <Button variant="outline" onClick={() => setInvoiceModalOpen(true)}>
    Create Invoice
  </Button>
  <Button
    variant="light"
    color="blue"
    disabled={!invoiceId}
    component="a"
    href={invoiceUrl ?? '#'}
    target="_blank"
    rel="noopener noreferrer"
  >
    View Invoice in Xero
  </Button>
</Group>
```

4. On `InvoiceCreationModal` `onSuccess`: update local state with new `invoiceId`, `invoiceNumber`, `invoiceUrl`.
5. Render `<InvoiceCreationModal>` passing `financialData` from existing tab state.

### Update `IProject` type

Add to `projects-frontend/src/types/` (whichever file holds `IProject`):

```typescript
xeroInvoiceId?:     string;
xeroInvoiceNumber?: string;
xeroInvoiceUrl?:    string;
```

---

## Settings Change

### `SystemSettings` — new field

```prisma
defaultSalesAccountCode  String  @default("200")
```

Migration name: `add_default_sales_account_code_to_system_settings`

### Settings → General tab

Add a small "Invoicing" section (admin only) with a single `TextInput` for **Default Xero sales account code** (default `200`). Save via existing `PATCH /api/settings` endpoint (extend to accept this field).

---

## Tests

### Backend — `invoiceRoutes.test.ts`

```
✓ POST /api/projects/:id/invoice — returns 403 for non-admin
✓ POST /api/projects/:id/invoice — returns 400 if project has no Xero contact
✓ POST /api/projects/:id/invoice — builds correct single-line payload
✓ POST /api/projects/:id/invoice — builds correct labour+materials payload
✓ POST /api/projects/:id/invoice — builds correct fully_itemised payload
✓ POST /api/projects/:id/invoice — saves xeroInvoiceId on project after success
✓ POST /api/projects/:id/invoice — returns 502 on Xero API failure
```

Mock `xero-node` SDK responses. Use existing `vitest` infrastructure once backend Vitest is set up (#12); otherwise add as a noted gap.

### Frontend — `InvoiceCreationModal.test.tsx`

```
✓ validateInvoiceData — blocking: missing Xero contact
✓ validateInvoiceData — blocking: labour structure with no charge-out rate
✓ validateInvoiceData — advisory: tasks with no hours
✓ validateInvoiceData — advisory: materials with no sale price
✓ validateInvoiceData — no warnings when all data present
✓ InvoiceCreationModal — renders structure options
✓ InvoiceCreationModal — disables Create button while blocking warnings present
✓ InvoiceCreationModal — calls onSuccess and closes on API success
```

---

## Xero API Notes

- Endpoint: `POST https://api.xero.com/api.xro/2.0/Invoices`
- Auth: existing `xero-node` SDK token flow (same pattern as `xeroRoutes.ts` and `contactRoutes.ts`)
- The SDK method: `xero.accountingApi.createInvoices(tenantId, { invoices: [payload] })`
- Response: `response.body.invoices[0]` contains `InvoiceID`, `InvoiceNumber`, and `OnlineInvoiceUrl` (or construct the edit URL as `https://go.xero.com/AccountsReceivable/Edit.aspx?InvoiceID={InvoiceID}`)
- Tax type for NZ: `OUTPUT2` (15% GST)
- `LineAmountTypes: 'EXCLUSIVE'` means unit amounts are ex-GST — Xero calculates GST on top
- Contact must exist in Xero — use `project.contact.xeroContactId` (already available via the `contactId` FK)
- Invoices created as `DRAFT` — no side effects until the user approves/sends from Xero

---

## Implementation Order

1. **Backend** — Prisma migration (3 fields on Project + `defaultSalesAccountCode` on SystemSettings)
2. **Backend** — `invoiceRoutes.ts` with line item builder logic + tests
3. **Backend** — Wire route in `index.ts`; extend `settingsRoutes.ts` for account code field
4. **Frontend** — Update `IProject` type
5. **Frontend** — `validateInvoiceData` pure function + unit tests
6. **Frontend** — `InvoiceCreationModal.tsx`
7. **Frontend** — Update `FinancialTab.tsx` — header buttons + modal wiring
8. **Frontend** — Settings → General — account code field
9. **ReleasesPanel** — add entry

---

## Open Questions

1. **Materials data shape** — how are material lines currently stored on a project for the Financial tab? Are they pulled from `ProjectFinancials`, a separate junction table, or directly from `Material` + project relations? Clarify before building the line item builder to ensure correct joins.

2. **Re-creating invoices** — if the user clicks Create Invoice again on a project that already has `xeroInvoiceId` set, should VisualOS (a) create a new invoice and overwrite the stored ID, (b) warn and require confirmation, or (c) always create fresh? Recommend: show a confirmation step in the modal ("A Xero invoice already exists for this project — create another?") and allow it, overwriting stored fields with the new invoice.

3. **Labour hours source** — should labour hours come from completed `TimeEntry` records only, or should estimated minutes be used as a fallback for tasks with no time logged? Recommendation: actual time entries first, fall back to `estimatedMinutes`, warn if both are zero.

4. **Discount handling** — the Financial tab has a discount (% or $). Should the discount be applied as a negative line item on the Xero invoice, or omitted (let the user apply it manually in Xero)? Recommendation: if a discount is set, add a negative line item labelled "Discount" so the Xero total matches the VisualOS total.

5. **Account code per line type** — currently a single `defaultSalesAccountCode` covers all lines. Should labour and materials use separate account codes (e.g. `200` for services, `260` for goods)? If yes, add two fields to SystemSettings instead of one.
