# Spec: Survey Photo Dimension Annotations

**Backlog item:** #38 (new)
**Labels:** `feature`, `survey`, `frontend`, `backend`
**Priority:** Medium
**Branch naming:** `feature/SurveyPhotoAnnotations`

---

## Overview

Staff can annotate survey photos by drawing lines that indicate where a set of recorded dimensions was measured. The annotation is purely a metadata overlay — it is never baked into the image file itself. Lines are stored as relative coordinates (0.0–1.0) so they render correctly at any display size. The UI supports pinch/scroll zoom and pan so staff can position annotation points accurately even on photos with fine detail.

---

## User Story

> As a signage installer reviewing a survey, I want to see where the width and height measurements were taken on the photo, so I can quickly understand the site context without re-reading notes.

---

## Scope

- **In scope:** Drawing dimension annotation lines on `SurveyPhoto` records; saving/loading/deleting annotations; zoom and pan on the annotator canvas.
- **Out of scope:** Annotating `CompletionPhoto` records (can be added later using the same component); freehand drawing; text-only labels; baking annotations into the image file.

---

## Data Model

### 1. Prisma schema change — `SurveyPhoto`

Add a nullable JSON field to store annotation lines:

```prisma
model SurveyPhoto {
  id          String   @id @default(uuid())
  surveyId    String
  survey      SiteSurvey @relation(fields: [surveyId], references: [id], onDelete: Cascade)
  filePath    String
  driveFileId String?
  driveUrl    String?
  notes       String?
  widthMm     Float?
  heightMm    Float?
  createdAt   DateTime @default(now())

  // NEW
  annotations Json?    @db.JsonB   // SurveyPhotoAnnotation[]
}
```

### 2. Annotation shape (TypeScript)

Defined in `projects-frontend/src/types/survey.ts` and mirrored as a Zod schema in the backend:

```ts
export interface SurveyPhotoAnnotation {
  id: string;           // client-generated uuid — used as React key and for deletion
  dimensionKey: 'width' | 'height' | string;  // extensible — maps to a label
  label: string;        // e.g. "Width", "Height", "Depth"
  value: string;        // e.g. "2400mm"
  x1: number;           // relative 0.0–1.0
  y1: number;
  x2: number;
  y2: number;
  colour: string;       // hex string — set at draw time from dimension colour map
}

export interface ISurveyPhoto {
  id: string;
  surveyId: string;
  filePath: string;
  driveFileId?: string;
  driveUrl?: string;
  notes?: string;
  widthMm?: number;
  heightMm?: number;
  annotations: SurveyPhotoAnnotation[];  // always an array, never null
  createdAt: string;
}
```

**Colour map** (defined as a constant in the annotator component):

```ts
const DIMENSION_COLOURS: Record<string, string> = {
  width:  '#378ADD',   // blue
  height: '#1D9E75',   // teal/green
  depth:  '#D85A30',   // coral/orange
};
// Any unknown dimensionKey falls back to '#888780' (gray)
```

---

## Migration

**Migration name:** `add_annotations_to_survey_photo`

```sql
ALTER TABLE "SurveyPhoto" ADD COLUMN "annotations" JSONB;
```

The column is nullable with no default. The backend normalises `null → []` on every read so the frontend always receives an array.

**Run locally:**
```bash
cd projects-backend
npx prisma migrate dev --name add_annotations_to_survey_photo
npx prisma generate
```

---

## Backend

### Route: `PATCH /api/survey-photos/:id/annotations`

**File:** `projects-backend/src/routes/surveyRoutes.ts`

**Auth:** `ensureAuthenticated` (same as all other survey routes)

**Request body:**
```json
{
  "annotations": [
    {
      "id": "abc-123",
      "dimensionKey": "width",
      "label": "Width",
      "value": "2400mm",
      "x1": 0.12,
      "y1": 0.45,
      "x2": 0.88,
      "y2": 0.45,
      "colour": "#378ADD"
    }
  ]
}
```

**Validation (Zod):**
- `annotations` must be an array (may be empty — clearing all annotations)
- Each item: `id` string, `dimensionKey` string, `label` string, `value` string, `x1/y1/x2/y2` numbers in `[0, 1]` range, `colour` string matching `/^#[0-9a-fA-F]{6}$/`
- Max 20 annotations per photo

**Response `200`:**
```json
{
  "id": "...",
  "annotations": [ /* full saved array */ ]
}
```

**Response `400`:** Zod validation failure — returns field-level errors.
**Response `404`:** Photo not found.

**Implementation notes:**
- Use `prisma.surveyPhoto.update({ where: { id }, data: { annotations } })`
- Return only `id` and `annotations` — no need to re-fetch the full photo

### Modify existing `GET /api/survey/:projectId` (or equivalent photo list endpoint)

Ensure `annotations` is included in the response and normalised:

```ts
const photos = await prisma.surveyPhoto.findMany({ where: { surveyId } });
return photos.map(p => ({ ...p, annotations: (p.annotations as SurveyPhotoAnnotation[]) ?? [] }));
```

---

## Frontend

### New component: `SurveyPhotoAnnotator`

**File:** `projects-frontend/src/components/Project/Survey/SurveyPhotoAnnotator.tsx`

A full-screen `Modal` (Mantine `size="100%"` or `fullScreen`) containing a zoomable, pannable canvas with annotation drawing capability.

#### Props

```ts
interface SurveyPhotoAnnotatorProps {
  photo: ISurveyPhoto;
  opened: boolean;
  onClose: () => void;
  onSaved: (updatedPhoto: ISurveyPhoto) => void;
}
```

#### Canvas behaviour

The annotator renders inside a `<canvas>` element sized to fill the modal content area. A `useEffect` hook measures the container `div` via `ResizeObserver` and sets the canvas `width`/`height` accordingly.

**Coordinate system:**
- All stored coordinates are relative (0.0–1.0) to the original image dimensions
- On render: `canvasX = rel.x * imgNaturalWidth * zoom + panOffsetX`
- On click: `rel.x = (canvasX - panOffsetX) / (imgNaturalWidth * zoom)`
- The image is always drawn to fill the canvas with `ctx.drawImage(img, 0, 0, canvasWidth, canvasHeight)` at zoom=1

**Zoom and pan:**
- Mouse scroll wheel: zoom in/out centred on cursor position
- Pinch gesture (two-finger touch): zoom centred on midpoint between fingers
- Pan mode: drag to pan (button toggle or hold Space)
- Draw mode: single tap/click for start point, second tap/click for end point
- Zoom bounds: `0.5×` minimum, `8×` maximum
- Zoom level displayed in toolbar as a percentage (e.g. `150%`)
- "Fit" button resets zoom to 1.0 and pan to origin

**Drawing flow:**
1. User selects a dimension from the dropdown (pre-populated from photo's `widthMm`/`heightMm` — see below)
2. User clicks/taps the canvas — a filled circle marker appears at the start point
3. A dashed preview line tracks the cursor/finger to the current position
4. User clicks/taps again — the line is committed, a label appears at the midpoint
5. Both endpoints show as small filled circles; the line has arrowheads at both ends
6. The new annotation is appended to local state; `isDirty` set to `true`

**Label rendering:**
- Drawn at the midpoint of the line
- Background: semi-transparent dark pill (`rgba(0,0,0,0.55)`, `border-radius: 4px`)
- Text: white, 12px, `label: value` format (e.g. `Width: 2400mm`)
- Uses `ctx.roundRect` for the pill (supported in all modern browsers; polyfill not required)

#### Dimension dropdown

Populated dynamically from the photo's existing dimension data:

```ts
const dimensionOptions = [
  photo.widthMm  && { key: 'width',  label: 'Width',  value: `${photo.widthMm}mm`,  colour: DIMENSION_COLOURS.width },
  photo.heightMm && { key: 'height', label: 'Height', value: `${photo.heightMm}mm`, colour: DIMENSION_COLOURS.height },
].filter(Boolean);
```

If the photo has no dimensions recorded yet, show an inline `Alert` explaining the user should add dimensions first (via the photo edit modal), with the draw tools disabled.

If only one dimension is present, auto-select it and hide the dropdown.

#### Toolbar layout (top of modal)

```
[−] [100%] [+] [Fit]   |   [Dimension: ▾]   [Draw] [Pan]   |   [Clear all]          [Save] [Cancel]
```

- Left group: zoom controls
- Middle group: dimension picker + mode toggle
- Right group: destructive action + save/cancel
- All buttons use Mantine `Button` with appropriate `variant` and `size="xs"`
- Save button shows a `Loader` while the PATCH request is in-flight

#### Annotation list (below canvas)

A small list of committed annotations with a colour-coded dot, label+value, and a remove button per item. Removing from the list also removes from canvas immediately (local state only — still requires Save to persist).

#### Save behaviour

On Save:
1. `PATCH /api/survey-photos/:id/annotations` with the current `annotations` array
2. On success: call `onSaved(updatedPhoto)` to update the parent's photo state; `isDirty = false`; show success toast via `notify.success`
3. On error: show error toast; do not close the modal
4. Unsaved changes (when `isDirty`) show a confirmation prompt if the user clicks Cancel or the modal close button — use Mantine's `useDisclosure` pattern with a small confirm modal rather than browser `confirm()`

### Modify: `SurveyTab.tsx`

**Add an Annotate button to each photo thumbnail.**

Currently each photo card in the thumbnail grid has a click handler that opens a lightbox. Add a secondary action button:

```tsx
<ActionIcon
  variant="light"
  size="sm"
  title="Annotate dimensions"
  onClick={(e) => { e.stopPropagation(); openAnnotator(photo); }}
>
  <IconRuler size={14} />
</ActionIcon>
```

Use Mantine's `IconRuler` (or `IconVectorBezier2` as fallback) from `@tabler/icons-react`.

If a photo already has at least one annotation, show a small badge/indicator on the thumbnail (e.g. a teal dot in the corner) so staff can see at a glance which photos are annotated.

**State additions to `SurveyTab`:**

```ts
const [annotatorPhoto, setAnnotatorPhoto] = useState<ISurveyPhoto | null>(null);

const handleAnnotationSaved = (updated: ISurveyPhoto) => {
  setPhotos(prev => prev.map(p => p.id === updated.id ? { ...p, annotations: updated.annotations } : p));
  setAnnotatorPhoto(null);
};
```

### New type additions: `types/survey.ts`

Add `SurveyPhotoAnnotation` and ensure `ISurveyPhoto` includes `annotations: SurveyPhotoAnnotation[]`.

---

## Unit Tests

### Backend (`surveyRoutes.test.ts`)

```ts
describe('PATCH /api/survey-photos/:id/annotations', () => {
  it('saves a valid annotation array', async () => { ... });
  it('returns 400 for out-of-range coordinates', async () => { ... });
  it('returns 400 for more than 20 annotations', async () => { ... });
  it('returns 400 for invalid colour format', async () => { ... });
  it('returns 404 for unknown photo id', async () => { ... });
  it('accepts an empty array (clears all annotations)', async () => { ... });
});
```

### Frontend (`SurveyPhotoAnnotator.test.tsx`)

```ts
describe('SurveyPhotoAnnotator', () => {
  it('renders disabled state when photo has no dimensions', () => { ... });
  it('auto-selects dimension when only one is present', () => { ... });
  it('adds annotation to list on second click', () => { ... });
  it('removes annotation from list when remove is clicked', () => { ... });
  it('calls onSaved with updated photo after successful PATCH', async () => { ... });
  it('shows confirm dialog when closing with unsaved changes', () => { ... });
});
```

---

## CLAUDE.md Updates

Add to the **Frontend components** section under `Project/Tabs/SurveyTab`:

> `SurveyPhotoAnnotator` — full-screen modal canvas for drawing dimension annotation lines on survey photos; zoom/pan/draw modes; annotations stored as relative coords in `SurveyPhoto.annotations` JSON field; dimensions pre-populated from photo's `widthMm`/`heightMm`

Add to **Data Models** under `SurveyPhoto`:

> `annotations` — nullable JSONB column storing an array of `SurveyPhotoAnnotation` objects (relative coords 0.0–1.0, colour hex, label, value); normalised to `[]` on read

---

## ReleasesPanel Entry

```
Survey photos — dimension annotations: draw lines directly on survey photos to show where measurements were taken. Tap to set start and end points; lines are labelled with the dimension value. Zoom in for precision. Annotations are saved as metadata — the image file is never modified.
```

---

## Acceptance Criteria

- [ ] `SurveyPhoto.annotations` JSONB column exists; migration applied cleanly
- [ ] `PATCH /api/survey-photos/:id/annotations` saves and returns annotation array
- [ ] Backend validates coordinate range, annotation count, and colour format
- [ ] Frontend `ISurveyPhoto` type includes `annotations: SurveyPhotoAnnotation[]`
- [ ] Annotator modal opens from each photo thumbnail via ruler icon button
- [ ] Dimension dropdown populated from photo's recorded `widthMm`/`heightMm`
- [ ] Alert shown (draw tools disabled) when photo has no recorded dimensions
- [ ] Draw mode: click/tap start → live preview line → click/tap end → line committed
- [ ] Lines render with arrowheads at both ends and a labelled pill at midpoint
- [ ] Scroll wheel and pinch-to-zoom both work; zoom range 0.5×–8×
- [ ] Pan works in Pan mode; Space key not required (nice to have)
- [ ] Annotations persisted correctly across page reload
- [ ] Annotated photos show a visual indicator (dot) on the thumbnail grid
- [ ] Unsaved changes prompt on close/cancel
- [ ] Backend and frontend unit tests written and passing
- [ ] ReleasesPanel entry added in same PR
- [ ] CLAUDE.md updated in same PR

---

## Open Questions

1. **Should the annotator also work on `CompletionPhoto` records?** The component is designed to be reusable — the only change would be a different save endpoint. Leaving out of scope for this PR but worth noting.
2. **Multiple annotation sets per photo?** Currently all annotations on a photo share one flat array. If a photo is re-surveyed and dimensions change, the old annotations stay. Consider adding an `archivedAt` field to annotations in future if this becomes an issue.
3. **Annotation line style for depth/custom dimensions?** Currently only `width` and `height` are sourced from the photo model. A future enhancement could allow arbitrary custom dimension labels typed in by the user.