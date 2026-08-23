# Survey Photo Multiple Dimensions

**Status:** 📋 Planned  
**Priority:** Medium  
**Labels:** `feature`, `survey`, `ux`

---

## Problem

Currently one `SurveyPhoto` record stores a single `width` and `height`. When a survey photo contains multiple measurable elements (e.g., five windows in a row), users must upload the same photo five times to capture each measurement — wasteful and tedious in the field.

---

## Solution

Add a `PhotoDimension` child model to `SurveyPhoto` so a single photo can carry multiple width/height/description entries. The original `width` and `height` scalar fields on `SurveyPhoto` are deprecated (kept nullable during migration, removed in a later cleanup).

---

## Prisma Schema Changes

```prisma
model SurveyPhoto {
  id          Int              @id @default(autoincrement())
  surveyId    Int
  survey      SiteSurvey       @relation(fields: [surveyId], references: [id])
  driveFileId String
  driveName   String
  driveUrl    String
  notes       String?
  width       Int?             // DEPRECATED — kept nullable for migration safety
  height      Int?             // DEPRECATED — kept nullable for migration safety
  order       Int              @default(0)
  dimensions  PhotoDimension[]
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt
}

model PhotoDimension {
  id          Int          @id @default(autoincrement())
  photoId     Int
  photo       SurveyPhoto  @relation(fields: [photoId], references: [id], onDelete: Cascade)
  label       String       // e.g. "Window 1", "Left panel"
  width       Int?         // mm
  height      Int?         // mm
  notes       String?
  order       Int          @default(0)
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
}
```

---

## Migration

```
npx prisma migrate dev --name add_photo_dimensions
```

- Adds `PhotoDimension` table with cascade delete
- Leaves existing `width`/`height` on `SurveyPhoto` as nullable (no data loss)
- Optionally: a one-time script migrates any existing `width`/`height` values into a single `PhotoDimension` row per photo with `label = "Dimensions"`

---

## API Routes

### Existing — extend to include dimensions

```
GET  /api/projects/:id/survey
```
Response: include `dimensions` array nested under each `SurveyPhoto`.

### New dimension CRUD

```
POST   /api/survey-photos/:photoId/dimensions
PATCH  /api/survey-photos/:photoId/dimensions/:dimensionId
DELETE /api/survey-photos/:photoId/dimensions/:dimensionId
```

**POST / PATCH body:**
```json
{
  "label": "Window 2",
  "width": 1500,
  "height": 900,
  "notes": "Fixed frame, no opening"
}
```

---

## TypeScript Types

```typescript
export interface IPhotoDimension {
  id: number;
  photoId: number;
  label: string;
  width?: number;
  height?: number;
  notes?: string;
  order: number;
}

// Extend existing ISurveyPhoto
export interface ISurveyPhoto {
  id: number;
  driveFileId: string;
  driveName: string;
  driveUrl: string;
  notes?: string;
  width?: number;   // deprecated
  height?: number;  // deprecated
  order: number;
  dimensions: IPhotoDimension[];
}
```

---

## Frontend — SurveyTab Changes

### Photo preview modal (post-upload / edit)

The existing photo detail modal gains a **Dimensions** section below the notes field:

- List of existing dimensions (label, W × H, notes) with edit/delete inline
- **+ Add Dimension** button opens a small inline form:
  - `Label` text input (e.g. "Window 1")
  - `Width (mm)` number input
  - `Height (mm)` number input
  - `Notes` text input (optional)
  - Save / Cancel

### Thumbnail grid

- Show dimension count badge on thumbnail: e.g. `3 dims` if photo has 3 entries
- On hover / tap: tooltip listing labels (e.g. "Window 1, Window 2, Cornice")

### Lightbox

- Show full dimensions list below the photo in a clean table:

```
| Label      | Width (mm) | Height (mm) | Notes              |
|------------|------------|-------------|--------------------|
| Window 1   | 1200       | 800         |                    |
| Window 2   | 1500       | 900         | Fixed frame        |
| Cornice    | 3600       | 150         | Wrap around corner |
```

---

## Components

| Component | Change |
|---|---|
| `SurveyTab.tsx` | Pass `dimensions` through to photo modal; show dim count badge on thumbnails |
| `PhotoUploadModal.tsx` | Add Dimensions section with inline add/edit/delete |
| `PhotoLightbox.tsx` | Render dimensions table below photo |
| New: `PhotoDimensionForm.tsx` | Reusable inline add/edit form for a single dimension |
| New: `PhotoDimensionList.tsx` | List with edit/delete controls |

---

## Acceptance Criteria

- [ ] `PhotoDimension` model created with cascade delete from `SurveyPhoto`
- [ ] Existing `width`/`height` on `SurveyPhoto` nullable (no migration data loss)
- [ ] `GET /api/projects/:id/survey` includes `dimensions[]` on each photo
- [ ] `POST`, `PATCH`, `DELETE` dimension routes working and tested
- [ ] Photo modal allows adding/editing/deleting multiple dimensions
- [ ] Thumbnail grid shows dimension count badge
- [ ] Lightbox renders dimensions table
- [ ] Unit tests for dimension CRUD routes
- [ ] TypeScript types updated (`IPhotoDimension`, `ISurveyPhoto`)

---

## Open Questions

- Should `label` be required? (Suggested: yes — forces useful naming like "Window 1" vs blank)
- Do we want an `order` drag-to-reorder on dimensions, or just creation order? (Suggested: creation order for now)
- Migrate existing `width`/`height` data into `PhotoDimension` rows automatically, or leave as-is and remove in cleanup? (Suggested: auto-migrate with label "Dimensions", then cleanup in a later PR)

---

## Out of Scope (Future)

- Drawing / annotation overlay on the photo to visually tag which dimension corresponds to which element (see Backlog #28 — Survey Annotator)
- Exporting dimensions to a cutting docket or PDF

---

*Added: March 2026*
