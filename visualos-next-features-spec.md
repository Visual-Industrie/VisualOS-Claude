# VisualOS — Next Features Spec for Claude Code

**Prepared:** February 2026  
**Status:** Ready to Build  
**Scope:** Survey Tab v2, Completion Photos, Contact/Project linking improvements, Brand Assets tab

---

## Context

VisualOS is a signage project management tool built with:
- **Backend:** Express + PostgreSQL + Prisma (TypeScript)
- **Frontend:** React + Mantine UI (Vite)
- **Deployment:** Docker on Synology NAS
- **Integrations:** Xero, Google Workspace (Gmail, Drive, Calendar)
- **Code style:** Prettier config enforces `singleQuote`, `trailingComma: 'es5'`, `printWidth: 100`
- **Linting:** ESLint with TypeScript strict mode, Mantine config

Contacts are already fully built and linked to Projects via `contactId` FK. Projects have `xeroStatus`, `status`, `productionStatus` etc.

---

## Feature 1 — Google Drive Folder Picker for Projects

### Overview

Each project in VisualOS should be linked to its corresponding job folder on Google Drive. The existing Drive folder structure is:

```
[Letter] / [Client Name] / [Year] / [Job Number]

e.g. C / Acme Corp / 2026 / 0042
```

Job numbers are sequential per client per year (reset each year). Folders are created and managed manually in Google Drive — VisualOS just needs to pick and link them.

### Schema Change

```prisma
model Project {
  // ... existing fields ...
  driveFolderId   String?   // Google Drive folder ID for this job folder
}
```

### Backend

**New route:**
```
PATCH /api/projects/:id/drive-folder
Body: { driveFolderId: string }
```
- Validates the folder is accessible via Google Drive API
- Updates `project.driveFolderId`
- Returns updated project

### Frontend — Folder Picker

- Add a **"Link Drive Folder"** button to the project detail page (suitable location: project header or settings area)
- On click, open Google Drive Picker using the **Google Picker API** (iframe-based, official Google UI)
- Picker should be configured to:
  - Show folder selection only (not files)
  - If the project's contact already has a `driveFolderId` set (see Feature 4), default the picker to open at that client folder
  - Otherwise open at Drive root
- On folder selected → call `PATCH /api/projects/:id/drive-folder` with the folder ID
- Show the linked folder name + a "Open in Drive" link once linked
- Show "Change Folder" button if already linked

### Google Picker API Setup

```javascript
// Load the Google Picker API
// Use: https://apis.google.com/js/api.js
// Required scope: https://www.googleapis.com/auth/drive.readonly (for picking)
// Required scope: https://www.googleapis.com/auth/drive.file (for uploading)

const picker = new google.picker.PickerBuilder()
  .addView(new google.picker.DocsView(google.picker.ViewId.FOLDERS)
    .setSelectFolderEnabled(true)
    .setParent(clientFolderId ?? 'root')) // default to client folder if available
  .setOAuthToken(accessToken)
  .setCallback(pickerCallback)
  .build();
```

### Acceptance Criteria
- [ ] `driveFolderId` added to Project schema, migration created
- [ ] `PATCH /api/projects/:id/drive-folder` route implemented
- [ ] Google Picker opens correctly and shows folders only
- [ ] Selected folder ID saved to project
- [ ] Linked folder name displayed with "Open in Drive" link
- [ ] If contact has a Drive folder linked, picker defaults to that folder
- [ ] Unit test: PATCH route updates driveFolderId correctly

---

## Feature 2 — Survey Tab: Upload Photos to Google Drive

### Overview

Currently survey photos are uploaded to local server storage. Replace this with Google Drive upload — photos go into a `Links` subfolder within the project's linked job folder.

### Folder Structure

```
[Job Folder] (stored as project.driveFolderId)
├── Links/              ← survey photos go here (Illustrator imports from this folder)
└── Completion Photos/  ← completion photos go here (see Feature 3)
```

### Backend Changes

**Update survey photo upload route:**
- Check project has `driveFolderId` set — if not, return error asking user to link a Drive folder first
- Check if `Links` subfolder exists inside `driveFolderId` — if not, create it
- Upload photo to `Links` folder via Google Drive API
- Store `driveFileId` and `driveFileUrl` on the `SurveyPhoto` record instead of local path

**Schema change:**
```prisma
model SurveyPhoto {
  // Replace local path fields with:
  driveFileId   String?   // Google Drive file ID
  driveFileUrl  String?   // Direct view/thumbnail URL
  // Keep: width, height, notes, createdAt etc.
}
```

**Helper function (shared, reusable):**
```typescript
async function getOrCreateSubfolder(
  parentFolderId: string,
  folderName: string,
  accessToken: string
): Promise<string> // returns subfolder ID
```

### Frontend Changes

- Survey tab: if project has no `driveFolderId`, show a warning banner: "Link a Drive folder to this project before uploading photos"
- Photo upload UX remains the same (existing modal with width/height/notes)
- Photos display as thumbnails using Drive thumbnail URLs
- Remove any references to local file paths/URLs

### Acceptance Criteria
- [ ] Survey photos upload to `Links/` subfolder in project Drive folder
- [ ] `getOrCreateSubfolder` helper implemented and reusable
- [ ] SurveyPhoto schema updated with Drive fields
- [ ] Warning shown if no Drive folder linked
- [ ] Thumbnails display correctly from Drive URLs
- [ ] Unit test: `getOrCreateSubfolder` creates folder if missing, returns existing if present

---

## Feature 3 — Completion Photos Tab

### Overview

Add a new **Completion Photos** tab to the project detail page. These photos are stored in Google Drive in a `Completion Photos` subfolder alongside `Links`.

### Folder Structure (same job folder)

```
[Job Folder]
├── Links/
│   └── [survey photos]
└── Completion Photos/
    └── [completion photos]
```

### Schema

```prisma
model CompletionPhoto {
  id            Int      @id @default(autoincrement())
  projectId     Int
  project       Project  @relation(fields: [projectId], references: [id])

  driveFileId   String
  driveFileUrl  String
  notes         String?

  createdAt     DateTime @default(now())
}
```

### Backend Routes

```
GET    /api/projects/:id/completion-photos     — list all completion photos
POST   /api/projects/:id/completion-photos     — upload photo to Drive + save record
DELETE /api/projects/:id/completion-photos/:photoId — delete from Drive + remove record
```

Upload logic:
- Check project has `driveFolderId`
- `getOrCreateSubfolder(driveFolderId, 'Completion Photos', accessToken)` (reuse helper from Feature 2)
- Upload file to that subfolder
- Save `CompletionPhoto` record

### Frontend

- New **Completion Photos** tab on project detail page (alongside Survey, Notes, Tasks etc.)
- If no Drive folder linked: show warning banner same as Survey tab
- Upload button → file picker → uploads with optional notes field
- Photo grid with thumbnails (same style as Survey tab)
- Delete button per photo (with confirmation)
- Empty state if no photos yet

### Acceptance Criteria
- [ ] `CompletionPhoto` schema + migration
- [ ] All three backend routes implemented
- [ ] Photos upload to `Completion Photos/` subfolder in Drive
- [ ] Reuses `getOrCreateSubfolder` helper
- [ ] New tab visible on project detail page
- [ ] Upload, display, delete all working
- [ ] Warning if no Drive folder linked
- [ ] Unit tests for upload and delete routes

---

## Feature 4 — Contact Detail: Brand Assets Tab

### Overview

Add a new **Brand Assets** tab to the Contact detail page. This captures brand colors, logo assets, and design notes for each client. Logos are stored on Google Drive — the contact links to their client-level Drive folder (one level above the job folders).

### Drive Folder Hierarchy Context

```
C / Acme Corp /          ← Contact links to THIS folder (client level)
    2026 /
        0042 /           ← Project links to THIS folder (job level)
            Links/
            Completion Photos/
```

### Schema Changes

```prisma
model Contact {
  // ... existing fields ...
  driveFolderId   String?   // Google Drive folder ID for the client folder
  brandAssets     BrandAssets?
}

model BrandAssets {
  id          Int      @id @default(autoincrement())
  contactId   Int      @unique
  contact     Contact  @relation(fields: [contactId], references: [id])

  brandColors String?  // Rich text / plain text field for brand color info
  notes       String?  // Rich text field for general brand notes

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

### Backend Routes

```
GET   /api/contacts/:id/brand-assets        — get brand assets for contact
PUT   /api/contacts/:id/brand-assets        — upsert brand assets (colors, notes)
PATCH /api/contacts/:id/drive-folder        — set driveFolderId on contact
```

### Frontend — Brand Assets Tab

**Tab contents:**
- **Drive Folder:** "Link Client Folder" button → Google Drive Picker (same iframe picker as projects, but picks the client-level folder). Shows folder name + "Open in Drive" link once linked.
- **Brand Colors:** Rich text editor (same component used for project descriptions) — reasonable size, for capturing hex codes, color names, usage notes etc.
- **Brand Notes:** Rich text editor for general design/brand guidelines
- **Save** button

**Folder picker behaviour:**
- Opens at Drive root (no parent folder to default to at contact level)
- Folder selection only

### Acceptance Criteria
- [ ] `driveFolderId` added to Contact schema
- [ ] `BrandAssets` model + migration
- [ ] GET and PUT routes for brand assets
- [ ] PATCH route for contact drive folder
- [ ] Brand Assets tab visible on Contact detail page
- [ ] Drive folder picker works and saves folder ID
- [ ] Rich text editor for brand colors and notes
- [ ] Save/update working correctly
- [ ] Unit tests for brand assets routes

---

## Feature 5 — Project View: Contact Linking Improvements

### Overview

Two small but important UX improvements to make it easy to navigate between a project and its linked contact.

### 5a — Clickable Contact Name (Bidirectional Navigation)

Currently you can navigate Contact → Projects (via Projects tab on contact detail).  
Add the reverse: on the project detail page, the client/contact name should be a clickable link that navigates to `/contacts/:contactId`.

**Implementation:**
- Find where `project.contact?.name` is rendered in the project detail page
- Wrap it in a React Router `Link` component pointing to `/contacts/${project.contactId}`
- Style consistently with other links in the app (Mantine `Anchor` component)

### 5b — Contact Details on Project View

Display the main contact's phone and email address somewhere visible on the project detail page (e.g. in the project header or a small info card near the contact name).

**Data available:** `project.contact.emailAddress` and `project.contact.phones` (array — use `phoneType === 'DEFAULT'` or first available)

**Display:**
- Email address with `mailto:` link
- Primary phone number
- Keep it compact — not a full contact card, just the essentials

**Backend:** Ensure project queries include `contact: { include: { phones: true } }` so phone data is available.

### Acceptance Criteria
- [ ] Contact name on project detail page is a clickable link to `/contacts/:id`
- [ ] Email address displayed on project detail page
- [ ] Primary phone number displayed on project detail page
- [ ] Project API query includes contact phones
- [ ] No TypeScript errors

---

## Shared Utilities to Build

These should be built once and reused across features:

### Google Drive Helper (`/lib/googleDrive.ts`)

```typescript
// Upload a file to a specific Drive folder
uploadFileToDrive(folderId: string, file: Buffer, filename: string, mimeType: string, accessToken: string): Promise<{ fileId: string, webViewLink: string, thumbnailLink: string }>

// Get or create a named subfolder within a parent folder
getOrCreateSubfolder(parentFolderId: string, folderName: string, accessToken: string): Promise<string> // returns folder ID

// Get folder metadata (name etc.) for display
getDriveFolderMeta(folderId: string, accessToken: string): Promise<{ id: string, name: string, webViewLink: string }>

// Delete a file from Drive
deleteFileFromDrive(fileId: string, accessToken: string): Promise<void>
```

### Google Picker Component (`/components/DrivePickerButton.tsx`)

Reusable React component that:
- Accepts `onFolderSelected: (folderId: string, folderName: string) => void`
- Accepts optional `defaultParentFolderId?: string` (to open picker at a specific folder)
- Loads Google Picker API
- Opens picker configured for folder selection only
- Calls callback on selection

---

## Implementation Order (Suggested)

1. **Prisma migrations** — `driveFolderId` on Project + Contact, `CompletionPhoto` model, `BrandAssets` model
2. **Google Drive helper** (`/lib/googleDrive.ts`) + unit tests
3. **DrivePickerButton component** (reusable, used in Features 1, 4)
4. **Feature 1** — Project Drive folder picker
5. **Feature 2** — Survey photos to Drive
6. **Feature 3** — Completion Photos tab
7. **Feature 4** — Contact Brand Assets tab
8. **Feature 5** — Contact linking improvements (quick wins, can do anytime)

---

## Notes & Reminders

- Use `studio@vil.nz` OAuth tokens for all Google API calls (same pattern as Gmail + Calendar integrations)
- Google OAuth scopes needed: `https://www.googleapis.com/auth/drive.file` (upload/create) + `https://www.googleapis.com/auth/drive.readonly` (picker/browse) — may require re-authorisation
- All Prisma migrations: use `npx prisma migrate dev --name <migration_name>` in dev, `npx prisma migrate deploy` in production
- Code style: single quotes, trailing commas (es5), 100 char print width
- Write unit tests alongside each feature — backend with Vitest, frontend with React Testing Library
- Encourage test coverage for: Drive helper functions, all new API routes, React components (at minimum: render without crash, key interactions)
- The `getOrCreateSubfolder` helper must be idempotent — calling it twice should not create duplicate folders
