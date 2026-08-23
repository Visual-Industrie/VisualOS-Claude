# Mileage Tracking — Feature Spec
**VisualOS Backlog Item #38**
**Status:** 📋 Planned
**Priority:** Medium
**Labels:** `feature`, `backend`, `frontend`, `mileage`, `vehicles`

---

## Overview

Staff can log vehicle mileage for any trip related to work — whether tied to a specific project, a specific task, or a general multi-stop run. Entries support both odometer readings and direct kilometre entry (for backfilling via Google Maps). Mileage is tracked per vehicle and per staff member, with a date picker defaulting to today to allow retroactive entry.

Future: mileage entries will feed into a reimbursement report, potentially exportable to Xero.

---

## Requirements

- Log mileage entries per vehicle (free-form vehicle name, not staff-assigned)
- Support odometer start/end **or** direct kilometre entry — not required to use both
- Date picker defaults to today but allows any past date (for retroactive entry)
- Entries can be:
  - Completely standalone (no project, no task) — for multi-stop or general runs
  - Linked to a project only
  - Linked to both a project and a task
- Vehicle name is free-form with autocomplete from previously used vehicle names
- Notes field for describing what the trip was for (especially important for standalone entries)
- Separate from timesheets — mileage does not appear on timesheet pages
- Staff member is recorded automatically from the logged-in session

---

## Data Model

### `MileageEntry` (new Prisma model)

```prisma
model MileageEntry {
  id              String       @id @default(cuid())

  // Staff
  staffMemberId   String       @db.Uuid
  staffMember     StaffMember  @relation(fields: [staffMemberId], references: [id], onDelete: Cascade)

  // Vehicle (free-form)
  vehicleName     String

  // Mileage — either odometer OR direct kilometres (both optional, at least one required at app level)
  odometerStart   Int?         // km reading at start of trip
  odometerEnd     Int?         // km reading at end of trip
  totalKilometres Int?         // direct entry if odometer not available

  // Optional links
  projectId       String?
  project         Project?     @relation(fields: [projectId], references: [id], onDelete: SetNull)

  taskId          String?
  task            Task?        @relation(fields: [taskId], references: [id], onDelete: SetNull)

  // Metadata
  date            DateTime     // Date of trip — defaults to today in UI, can backfill
  notes           String?      // What was the trip for?

  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt
}
```

**Also add to existing models:**
```prisma
// On StaffMember:
mileageEntries  MileageEntry[]

// On Project:
mileageEntries  MileageEntry[]

// On Task:
mileageEntries  MileageEntry[]
```

**Computed field (app layer, not DB):**
- `calculatedKilometres` = `odometerEnd - odometerStart` if both present, otherwise `totalKilometres`

---

## Validation Rules (app layer)

- Either (`odometerStart` AND `odometerEnd`) OR `totalKilometres` must be provided
- If odometer: `odometerEnd` must be greater than `odometerStart`
- `vehicleName` required
- `date` required (defaults to today)
- `staffMemberId` required (auto-resolved from session)
- `taskId` requires `projectId` to also be set (a task always belongs to a project)

---

## Backend API Routes

All routes under `/api/mileage`. Protected by `ensureAuthenticated`.

### `GET /api/mileage`
Returns mileage entries for the current staff member (resolved via email → StaffMember).

Query params:
- `dateFrom` / `dateTo` — ISO date strings for range filter
- `projectId` — filter by project
- `vehicleName` — filter by vehicle

### `GET /api/mileage/all` _(admin only)_
Returns all staff mileage entries. Same query params as above, plus:
- `staffMemberId` — filter by staff member

### `GET /api/projects/:id/mileage`
Returns all mileage entries linked to a project (all staff).

### `POST /api/mileage`
Create a new mileage entry.

Request body:
```json
{
  "vehicleName": "Work Van",
  "odometerStart": 45200,
  "odometerEnd": 45287,
  "totalKilometres": null,
  "projectId": "clxyz123",
  "taskId": null,
  "date": "2026-04-09",
  "notes": "Site measure at Smith St + drop off samples"
}
```

### `PATCH /api/mileage/:id`
Update an existing entry. Staff can only edit their own; admins can edit any.

### `DELETE /api/mileage/:id`
Delete an entry. Staff can only delete their own; admins can delete any.

### `GET /api/mileage/vehicles` _(autocomplete)_
Returns a distinct list of vehicle names previously used by the current staff member (for autocomplete in the form).

---

## Frontend

### Log Mileage Modal (`MileageEntryModal`)

Accessible from:
- A "Log Mileage" button in the sidebar action stack (alongside "Take a Message")
- The Mileage tab on a project detail page (pre-fills `projectId`)
- The Shop Floor task card (pre-fills `projectId` and `taskId`) — future enhancement

**Form fields:**
| Field | Component | Notes |
|---|---|---|
| Vehicle | `Autocomplete` | Free-form, suggests previously used names |
| Date | `DatePickerInput` | Defaults to today, allows backfill |
| Odometer Start | `NumberInput` | Optional — show/hide based on entry mode |
| Odometer End | `NumberInput` | Optional — show/hide based on entry mode |
| Total Kilometres | `NumberInput` | Optional — show/hide based on entry mode |
| Project | `Select` | Optional — searchable project list |
| Task | `Select` | Optional — filtered to selected project's tasks |
| Notes | `Textarea` | Optional but encouraged for standalone entries |

**Entry mode toggle:** A `SegmentedControl` or pair of radio buttons switches between:
- **Odometer** — shows Start + End fields; calculated km shown as `= X km`
- **Direct entry** — shows Total Kilometres field only

### My Mileage Page (`/my-mileage`)

Similar layout to `MyTimesheet.page.tsx`:
- Entries grouped by date, most recent first
- Date range filter (`DatePickerInput`)
- Vehicle filter (`Select` — distinct vehicles from entries)
- Each row shows: date, vehicle, km, project (if linked), notes
- Add / Edit / Delete actions
- Total km summary for the filtered period

### Admin Mileage Page (`/mileage`) _(admin only)_

Same as My Mileage but across all staff:
- Additional "Staff Member" filter
- Totals per staff member at the bottom
- Future: rate per km column + reimbursement total

### Project Mileage Tab

New tab on the project detail page (alongside Schedule, Timesheets, etc.):
- Lists all mileage entries linked to this project (all staff)
- Shows staff name, vehicle, km, date, notes
- "Log Mileage" button pre-fills `projectId`
- Total km for the project at the top

---

## Reimbursement (Future — not built in Phase 1)

When reimbursement is needed:
- Add `ratePerKm Decimal?` to a `VehicleRate` config table (or a simple settings field)
- Calculate `reimbursableAmount = calculatedKm * ratePerKm`
- Export to CSV or push to Xero as an expense claim
- IRD mileage rate (NZ) currently 73c/km for the first 14,000km — store as a system setting so it can be updated

---

## Navigation & Access

- Sidebar: "Log Mileage" quick-action button (all staff)
- Sidebar nav: "My Mileage" link (all staff)
- Settings → Admin nav: "Mileage" link (admin only)
- Project detail: "Mileage" tab (all staff)

---

## Releases Entry (to add in `ReleasesPanel.tsx` on ship)

```
Mileage Tracking — Staff can now log vehicle mileage for any work-related trip.
Entries can be standalone, linked to a project, or linked to a specific task.
Supports odometer readings or direct kilometre entry for backfilling.
Vehicle names are free-form with autocomplete. Accessible from the sidebar,
project detail pages, and a dedicated My Mileage page.
```

---

## Implementation Order (suggested for Claude Code)

1. Prisma migration — `MileageEntry` model + relations on `StaffMember`, `Project`, `Task`
2. Backend routes — `mileageRoutes.ts` with full CRUD + vehicle autocomplete endpoint
3. Unit tests for all new routes
4. `MileageEntryModal` component — form with entry mode toggle
5. `My Mileage` page (`/my-mileage`)
6. Sidebar "Log Mileage" button
7. Project Mileage tab
8. Admin Mileage page (`/mileage`)
9. Frontend unit tests for new components/hooks
10. Releases panel update

---

## Open Questions

- Should the Shop Floor task card have a "Log Mileage" shortcut, or is the sidebar button sufficient for Phase 1?
- Should vehicle names be shared across staff (org-wide autocomplete) or per-staff-member only?
- IRD rate: store as a system setting from day one, or add later when reimbursement is built?

---

*Last updated: April 9, 2026*
*Spec author: Claude (via voice session with Bren)*
