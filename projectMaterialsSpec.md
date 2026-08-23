# Spec: Project Materials Tab

**Replaces:** Deliverables tab (which gets removed)
**Branch:** `feature/MaterialsTab`
**Priority:** High

---

## Overview

A simple tabular view on each project for tracking materials — what you planned to use (estimate) and what you actually used (actuals). Works with known materials from the catalogue or one-off entries typed in directly.

---

## Data Model

### New model: `ProjectMaterial`

```prisma
model ProjectMaterial {
  id          String   @id @default(cuid())
  projectId   String
  project     Project  @relation(fields: [projectId], references: [id], onDelete: Cascade)
  materialId  String?  // null = one-off, not from catalogue
  material    Material? @relation(fields: [materialId], references: [id])
  name        String   // always stored — copied from Material.name or typed freeform
  description String?
  estimatedQty Float?
  actualQty   Float?
  unit        String?  // e.g. "m", "sheets", "rolls" — freeform for now
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

Add relation on `Project`:
```prisma
projectMaterials ProjectMaterial[]
```

Add relation on `Material`:
```prisma
projectMaterials ProjectMaterial[]
```

### Migration name
`add_project_materials`

### Existing `Deliverable` model
Leave the model in place for now but remove the tab from the frontend. This avoids a destructive migration while the feature is being tested. Add a `backlog.md` note to drop the table in a future cleanup.

---

## Backend

### Route file: `materialUsageRoutes.ts` (new)

Mount at: `/api/projects/:projectId/materials`

#### `GET /api/projects/:projectId/materials`
- Returns all `ProjectMaterial` rows for the project, ordered by `createdAt ASC`
- Include `material { name, category }` relation (for display)

#### `POST /api/projects/:projectId/materials`
Body:
```json
{
  "name": "3M 1080 Matte Black",
  "materialId": "clxxx...",   // optional — null for freeform
  "description": "Hood and roof",
  "estimatedQty": 5.5,
  "actualQty": null,
  "unit": "m"
}
```
- `name` required
- Create and return the new row

#### `PATCH /api/projects/:projectId/materials/:id`
Body: any subset of `{ name, description, estimatedQty, actualQty, unit }`
- Returns updated row

#### `DELETE /api/projects/:projectId/materials/:id`
- Returns 204

### Autocomplete endpoint: `GET /api/materials/search?q=xxx`
- Already exists on `materialRoutes` (check) or add it
- Returns `[{ id, name, category }]` — max 20 results, case-insensitive `name` ILIKE
- Used by the frontend autocomplete

### Mount in `index.ts`
```ts
import materialUsageRoutes from './routes/materialUsageRoutes';
// ...
app.use('/api/projects', ensureAuthenticated, materialUsageRoutes);
```

---

## Frontend

### Tab placement
Remove the **Deliverables** tab from `Project.tsx`. Add a **Materials** tab in its place, same position.

### Component: `MaterialsTab.tsx`

Location: `src/components/Project/Tabs/MaterialsTab.tsx`

#### Layout
- Header row: "Materials" title left, "Add material" button right (small, `variant="light"`)
- Table with columns: **Material** | **Description** | **Est. qty** | **Actual qty** | **Unit** | *(actions)*
- Inline editing — clicking a row's field opens it for edit directly in the table cell (like a simple editable table)
- Or: a modal for add, inline edit for actuals (simpler to build — go with this approach)
- Empty state: "No materials added yet" with an "Add material" button

#### Add / Edit Modal: `MaterialModal.tsx`

Fields:
1. **Material** — `Autocomplete` component
   - User types; fetches `GET /api/materials/search?q=xxx` after 300ms debounce
   - If they select a known material: populate `name`, store `materialId`
   - If they type something not in the list and press Enter or blur: store as freeform `name`, `materialId = null`
   - Label: "Material name"
2. **Description** — `TextInput`, optional, placeholder "e.g. Hood and roof panels"
3. **Est. qty** — `NumberInput`, optional, min 0, step 0.5, placeholder "0"
4. **Actual qty** — `NumberInput`, optional, min 0, step 0.5, placeholder "0"
5. **Unit** — `TextInput`, optional, short, placeholder "m / sheets / rolls"

Submit: POST (add) or PATCH (edit).

#### Table UX
- Each row has an edit icon (pencil) and delete icon (trash) at the right
- Delete: inline confirmation (red trash icon → confirm click), no modal needed
- Totals row at the bottom: sum of Est. qty and Actual qty (only when both columns have at least one value)
- If estimated and actual are both set, show a small coloured indicator:
  - Actual > estimated → amber (used more than expected)
  - Actual ≤ estimated → green (on track)

#### Types: `IProjectMaterial`
```ts
// src/types/material.ts  (add to existing file or create if absent)
export interface IProjectMaterial {
  id: string;
  projectId: string;
  materialId?: string | null;
  name: string;
  description?: string | null;
  estimatedQty?: number | null;
  actualQty?: number | null;
  unit?: string | null;
  createdAt: string;
  updatedAt: string;
}
```

---

## Tests

### Backend (`materialUsageRoutes.test.ts`)
- POST creates a row with `materialId = null` for freeform entries
- POST with a valid `materialId` links correctly
- PATCH updates `actualQty` only
- DELETE returns 204
- GET returns rows ordered by `createdAt`

### Frontend (`MaterialsTab.test.tsx`)
- Renders empty state when no materials
- "Add material" button opens modal
- Freeform name (no materialId) is accepted
- Totals row appears only when values exist
- Over-estimate indicator renders when `actualQty > estimatedQty`

---

## Releases entry

```
Materials tab — replaces Deliverables with a simpler estimate vs actuals view per project.
Add any material (from the catalogue or freeform), set an estimated quantity, and fill in actuals
as the job progresses. Over-run indicator flags when you've used more than planned.
```

---

## What stays / what goes

| Item | Action |
|---|---|
| `Deliverable` model | Keep in DB, hide tab in frontend |
| `DeliverablesTab.tsx` | Remove from `Project.tsx` tab list (keep file for now) |
| `Material` catalogue | Unchanged — used as autocomplete source |
| `materialRoutes.ts` | Extend with `/search` endpoint if not present |
| `ProjectMaterial` model | New |
| `materialUsageRoutes.ts` | New |
| `MaterialsTab.tsx` | New |
| `MaterialModal.tsx` | New |

---

## Open questions (resolve before handing to Claude Code)

1. Does `GET /api/materials/search?q=` already exist, or does it need to be added to `materialRoutes.ts`?
2. Should unit be a taxonomy-driven dropdown (e.g. `material_unit` type) or freeform for now? **Recommendation: freeform for v1, add taxonomy later if needed.**
3. Is there existing data in `Deliverable` that should be migrated to `ProjectMaterial`? If so, a one-time migration script is needed.