# Spec #51 — Brand Guideline Files on Contact Brand Assets

**Status:** 📋 Specced — ready to build
**Priority:** Medium
**Labels:** `feature`, `contacts`, `google-drive`, `brand-assets`
**Branches:** `feature/BrandGuidelineFiles` (projects-backend + projects-frontend)
**Source:** Daily Log, 20 Oct 2026 — "Add file picker to brand assets for any guideline files. Should allow multiple files and let the user set a name or description for each file. Same as design approval, it's just a link to G Drive."

---

## Goal

On a contact's **Brand Assets** tab, staff can link any number of Google Drive files (brand guidelines, logo packs, colour specs, font files) and give each a name and description. Files stay in Drive; Kit stores a link, exactly like design files.

### Non-goals

- Uploading files into Drive from Kit (pick existing files only)
- Copying or caching file contents
- Showing brand files on the project page or Shop Floor (possible follow-up, see Notes)

---

## Data model

```prisma
model ContactBrandFile {
  id            String   @id @default(uuid())
  contactId     String
  contact       Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  driveFileId   String
  driveFileName String   // name in Drive at the time it was picked
  mimeType      String?
  iconUrl       String?  // Picker iconUrl, for the list
  displayName   String   // staff-set name, defaults to the Drive name without extension
  description   String?
  sortOrder     Int      @default(0)
  addedById     Int?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  @@unique([contactId, driveFileId])
  @@index([contactId])
}
```

- `Contact` gains `brandFiles ContactBrandFile[]`
- Match `contactId`'s type to the existing `Contact.id`
- Ready to take `organisationId` under #32
- Links are built on the fly (`https://drive.google.com/file/d/{id}/view`), not stored

Migration: hand-written SQL → `psql` → `prisma migrate resolve --applied` (session-table drift workaround in `CLAUDE.md`).

---

## Backend

### Routes — `contactBrandFileRoutes.ts`, mounted at `/api/contacts/:contactId/brand-files`

| Method | Path | Permission | Notes |
|---|---|---|---|
| `GET` | `/` | Brand Assets `.view` | Sorted by `sortOrder`, then `createdAt` |
| `POST` | `/` | Brand Assets `.edit` | Body `{ files: [{ driveFileId, driveFileName, mimeType?, iconUrl?, displayName, description? }] }` — bulk, one request per picker session. Files already linked are skipped and returned in `skipped` (no 409). Max 20 per request. |
| `PATCH` | `/:id` | Brand Assets `.edit` | `displayName`, `description` |
| `POST` | `/reorder` | Brand Assets `.edit` | `{ ids: string[] }` |
| `DELETE` | `/:id` | Brand Assets `.edit` | Unlinks only — never touches the Drive file |

- Use the existing Brand Assets tab key pair from `permissions/registry.ts` (check the exact name, e.g. `contact.tab.brand-assets.view/.edit`). No new keys.
- Zod validation: `displayName` 1–120 chars, trimmed; `description` ≤ 500 chars; `driveFileId` non-empty, Drive-id characters only.
- 404 when the contact doesn't exist; 404 when the file id belongs to a different contact.

### Pure util — `utils/brandFiles.ts`

| Function | Does |
|---|---|
| `defaultDisplayName(fileName)` | Strips the extension, turns `_`/`-` runs into spaces, trims; falls back to the raw name |
| `partitionNewFiles(picked, existingDriveIds)` | `{ toCreate, skipped }`, also de-duping within the picked list |
| `nextSortOrder(existing)` | Appends new files after the current last one |

---

## Frontend

### Brand Assets tab — new "Guidelines & files" section

Below the existing brand Drive folder picker.

- **Add files** button → `DriveFilePickerButton` (the design-file picker), with:
  - multi-select enabled (`google.picker.Feature.MULTISELECT_ENABLED`) — check whether `DriveFilePickerButton` already supports a `multiple` prop; add one if not, default `false` so design approval is unchanged
  - starts in the contact's brand folder when `contact.driveFolderId` is set, otherwise the system base folder (same smart-default rule as the folder pickers)
  - all file types allowed (PDF, AI, images, fonts, zips)
- After picking → **Name your files** modal: one row per picked file with `displayName` (pre-filled via `defaultDisplayName`) and an optional description. Already-linked files show greyed with "Already added". **Save** sends one bulk `POST`.
- **File list:** file-type icon, display name (opens in Drive in a new tab), description underneath, drag-to-reorder (`@hello-pangea/dnd`), edit (pencil → same fields in a small modal) and remove (confirm: "Remove the link? The file stays in Drive.").
- Empty state: "No guideline files yet — add brand guidelines, logo packs or colour specs from Drive."
- Edit controls hidden without the `.edit` key (`<Can>`); the list still shows with `.view`.

Shared pure helpers duplicated on the frontend where needed (`defaultDisplayName`) get their own frontend test — or expose the default name from the backend response instead and skip the duplicate.

---

## Tests

### Backend (Vitest)

| File | Covers |
|---|---|
| `utils/brandFiles.test.ts` | Extension stripping (`Brand_Guidelines_v2.pdf`, no extension, dotfiles, multiple dots); partition skips existing and in-batch duplicates; sort order append |
| `routes/contactBrandFileRoutes.test.ts` | Permission gating (view vs edit); bulk create with skipped duplicates; validation errors; PATCH and DELETE scoped to the contact; reorder; cascade on contact delete |

### Frontend (Vitest)

- Name-your-files modal: pre-filled names, already-added rows disabled, Save payload shape
- File list: edit/remove hidden without `.edit`; empty state

---

## Acceptance criteria

- [ ] `ContactBrandFile` model + migration
- [ ] Brand-files routes with bulk create, edit, reorder, unlink
- [ ] `brandFiles` pure util + tests
- [ ] Multi-select support on `DriveFilePickerButton` without changing design approval
- [ ] Guidelines & files section on Brand Assets: add, name, describe, reorder, edit, remove
- [ ] Backend + frontend tests as above
- [ ] `CLAUDE.md` updated (routes, models, Brand Assets tab)
- [ ] `ReleasesPanel.tsx` updated in the same PR

## Notes

- Drive access: picking works today because we hold `drive.readonly`. If #41 later drops it in favour of `drive.file`, picked files are still covered as long as `setAppId` is in place — no extra work here, but test brand files alongside the other pickers when #41 is done.
- Follow-up idea (not in scope): show the customer's brand files on the project page and the Shop Floor design preview, so designers and installers can reach them from the job.
