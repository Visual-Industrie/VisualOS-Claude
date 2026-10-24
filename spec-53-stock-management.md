# 53. Stock & Offcut Management

**Status:** ✅ Shipped October 2026 (see `complete.md`; low-stock alerts and the usage report moved to #55) **Priority:** Medium **Labels:** `feature`, `production`, `inventory`, `materials`, `deliverables` **Includes:** Deliverables on projects (standard signage sizes). Related to #6 (Materials Catalog & AI Recommendation, stays parked and untouched).

## Before building anything

**Audit the existing Deliverable data structure first.** A `Deliverable` model and tab already exist (type, dimensions, material, laminate, optional cutting diagram), and there is no numbered backlog item for deliverables, so the upgrade is folded into this item rather than split out. Before any code is written:

- Read the current `Deliverable` model in `schema.prisma` and the Deliverables tab component
- Check whether dimensions and material are free text or structured, and how existing rows would migrate
- Check how deliverables relate to project stages, tasks and the vinyl calculator so nothing is duplicated
- Write up what was found and the proposed migration, then confirm with Bren before building

The structured fields in this spec (width, height, material type, colour, thickness) are a proposal. If the audit shows a better shape, change the spec first.

### Audit findings (2026-10-24, agreed with Bren)

- **The Deliverables tab is unmounted.** `bec231a` replaced it with the Materials tab; `DeliverablesTab.tsx` / `EditDeliverableModal.tsx` still exist but nothing renders them. Deliverables are still read by the Shop Floor "Show deliverables" modal. **Decision: bring the Deliverables tab back** alongside Materials, as the place stock is picked.
- **No `project.tab.deliverables` keys exist.** Deliverable routes are `ensureAuthenticated` only, and `PATCH`/`DELETE` don't scope to the project. Add `project.tab.deliverables.view/.edit`, gate the routes, scope writes to `projectId` (404 otherwise).
- **Size is already structured.** `Deliverable.widthMm` / `heightMm` are `Float?` (manual or from a survey `PhotoDimension`). No conversion needed.
- **`Deliverable.materialType` means something else** (vinyl / print-media / substrate, free string). Leave it alone. Legacy `type`, `finishType`, `materialId`, `laminateId` are only read by Shop Floor; dropping them is backlogged, not part of #53. Live material lines are `DeliverableMaterial`.
- **No colour or thickness anywhere** (not on `Deliverable` or `Material`).
- **`Material` already has `isSheet`, `sheetWidthMm`, `sheetHeightMm`**: use them to pre-fill full-sheet size when a Xero material is linked.
- **No overlap**: the vinyl calculator stores its own JSON pieces; deliverables aren't linked to tasks or stages; `ProjectMaterial` is a costing line with no dimensions.
- **No taxonomy for ACM / Core flute** (`material_category` is the broad category). Add a new taxonomy type `stock_material`.

**Migration:** additive only. New nullable `Deliverable` columns: `substrateType`, `substrateColour`, `substrateThicknessMm`, `stockPieceId`. No existing rows rewritten.

## Goal

Replace the offcut spreadsheet with a stock layer inside Kit. Track every physical sheet and offcut of substrate (mainly ACM and core flute) with its exact dimensions, so that when a job needs a sign, Kit can suggest existing pieces that fit before anyone opens a new sheet.

Start small and simple. This is not the AI recommendation system from #6.

## Core concept

A **stock piece** is one physical rectangle of material. Pieces are either full sheets or offcuts, and both are the same thing: a piece with dimensions. Using a piece retires it and creates new offcut pieces from whatever is left.

The Xero materials list stays as the costing catalogue. Stock sits on top of it with an **optional** link.

## Data model

### `StockPiece`

- `id`
- `materialType` (e.g. ACM, Core flute, from the `stock_material` taxonomy)
- `colour` (e.g. White, Black, Yellow)
- `thicknessMm`
- `widthMm`, `heightMm`
- `isFullSheet` (boolean, for reporting and low-stock counts)
- `quantity` (Bren, 2026-10-24): **full sheets are a count, not individual rows.** One available row per spec (material, colour, thickness, size, Xero link) with a quantity; adding more of the same spec tops up that row. 0 keeps the spec with none on hand. Offcuts are one row per piece, always 1. Reserving a full sheet takes one off the stack and creates a separate reserved row (quantity 1), which then follows the normal used → offcuts flow. Stock take counts a stack (adjust the number) rather than confirming each sheet
- `materialId?` → `Material`. Optional. Standard black and white ACM link to a Xero material for costing. One-off special-order stock (e.g. a yellow sheet) has no link.
- `status`: `available` | `reserved` | `used` | `discarded`
- `reservedForProjectId?`, `reservedForDeliverableId?`
- `parentPieceId?` → `StockPiece`. Which piece this offcut was cut from (gives a trace-back history).
- `source`: `delivery` | `offcut` | `special_order` | `manual`
- `notes?`
- `createdAt`, `updatedAt`, `createdById`

All models take an `organisationId` FK when #32 lands.

### Deliverable link

- ~~`Deliverable.stockPieceId?`~~ Dropped at build (step 4): the chosen piece is `StockPiece.reservedForDeliverableId` (kept when the piece is used), so there aren't two links that could disagree
- Matching uses the existing `widthMm`, `heightMm` plus new `substrateType`, `substrateColour`, `substrateThicknessMm` (see audit; the legacy `materialType` is not reused). Standard signage first.

## Behaviour

### 1. Adding stock

- Add full sheets manually: material type, colour, thickness, size, quantity (creates N identical pieces). Optional Xero material link.
- Add a single freeform piece at any time, e.g. a special-order yellow sheet where the customer was charged for the full sheet and a small offcut remains.
- Linking to a Xero material is optional, never required.

### 2. Suggesting pieces for a deliverable

When a deliverable has dimensions and a material spec, Kit lists matching available pieces:

- Same material type, colour and thickness
- Piece is big enough, **allowing rotation** (a 1200 x 800 sign fits a 850 x 1250 piece)
- Ranked by least waste (smallest piece that fits first), so small offcuts get used up before full sheets are opened
- Shows several candidates, not one
- If nothing fits: clear "No suitable stock, you'll need a new sheet" message

Kit **suggests only**. Staff can pick any piece, including one not suggested. No layout is enforced on the shop floor.

### 3. Using a piece

- Picking a piece for a deliverable marks it `reserved` for that project.
- On completion of the cut, a **Record offcuts** step asks what's left: add zero or more rectangles (width x height). Defaults to one empty row since most leftovers are a single rectangle.
- On confirm: the source piece becomes `used`, and each recorded offcut becomes a new `available` piece with `parentPieceId` set, same material, colour and thickness inherited.
- Cutting two signs from one sheet and ending up with two rectangular offcuts is a single Record offcuts step with two rows.
- Reserved pieces can be released back to `available` if the job changes.

### Stock take

A tablet-first guided count that staff can run at any moment, in the garage, with big buttons and minimal typing. Ease of use is the most important requirement here.

- Start a stock take from the stock list page. Only one can be open at a time
- Walks through stock one material group at a time (material type, then colour and thickness), showing what Kit believes is there as large tappable cards with sizes
- Per piece: **Confirm** (one tap), **Adjust size** (number pad), **Missing** or **Discard**
- **Add piece** for anything physically present that Kit doesn't know about (freeform, same fields as manual add)
- Progress indicator per group, ability to pause and resume, and groups can be skipped
- Nothing changes until the end. A **review summary** lists differences: confirmed, resized, missing, added. Staff then tap **Apply** to commit them
- Applying writes the changes to stock and keeps the stock take as a record
- Reserved pieces are shown but flagged, so a piece set aside for a job isn't mistaken for a missing one
- Works offline-tolerant on a tablet: draft progress is kept locally until applied

Models:

- `StockTake`: `id`, `status` (`in_progress` | `applied` | `cancelled`), `startedById`, `startedAt`, `appliedAt?`, `notes?`
- `StockTakeLine`: `stockTakeId`, `stockPieceId?`, `outcome` (`confirmed` | `resized` | `missing` | `added` | `discarded`), `oldWidthMm?`, `oldHeightMm?`, `newWidthMm?`, `newHeightMm?`, plus the new piece's material fields for `added` lines

Permissions: `action.stock.take` (run a stock take). Applying changes also needs `action.stock.edit`.

### 4. Stock list page

- Filterable by material type, colour, thickness, status
- Sortable by size
- Quick add, edit, discard
- Tiny offcuts can be marked `discarded` when they are not worth keeping. Consider a configurable minimum size below which offcuts are not suggested.

### 5. Low-stock warnings (optional, off by default)

- `SystemSettings.stockLowAlertsEnabled` (default `false`), editable in Settings → Admin
- When on: a minimum full-sheet level per linked material, with a warning badge and optional task when stock falls below it
- Off for Visual Industrie (consignment stock, supplier replenishes monthly), but included because other businesses will need it. Design for multi-tenancy.

### 6. Usage report (nice-to-have)

- "Sheets used this month" per material, derived from pieces marked `used`
- Handy for the monthly consignment report to the supplier

## Out of scope (for now)

- Non-rectangular offcuts (L-shapes). Only the largest usable rectangle is tracked.
- Automatic nesting or cutting layout optimisation (that lives in #6)
- AI recommendations
- Vinyl rolls (the vinyl calculator already covers those)
- Barcode or label printing

## Permissions

Follow the registry rules in CLAUDE.md. New keys, all off by default for non-admin roles:

- `page.stock` (stock list page)
- `action.stock.edit` (add, edit, discard pieces), action.stock.take (run and apply a stock take)
- `project.tab.deliverables.view/.edit` (new: they don't exist yet, see audit) cover the revived tab and piece selection

## Tests for stock take

- `diffStockTake(pieces, lines)` builds the review summary of differences, with tests for every outcome type
- `applyStockTake(lines)` plans the stock changes, including that reserved pieces are never silently removed
- Component tests for the tablet flow: confirm, adjust, missing, add, review, apply

## Pure utils (each needs Vitest tests in the same PR)

- `pieceFits(piece, deliverable)`: true if it fits, allowing rotation
- `rankCandidates(pieces, deliverable)`: filter by material spec, then sort by least waste
- `planOffcuts(sourcePiece, offcutRects)`: validates the offcuts fit within the source and builds the new piece records
- `monthlyUsage(pieces, month)`: used-sheet counts per material

Test cases to include: exact fit, rotated fit, too small in one dimension, wrong colour, wrong thickness, reserved pieces excluded, offcuts larger than the source rejected.

## Suggested build order

1. **Audit the existing Deliverable structure** and agree the migration with Bren (no building until this is done)
2. `StockPiece` model, migration, CRUD routes, permission keys
3. Stock list page with manual add and edit
4. Deliverable upgrade: bring the tab back, add `project.tab.deliverables` keys, gate and project-scope the routes, add the `substrate*` fields
5. `pieceFits` and `rankCandidates`, then the suggestion panel on the deliverable
6. Reserve, then the Record offcuts flow
7. Stock take: tablet walk-through, review summary, apply
8. Optional: low-stock setting, usage report

## Acceptance criteria

- [x] Deliverable audit written up and agreed before any build starts
- [x] `StockPiece` model and migration
- [x] Manual add of full sheets (with quantity) and freeform pieces, Xero link optional
- [x] Stock list page with filters
- [x] Deliverables carry structured size and material spec
- [x] Suggestions rank least-waste first, allow rotation, show several candidates
- [x] Clear "order a new sheet" message when nothing fits
- [x] Record offcuts retires the source piece and creates offcut pieces with `parentPieceId`
- [ ] Low-stock alerts behind an admin setting, off by default
- [x] Stock take: tablet-friendly guided count with confirm, adjust, missing and add, a review summary, and nothing changed until Apply
- [x] Stock take history kept: who counted, when, and what changed
- [x] Permission keys added to registry and frontend types
- [x] Backend and frontend tests for all pure utils and routes
- [x] `ReleasesPanel.tsx` updated
