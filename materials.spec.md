# VisualOS — Materials & Deliverables Feature Spec

**Status:** 🟡 Ready to Build  
**Priority:** Medium  
**Labels:** `feature`, `xero-api`, `materials`, `deliverables`  
**Last Updated:** February 25, 2026

---

## Overview

Add a Materials Catalog and Deliverables system to VisualOS. A `Deliverable` sits between a `Project` and its `Material` selections — since a single project may require signage, vehicle wraps, and apparel all at once, each with different material requirements.

Materials are synced from Xero (the source of truth for standard stock) but can also be created as free-form custom entries for one-off purchases. Custom materials can optionally be saved back to the database for reuse on future deliverables.

**No wizard in MVP.** Users manually select materials per deliverable. The wizard (guided questions per deliverable type) is deferred to a later phase to keep the UI fast and give users full control.

---

## Scope

- Prisma schema: `Material`, `Deliverable`, `DeliverableMaterial`
- Xero sync: bulk import of Xero inventory items → upsert locally as `Material` records
- `Project` model: add relation to `Deliverable`
- Backend REST routes: CRUD for deliverables and materials
- Frontend: Deliverables tab on Project detail page; Materials catalog page; material picker (Xero catalogue + free-form)
- Materials can be categorised manually (substrate, vinyl, laminate, etc.) since Xero does not categorise them

---

## What We Are NOT Doing (MVP)

- No wizard / guided questions per deliverable type (deferred to Phase 2)
- No AI-based material recommendations (parked)
- No cutting layout optimisation (parked)
- No inventory quantity tracking (future)
- No cost estimation (future)

---

## Background: Why Deliverables?

A project in VisualOS maps to a customer job. A single job often has multiple distinct components — for example:

- Storefront signage (ACM substrate + printed vinyl + laminate)
- Vehicle wrap on the client's van (conformable vinyl, no substrate)
- Branded staff apparel (DTF print on garments)

Each of these is a **Deliverable** — a discrete thing being produced. Each deliverable has its own type, its own material requirements, and potentially its own production workflow. Linking materials directly to the project would collapse all of this into one flat list. The `Deliverable` layer keeps things clean.

---

## Deliverable Types

The system must support adding new deliverable types in the future without schema changes. Store `type` as a `String` rather than a database enum.

**Initial types:**

| Type | Description |
|---|---|
| `signage` | Rigid or flat signs — ACM, dibond, corflute, foam PVC, etc. |
| `vehicle` | Vehicle wraps or partial graphics |
| `apparel` | T-shirts, polos, uniforms, branded clothing |
| `print-outsource` | Print jobs sent to external print shops for management |
| `other` | Anything that doesn't fit the above |

---

## Material Sources

Materials come from two sources:

### 1. Xero-Synced Materials
- Pulled from Xero inventory items via the Xero API
- Stored with `xeroItemId` (Xero's `ItemID`) for identification
- Xero is source of truth — fields are overwritten on each sync
- These represent standard stock materials used regularly

### 2. Custom Materials
- Created free-form in VisualOS when a job requires a one-off purchase
- No `xeroItemId` — flagged as `isCustom: true`
- Can optionally be **saved to the materials database** for reuse in future deliverables
- Example: a specific DTF supplier's product, or a specialty vinyl for a one-off job

---

## Data Model

### `Material`

```prisma
model Material {
  id          Int     @id @default(autoincrement())

  // Xero identity — null for custom materials
  xeroItemId  String? @unique  // Xero ItemID

  // Core fields
  name        String
  code        String?           // Item code / SKU
  description String?
  isCustom    Boolean @default(false)  // true = created in VisualOS, not from Xero

  // Categorisation (manual — Xero does not categorise materials)
  category    String?  // substrate | vinyl | laminate | apparel | dtf | ink | other

  // Supplier info (useful for custom/one-off materials)
  supplierName  String?
  supplierCode  String?  // Supplier's own product code

  // Xero metadata
  xeroUpdatedDateUtc DateTime?

  // Relations
  deliverableMaterials DeliverableMaterial[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### `Deliverable`

```prisma
model Deliverable {
  id          Int     @id @default(autoincrement())

  // Which project this belongs to
  project     Project @relation(fields: [projectId], references: [id], onDelete: Cascade)
  projectId   Int

  // What type of thing we're making
  // String not enum — allows new types to be added without a migration
  type        String   // signage | vehicle | apparel | print-outsource | other
  name        String   // e.g. "Main Fascia Sign", "Company Van Wrap", "Staff Polos"
  description String?
  notes       String?
  sortOrder   Int      @default(0)  // Controls display order within a project

  // Relations
  materials   DeliverableMaterial[]

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

### `DeliverableMaterial`

Junction table linking a deliverable to its materials. Allows multiple materials per deliverable and captures role and notes about how each material is used.

```prisma
model DeliverableMaterial {
  id            Int         @id @default(autoincrement())

  deliverable   Deliverable @relation(fields: [deliverableId], references: [id], onDelete: Cascade)
  deliverableId Int

  material      Material    @relation(fields: [materialId], references: [id])
  materialId    Int

  // How this material is used in this deliverable
  role          String?   // substrate | vinyl | laminate | print | garment | other
  notes         String?

  createdAt     DateTime  @default(now())
}
```

### Updated `Project`

Add the following relation to the existing `Project` model:

```prisma
model Project {
  // ... existing fields ...

  deliverables  Deliverable[]
}
```

---

## Backend Implementation

### New API Routes

```
GET    /api/materials                           — List all materials (searchable, filterable by category)
POST   /api/materials                           — Create a new custom material
PATCH  /api/materials/:id                       — Update a material (category, supplier info, etc.)
POST   /api/materials/sync                      — Trigger bulk sync from Xero inventory (admin/internal)

GET    /api/projects/:id/deliverables           — List deliverables for a project (includes materials)
POST   /api/projects/:id/deliverables           — Create a deliverable on a project
PATCH  /api/projects/:id/deliverables/:did      — Update a deliverable (name, type, notes, sortOrder)
DELETE /api/projects/:id/deliverables/:did      — Delete a deliverable (cascades to DeliverableMaterial)

POST   /api/deliverables/:id/materials          — Add a material to a deliverable
PATCH  /api/deliverables/:id/materials/:dmid    — Update role/notes on a DeliverableMaterial
DELETE /api/deliverables/:id/materials/:dmid    — Remove a material from a deliverable
```

No `DELETE /api/materials` — materials are not deleted, just archived or left unused.

### Xero Sync (`POST /api/materials/sync`)

```typescript
// Paginate through all Xero Items (page=1, pageSize=100, repeat until empty)
// Skip items where IsPurchased = false (sales-only items like labour)
// Call upsertMaterialFromXero() for each
// Return { synced: number, skipped: number, errors: number }
// Guard with internal auth (admin session or secret header)
```

**Upsert logic:**

```typescript
async function upsertMaterialFromXero(xeroItem: Item): Promise<void> {
  const data = {
    xeroItemId:          xeroItem.itemID!,
    name:                xeroItem.name!,
    code:                xeroItem.code ?? null,
    description:         xeroItem.description ?? null,
    isCustom:            false,
    xeroUpdatedDateUtc:  xeroItem.updatedDateUTC ? new Date(xeroItem.updatedDateUTC) : null,
  };

  await prisma.material.upsert({
    where:  { xeroItemId: data.xeroItemId },
    update: data,
    create: data,
  });
}
```

Note: `category`, `supplierName`, `supplierCode` are intentionally excluded from the upsert — they are set manually in VisualOS and should not be overwritten by Xero syncs.

### Create Custom Material (`POST /api/materials`)

```typescript
// Request body: { name, code?, description?, category?, supplierName?, supplierCode? }
// Validates that name is present
// Creates Material with isCustom: true, xeroItemId: null
// Returns the new Material record
```

### Add Material to Deliverable (`POST /api/deliverables/:id/materials`)

```typescript
// Request body: { materialId, role?, notes? }
// Optionally: { customMaterial: { name, code?, supplierName?, supplierCode? }, saveToDatabase?, role?, notes? }
//   — if customMaterial provided, creates the Material first with isCustom: true, then links it
//   — saveToDatabase flag has no effect on current behaviour (material is always saved) but is used
//     by the frontend to decide whether to show it in the catalogue search going forward
// Returns the updated DeliverableMaterial with material included
```

---

## Frontend Implementation

### Routes

| Route | Component | Description |
|---|---|---|
| `/materials` | `MaterialsPage` | Searchable, filterable catalog of all materials |
| `/projects/:id` (tab) | `DeliverablesTab` | Deliverables list + material picker per deliverable |

### Materials Catalog Page (`/materials`)

- Search input filtering by name or code
- Filter chips/select for category (substrate, vinyl, laminate, apparel, dtf, ink, other)
- Table columns: name, code, category badge, supplier, Xero badge (if synced), Custom badge (if custom)
- "Sync from Xero" button → `POST /api/materials/sync`, shows progress toast
- "Add Custom Material" button → opens `CreateMaterialModal`
- Clicking a row opens a detail drawer/modal: name, description, category (editable), supplier info (editable), list of projects/deliverables this material has been used on

### Create / Edit Material Modal

| Field | Required | Notes |
|---|---|---|
| Name | ✅ | |
| Code / SKU | No | |
| Description | No | |
| Category | No | Select: substrate, vinyl, laminate, apparel, dtf, ink, other |
| Supplier name | No | |
| Supplier code | No | |

### Deliverables Tab (on Project detail page)

Add a **Deliverables** tab alongside the existing Notes, Tasks, Survey tabs.

- List of deliverables, each as a card or accordion row
- Each card shows: deliverable name, type badge, list of attached materials (name + role badge)
- "Add Deliverable" button → opens inline form or modal
- Deliverables are sortable (drag handle or up/down arrows, updates `sortOrder`)
- Each deliverable has Edit and Delete buttons

**Deliverable form fields:**

| Field | Required |
|---|---|
| Name | ✅ |
| Type | ✅ (select: signage, vehicle, apparel, print-outsource, other) |
| Description | No |
| Notes | No |

### Material Picker (per deliverable)

Triggered by "Add Material" on a deliverable card. Opens a modal with two modes via a Mantine `SegmentedControl`:

**From Catalogue tab:**
- Search input → debounced query to `GET /api/materials?search=...`
- Results list: name, code, category badge
- On select: optionally set Role (substrate, vinyl, laminate, print, garment, other) and Notes
- Save button

**Custom / One-Off tab:**
- Fields: Name (required), Supplier, Code, Notes, Role
- Checkbox: "Save this material to the catalog for future use"
  - If checked: creates a `Material` record with `isCustom: true` and links it
  - If unchecked: creates the material record anyway (needed for the FK) but it won't appear prominently in catalogue searches by default — consider an `isArchived` or `catalogVisible` flag in future
- Save button

---

## Material Categories

Since Xero doesn't categorise materials, categories are set manually in VisualOS. Synced materials will have `category: null` until manually assigned.

| Value | Label |
|---|---|
| `substrate` | Substrate |
| `vinyl` | Vinyl |
| `laminate` | Laminate |
| `apparel` | Apparel / Garment |
| `dtf` | DTF Print |
| `ink` | Ink |
| `other` | Other |

---

## Phase 2 — Wizard (Deferred)

Once the MVP is in use and patterns emerge, a guided wizard can be layered on top of deliverable creation. The wizard would ask targeted questions based on deliverable type and pre-populate the material picker accordingly. It does **not** change the underlying data model — it just guides the user to the right selections faster.

Example questions per deliverable type:

**All types:**
- Printed, cut vinyl, or both?
- Indoor or outdoor?

**Signage:**
- Substrate type? (ACM, Dibond, Foam PVC, Corflute, Acrylic...)
- Is this a lightbox? (affects vinyl choice)
- Does it need laminate?

**Vehicle wraps:**
- Flat graphics only, or does vinyl need to conform to curves?
- Full wrap or partial?

**Apparel:**
- Garment type? (T-shirt, polo, hoodie...)
- Print method? (DTF, screen print, embroidery, heat transfer)

**Print outsource:**
- Media type and finish
- Which external supplier?

---

## Migration Plan

Execute in this order:

1. Prisma migration: add `Material`, `Deliverable`, `DeliverableMaterial` models
2. Prisma migration: add `deliverables` relation to `Project`
3. Deploy backend with new routes
4. Run `POST /api/materials/sync` to populate materials from Xero inventory
5. Manually categorise key materials via the Materials catalog page
6. Deploy frontend (Deliverables tab + Materials catalog page)

---

## Testing Checklist

### Backend Unit Tests
- [ ] `upsertMaterialFromXero()` — creates new material with correct fields
- [ ] `upsertMaterialFromXero()` — updates existing material on re-sync, does not overwrite `category`
- [ ] `upsertMaterialFromXero()` — idempotent (run twice, no duplicates)
- [ ] `POST /api/materials` — creates custom material with `isCustom: true`, no `xeroItemId`
- [ ] `POST /api/projects/:id/deliverables` — creates deliverable linked to correct project
- [ ] `PATCH /api/projects/:id/deliverables/:did` — updates name, type, notes
- [ ] `DELETE /api/projects/:id/deliverables/:did` — cascades to `DeliverableMaterial`
- [ ] `POST /api/deliverables/:id/materials` — links existing material to deliverable
- [ ] `POST /api/deliverables/:id/materials` with `customMaterial` — creates material and links it
- [ ] Bulk sync handles Xero pagination correctly (mock multiple pages of 100)
- [ ] Bulk sync skips non-purchased items

### Frontend Unit Tests
- [ ] Deliverables tab shows empty state when project has no deliverables
- [ ] Material picker catalogue search debounces correctly
- [ ] "Save to catalog" checkbox creates material visible in future searches
- [ ] Deliverable type badge renders correctly for all types
- [ ] Materials catalog page filters by category correctly

---

## Acceptance Criteria

- [ ] Prisma migration for all new tables passes cleanly
- [ ] `Material.xeroItemId` has unique index
- [ ] Xero sync paginates and upserts all purchased inventory items
- [ ] Synced materials do not overwrite manually-set `category`, `supplierName`, `supplierCode`
- [ ] Custom materials created with `isCustom: true` are visually distinct in the UI
- [ ] A project can have multiple deliverables of different types
- [ ] Deliverables can be reordered within a project
- [ ] Each deliverable can have multiple materials attached with a role
- [ ] Material picker supports both catalogue selection and free-form custom entry
- [ ] Free-form custom materials can be saved to the catalogue for reuse
- [ ] Materials catalog page is searchable and filterable by category
- [ ] Deliverables tab is functional on the project detail page
- [ ] No TypeScript errors, ESLint clean per project config
- [ ] Unit tests written for upsert logic and all new route handlers

---

## Open Questions

- **Which Xero item types to sync?** Confirm whether to sync all items or only `IsPurchased = true`. Sales-only items like labour codes should probably be excluded.
- **Xero OAuth scope:** Confirm `accounting.inventory` read scope is granted. May require re-authorisation if not already present.
- **Category on sync:** Leave `category` null after sync (manual assignment) or attempt auto-categorisation based on name/code patterns?
- **`catalogVisible` flag:** Should "unsaved" custom materials be hidden from catalogue search, or is a simple `isCustom` badge sufficient to distinguish them?
- **Deliverable ordering:** `sortOrder` field included in schema — confirm if drag-and-drop reordering is wanted in MVP or can be deferred.