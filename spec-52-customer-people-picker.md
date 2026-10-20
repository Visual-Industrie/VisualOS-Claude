# Spec #52 — Customer People, Contact Picker + Design Approval Email

**Status:** 📋 Specced — ready to build
**Priority:** Medium
**Labels:** `feature`, `contacts`, `xero`, `ux`, `database`
**Branches:** `feature/CustomerPeople` (projects-backend), `feature/CustomerPeople` (projects-frontend)
**Source:** Chat, 21 Oct 2026. Schema checked against `prisma/schema.prisma` (Contact/ProjectContact ids are `Int`).
**Sequencing:** #50 has shipped, so the contact tab strip is settled; this adds to the People tab

---

> Two parts, buildable independently: **Part A** customer people + project contact picker; **Part B** design approval email full-text editor + quote link.

# Part A — Customer People + Project Contact Picker

## Goal

Stop retyping project contacts on every job.

- Every person added to a project is saved against the **customer** (`Contact`) and can be reused on later jobs.
- The project contacts picker offers the customer's **Xero billing contact** and every saved **Kit person** in one searchable list.
- New projects get the Xero billing contact as **primary** by default. About 90% of customers are small businesses with a single contact, so most jobs need no action.

### Xero is read-only for people

Xero's API doesn't allow contact updates from Kit, so nothing in this feature writes to Xero.

- Xero keeps only the **billing contact**: the person on the Xero contact record.
- Any extra people, for example several staff at a local council, live **only in Kit**.

---

## Data model

### New: `CustomerPerson`

A person who belongs to a customer.

```prisma
model CustomerPerson {
  id              Int       @id @default(autoincrement())
  contactId       Int                         // the customer
  contact         Contact   @relation(fields: [contactId], references: [id], onDelete: Cascade)
  firstName       String
  lastName        String?
  email           String?
  emailNormalised String?                     // lower-cased + trimmed, for matching
  phone           String?                     // as entered
  phoneE164       String?                     // via normalisePhone(), for matching + SMS
  jobTitle        String?
  source          String    @default("kit")   // 'kit' | 'xero_billing'
  isArchived      Boolean   @default(false)
  createdByUserId Int?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  projectContacts ProjectContact[]

  @@index([contactId])
  @@index([contactId, emailNormalised])
}
```

- **At most one `xero_billing` row per customer.** Enforce this in code: a partial unique index fights Prisma drift, so use an upsert keyed on `(contactId, source='xero_billing')`.
- Takes an `organisationId` under #32.

### Changed: `ProjectContact`

- Add `customerPersonId String?` (FK → `CustomerPerson`, `onDelete: Restrict`). Archive people; never delete one that is linked to a job.
- `isPrimary` stays on `ProjectContact`, because primary is a per-job flag.
- Today `ProjectContact` holds its own `firstName`, `lastName`, `email`, `phone` and `xeroContactId`. Keep those columns for one release as a read fallback; every read path resolves through `customerPerson` first. Dropping them is a later cleanup (backlogged as a follow-up, same pattern as #14).
- **Editing a person updates every job they're on.** That's intended: the link is the source of truth.

> **Billing contact source:** `Contact.firstName` / `lastName` / `emailAddress` (already synced from Xero), with the phone taken from the contact's `ContactPhone` rows (mobile first, then default; reuse `sortContactPhones` from #50). No new Xero API calls.

---

## Xero billing person

The billing contact appears in Kit as a `CustomerPerson` with `source = 'xero_billing'`.

- **Created lazily.** Kit creates the row the first time it's needed: when a project defaults to it, when it's picked, or when the backfill runs. It is never created as a side effect of a GET request.
- **Read-only in Kit.** The People tab shows it with a "Xero" badge and an "Edit in Xero" link, with no edit or archive.
- **Refreshed on contact sync.** The Xero contact webhook and the manual contact sync update its name, email and phone through `ensureXeroBillingPerson()`.
- **Removal.** If Xero's billing contact is blanked, the Kit row is kept. It is not archived automatically, because it may be linked to old jobs.
- **Existing Xero `ContactPerson` rows.** If any are synced, the picker lists them as Xero people too. Picking one creates a `kit` person from its details, using the same matching rules. No other change to `ContactPerson`.

---

## Matching and de-duplication

Pure util `utils/customerPeople.ts`, unit-tested:

| Function | Does |
|---|---|
| `normaliseEmail(email)` | Trim + lower-case; empty → null |
| `findMatchingPerson(people, candidate)` | Returns an existing non-archived person for the same customer when the email matches (case-insensitive). If the candidate has no email, it matches on normalised full name **plus** `phoneE164`. Otherwise it returns null. Name alone never matches. |
| `billingPersonFromContact(contact)` | Builds the `xero_billing` person fields from the synced Xero contact; null if there's no name and no email |
| `buildPickerOptions(kitPeople, billingPerson, xeroContactPersons, alreadyOnProject)` | Merged list for the picker: billing first, then Kit people A–Z, then any Xero `ContactPerson`s not already matched. Excludes archived people and people already on the project. Each option is tagged `xero` or `kit`. |

Adding or picking someone who matches an existing person **reuses** that person instead of creating a duplicate.

---

## API

All routes enforce permissions server-side. Check the actual key names in `permissions/registry.ts`.

**Customer people:** `contact.tab.people.view` / `.edit`
- `GET /api/contacts/:contactId/people?includeArchived=` — Kit people + the billing person (if it exists yet) + synced Xero `ContactPerson`s, tagged by source
- `POST /api/contacts/:contactId/people` — create a Kit person (runs `findMatchingPerson` and returns 409 with the existing person on a match, so the UI can offer "Use existing")
- `PATCH /api/contacts/:contactId/people/:personId` — edit; 409 on `xero_billing`
- `POST /api/contacts/:contactId/people/:personId/archive` and `/restore` — 409 on `xero_billing`

**Project contacts:** existing `projectContactRoutes` (`/api/projects/:id/contacts`), existing project permission
- `GET` — unchanged shape, now resolved through `customerPerson`
- `POST` body accepts one of:
  - `{ customerPersonId, isPrimary? }` — link an existing person
  - `{ xeroBilling: true, isPrimary? }` — ensure the billing person exists, then link it
  - `{ xeroContactPersonId, isPrimary? }` — create or match a Kit person from that Xero person, then link it
  - `{ newPerson: { firstName, lastName?, email?, phone?, jobTitle? }, isPrimary? }` — match or create a Kit person on the project's customer, then link it
- Linking a person who is already on the project → 409
- Setting `isPrimary` clears it on the project's other contacts (keep the current behaviour)
- `DELETE` unlinks from the project only; the person stays on the customer
- `GET /api/projects/:id/contacts/options` — `buildPickerOptions` output for the picker

### Default primary on new projects

When a project is created with a customer and has no project contacts, link the Xero billing person as primary using `ensureXeroBillingPerson()`. This applies to every creation path: New Project, the enquiry form, quote requests, the chatbot `createLeadFromEnquiry`, and Xero project import.

- Skip silently if the customer has no billing name or email.
- Put this in one helper, `attachDefaultPrimaryContact(prisma, projectId)`, called from each path. Don't inline it in each path.

### Notifications

- `resolveNotificationRecipient` and the design approval send read name and email through `customerPerson` (falling back to the legacy columns).
- Resolution order is unchanged.
- Add `phoneE164` to the resolved recipient so #29 and SMS can use it later.

---

## Backfill (one-off)

`POST /api/admin/backfill-customer-people`, gated on `settings.tab.admin.edit`. It is a temporary endpoint, per the NAS migration pattern, and is **removed after it runs in prod**.

1. For every existing `ProjectContact` without a `customerPersonId` on a project that has a customer, run `findMatchingPerson` against that customer's people. Create a Kit person or reuse a match, then link it.
2. For every project with a customer and **no** project contacts, run `attachDefaultPrimaryContact`.
3. **Idempotent:** a second run changes nothing.
4. Returns counts: created, matched, linked, defaulted, skipped (with reasons, such as no customer).

The decision logic lives in a pure function, `planBackfill(projectContacts, peopleByCustomer)`, so it can be tested without the DB.

---

## Frontend

### `PersonPicker` (`components/People/PersonPicker.tsx`)

- Searchable Mantine `Combobox`, with options grouped **Xero** and **Kit people** and a badge on each
- Shows name, email and phone so two "Sarah"s can be told apart
- The last option is always **"+ Add new person"**. It opens an inline form (first name required; last name, email, phone, job title optional) with no "save for later?" checkbox, because new people are always saved to the customer
- On a 409 match, it shows "Sarah Smith (sarah@…) already exists — use her?" with a **Use existing** button
- Disabled with a hint when the project has no customer linked

### `ProjectContactsPanel`

- Replace the free-text add form with `PersonPicker`
- Primary toggle unchanged
- The remove action says "Remove from this job", which is honest that the person stays on the customer
- Editing a Kit person from here edits the person (all jobs), and the modal says so. Xero billing person: no edit, "Edit in Xero" link

### Contact page — People tab

- Sections: **Xero billing contact** (read-only, badge, Edit in Xero), **Xero people** (existing synced `ContactPerson`s, read-only), **Kit people** (add / edit / archive, "Show archived" toggle)
- Each Kit person shows the jobs they're on (count + link), so archiving is an informed choice
- Gated on `contact.tab.people.view`; actions on `.edit`

### Types

`ICustomerPerson`, `IPersonOption` (`{ kind: 'xero_billing' | 'xero_person' | 'kit', … }`), and `IProjectContact` gains `customerPerson`.

---

## Tests

Every new pure function and route gets a test in the same PR.

**Backend (Vitest)**

| File | Covers |
|---|---|
| `utils/customerPeople.test.ts` | Email normalising; match by email (case/whitespace); name + phone fallback; name alone never matches; archived people ignored; billing person from a contact with missing name or email; picker option order, exclusions and tags |
| `utils/backfillCustomerPeople.test.ts` | `planBackfill`: creates vs reuses; de-dupes across several projects for the same customer; skips projects with no customer; defaults primary only when a project has no contacts; second run is a no-op |
| `routes/customerPeopleRoutes.test.ts` | CRUD; 409 on duplicate with existing person returned; 409 on editing or archiving `xero_billing`; permission 403s |
| `routes/projectContactRoutes.test.ts` | Each POST body shape; already-on-project 409; primary flag clears the others; DELETE leaves the person on the customer |
| `utils/attachDefaultPrimaryContact.test.ts` | Links billing as primary; skips when contacts exist or the billing contact is blank; idempotent |
| Contact sync | Webhook / manual sync refreshes the `xero_billing` person; blanked billing contact leaves the row alone |

**Frontend (Vitest + happy-dom)**

| File | Covers |
|---|---|
| `PersonPicker.test.tsx` | Groups and badges render; search filters; "+ Add new person" opens the form; 409 shows Use existing; disabled with no customer |
| `ProjectContactsPanel.test.tsx` | Adding via the picker; primary toggle; remove wording; no edit on the Xero billing person |
| `utils/personDisplay.test.ts` (if added) | Option label formatting |

---

## Acceptance criteria

- [ ] `CustomerPerson` model; `ProjectContact.customerPersonId`; migration (manual SQL + `migrate resolve` if the session-table drift bites)
- [ ] Pure utils and routes above, with tests
- [ ] Picker on the project contacts panel: Xero billing + Kit people in one list, add new inline, match → use existing
- [ ] Every person added to a job is saved to the customer and offered on that customer's next job
- [ ] New projects (all creation paths) default the Xero billing contact as primary
- [ ] People tab: Xero billing (read-only) + Kit people (add / edit / archive)
- [ ] Notification and approval sends resolve through the person; phone exposed for later
- [ ] Backfill endpoint run in prod, then removed
- [ ] No writes to Xero anywhere
- [ ] `CLAUDE.md` updated (models, routes, People tab, picker)
- [ ] `ReleasesPanel.tsx` updated in the same PR (both repos' PRs reference it)
- [ ] Backlog follow-up added: drop legacy `ProjectContact.name` / `email` columns

## Notes

- Ties into #29 (opt-outs): `doNotContactVia` at person level belongs on `CustomerPerson` rather than `ProjectContact` once this lands. Update #29's spec when it's picked up.
- Ties into #49 (SMS history): `phoneE164` lets an SMS thread match a specific person, not just the organisation.
- Ties into #26 (portal): a "who gets the link" decision can now point at a person.

---

# Part B — Design Approval Email: Full-Text Editor + Quote Link

**Branches:** same as Part A, or `feature/ApprovalEmailEditor` in both repos if it ships separately. Part B has no dependency on Part A.

## Goal

1. The send-for-approval modal shows the **whole** email (subject and body from the `approval_request` template), not just a "custom message" box, so staff can edit every word before it goes.
2. If the job has a quote, the email includes the customer's **public quote link**. We currently paste this in by hand.
3. Things that differ per recipient or can't be known until send time show in the editor as **square-bracket placeholder chips**: `[Approve button]`, `[Quote link]`, `[First name]`.

## Today (checked in code)

- `DesignTab.tsx` has a TipTap `RichTextEditor` for a `customMessage`, which the server drops into the `[customMessage]` shortcode of the template. The rest of the template isn't editable at send time.
- `renderTemplate()` replaces **any** `[word]`, and unknown ones become `''`. Staff-typed brackets would be silently eaten if they reached it.
- **Bug, fixed by this work:** scheduled sends (`workers/approvalScheduler.ts`) never receive the custom message. It isn't stored, so scheduled approval emails go out without it.

## Xero public quote link — not available via API

Xero's API doesn't return the customer-facing ("online") quote link. It's an open request on Xero's ideas board, and `learnings.md` already notes it. So Kit **stores the link once it's pasted**:

- `ProjectQuote.publicUrl String?`, a new column. `xeroQuoteUrl` stays: it's the internal Xero link.
- Validated by the pure util `isXeroPublicQuoteUrl(url)`: https, on a `xero.com` host (e.g. `in.xero.com`). Anything else is rejected with a clear message.
- **Two places to paste it:**
  - Financial tab, on the quote row: a "Customer link" field with a copy button.
  - The approval modal itself: when the job has a quote but no link, an inline "Paste the customer quote link" field appears. It saves to the quote, so the link is captured where the need arises and reused on every later send.
- `PATCH /api/projects/:id/quotes/:quoteId/public-url` — allowed with **either** `action.quote.manage` (the key the quote link/create routes already use) **or** `action.design.send-approval`, because the person sending the design may not have quote access.
- Pasting the wrong kind of link gets a helpful message instead of a generic error:
  - An internal `go.xero.com` link pasted into **Customer link** → "That's the internal Xero link — use Copy link on the quote in Xero for the customer one."
  - A customer `in.xero.com` link pasted into **Link existing quote** → "That's a customer link — paste it into Customer link on the quote row." Pure util `classifyXeroQuoteUrl(url)` → `'internal' | 'public' | 'invalid'`.

### Internal Xero quote links are always **view** links

The Financial tab quote row links staff to the quote in Xero. That link should open the quote in **view** mode, never edit.

- Today `POST /:id/quote` (create) stores `https://go.xero.com/app/quotes/edit/{id}`, while `POST /:id/quotes/link` already builds a `/view/` URL from the pasted GUID.
- New pure util `xeroQuoteViewUrl(quoteId)` builds `https://go.xero.com/app/quotes/view/{id}`. Both routes use it, so there's one place the format lives.
- Pure util `toXeroQuoteViewUrl(url)` swaps `/quotes/edit/` for `/quotes/view/` in any internal Xero quote URL (case-insensitive, path segment only, leaves anything else alone). Applied wherever an internal quote URL is saved.
- Migration (SQL in the same migration as the new columns): `UPDATE "ProjectQuote" SET "xeroQuoteUrl" = replace("xeroQuoteUrl", '/quotes/edit/', '/quotes/view/') WHERE "xeroQuoteUrl" LIKE '%/quotes/edit/%';` so existing rows are fixed too.
- The Customer link (`publicUrl`) sits on **the same quote row** on the Financial tab, next to the internal link, with its own copy button and an edit/paste field.

> Phase 2 idea, not in scope: if Xero's "send me a copy" puts quote emails in studio@, Kit could lift the link out of that copy automatically through the Gmail API.

## Placeholders

| Stored as | Shown in editor | Filled at send time with |
|---|---|---|
| `[viewApproveButton]` | `[Approve button]` | The recipient's portal button (each recipient has their own token) |
| `[portalUrl]` | `[Approve link]` | The recipient's raw portal URL (only if the template uses it) |
| `[quoteLink]` | `[Quote link]` | A "View your quote" link button to `ProjectQuote.publicUrl` |
| `[customerName]` | `[First name]` | The recipient's first name; if they have none, the **business name** (`Contact.name` of the project's customer); if neither, "there" |

- **Static values are resolved into plain text in the draft:** project name, file name, version and expiry date. Staff see and can edit the real words.
- First-name fallback lives in one pure helper, `greetingName(person, customer)` → first name → business name → `"there"`, trimmed, never an empty string. Used for every recipient type (project contact, Part A person, Xero billing fallback).
- `[customerName]` stays a chip even with one recipient. This keeps multi-recipient sends correct and means the draft doesn't have to be rebuilt when the recipient selection changes.
- Chips are a custom TipTap inline **atom** node: non-editable, deletable, movable, styled as a pill. They serialise back to the stored `[key]` form.
- An "Insert" menu on the toolbar re-adds any chip that was deleted. `[Quote link]` only appears there when a quote with a link is selected.
- **The approve button is required.** Send is disabled, and the server returns 400, if the body has neither `[viewApproveButton]` nor `[portalUrl]`. Otherwise the customer would have no way to approve.

## Server

New pure module `utils/approvalEmail.ts`, unit-tested:

| Function | Does |
|---|---|
| `extractEmailBody(templateHtml)` | Returns the inner HTML of `<body>` and the wrapper (doctype, `<body style>`) separately |
| `wrapEmailBody(innerHtml, wrapper)` | Puts the edited content back inside the template's wrapper, so the layout and font are kept |
| `prepareApprovalDraft(template, ctx)` | Resolves static shortcodes to escaped text. Keeps the per-send placeholders. Removes `[customMessage]` (legacy). If there's no quote link, removes the paragraph that holds `[quoteLink]` so no empty line is left. Returns `{ subject, bodyHtml }` |
| `renderApprovalEmail(draft, recipientCtx)` | Replaces **only** the allowlisted placeholders above and leaves any other bracket text alone. Builds the buttons. Returns `{ subject, html }` |
| `hasApproveAction(bodyHtml)` | True if `[viewApproveButton]` or `[portalUrl]` is present |
| `sanitiseApprovalHtml(html)` | Allowlist sanitise of staff HTML: `p`, `h2`, `h3`, `strong`, `em`, `u`, `a[href]` (https/mailto only), `ul`, `ol`, `li`, `br`, `hr`, `span[style=color]`. Uses `sanitize-html` (new backend dependency; install locally and in the container per `CLAUDE.md`) |
| `isXeroPublicQuoteUrl(url)` | URL validation as above |
| `greetingName(person, customer)` | First name → business name → "there" |

**Routes** (`designFileRoutes.ts`, `action.design.send-approval`):

- `GET /api/projects/:id/design-files/:fileId/approval-draft` returns:
  - `{ subject, bodyHtml }` from `prepareApprovalDraft`
  - `quotes: [{ id, xeroQuoteNumber, publicUrl }]`
  - `selectedQuoteId`: the latest quote that has a `publicUrl`, otherwise the latest quote
- `POST .../preview-email` accepts `{ subject, bodyHtml, quoteId? }` and renders it for the first recipient (or a "there" sample).
- `POST .../send-approval` accepts `{ subject, bodyHtml, quoteId?, projectContactIds?, scheduledFor? }`:
  - Sanitises the body and checks `hasApproveAction`. If `[quoteLink]` is in the body but the chosen quote has no `publicUrl`, it returns 400 "Add the customer quote link or remove [Quote link]".
  - Stores the draft on the design file for scheduled sends and for the record.
  - Renders per recipient with `renderApprovalEmail`.
  - `customMessage` is still accepted for one release and falls back to the old path if `bodyHtml` is missing. Remove it afterwards.
- **Scheduler** uses the stored draft. Rows scheduled before this ships (no stored draft) use the old template path, so nothing breaks mid-deploy.

**Schema:**

- `DesignFile.approvalSubject String?`, `DesignFile.approvalBodyHtml String?`, `DesignFile.approvalQuoteId Int?`
- `ProjectQuote.publicUrl String?`

**Template:**

- Add `[quoteLink]` to `TEMPLATE_SHORTCODES.approval_request` with a description.
- Mark `[customMessage]` as legacy in its description, then drop it.
- Migration to update the seeded `approval_request` body:
  - Add `<p>You can view your quote here: [quoteLink]</p>` after the file line.
  - Remove `[customMessage]`.
  - Rewrite the small-print lines to use `<span style="color:#666">` so they keep their colour through the editor. Font-size isn't kept, which is acceptable.
  - **Check the live template text first.** It may have been edited in Settings → Templates. If it differs from the seed, make the same changes to the live text rather than overwriting it.

## Frontend

The send modal on `DesignTab.tsx` (extract to `components/Design/ApprovalEmailModal.tsx` while doing this):

- **Order:** recipients, then quote, then subject, then the full-height editor.
  - Recipients use Part A's people when available.
  - The quote `Select` appears only if the project has quotes. It shows the "Paste the customer quote link" inline field when the selected quote has no link.
  - Subject is an editable `TextInput`.
- **Editor:** `RichTextEditor` with headings, bold, italic, underline, link, lists, horizontal rule, text colour, the chip node, and the Insert menu.
- **Preview** button: existing behaviour, now on the edited draft.
- **Reset to template** button: re-fetches the draft, after a confirmation.
- **Send / Schedule:** existing behaviour. Disabled with a reason when the approve chip is missing.
- **Mobile:** single column, with the editor taking the remaining height.

## Tests

**Backend (Vitest)**

| File | Covers |
|---|---|
| `utils/xeroQuoteUrls.test.ts` | `xeroQuoteViewUrl` format; `toXeroQuoteViewUrl` turns `/quotes/edit/` into `/quotes/view/`, leaves `/view/` and non-quote URLs unchanged, handles upper case and a trailing slash or query string; `classifyXeroQuoteUrl` for internal, public, lookalike and junk URLs |
| `utils/approvalEmail.test.ts` | Body extract and wrap round trip keeps the wrapper. Static shortcodes are resolved and escaped (a project name containing `<` or `&`). Per-send placeholders are kept. `[customMessage]` is removed. The quote paragraph is removed when there's no link and kept when there is. `renderApprovalEmail` replaces only allowlisted keys and leaves `[Note to self]` intact. Buttons render with the right URLs. `hasApproveAction` both ways. `greetingName`: first name used; blank or whitespace first name falls back to business name; neither gives "there". Sanitiser strips `script`, `onclick` and `javascript:` hrefs and keeps the allowlist. `isXeroPublicQuoteUrl` accepts and rejects the right URLs (http, non-xero host, lookalike `xero.com.evil.com`) |
| `routes/designFileRoutes.test.ts` | Draft endpoint picks the right quote. 400 without an approve action. 400 when `[quoteLink]` has no link. A scheduled send stores the draft. Legacy `customMessage` path still works |
| `workers/approvalScheduler.test.ts` | Uses the stored draft; falls back when it's null (regression test for the dropped custom message) |
| Quote link route | Either permission key allows it; neither returns 403; invalid URL returns 400 |

**Frontend (Vitest + happy-dom)**

| File | Covers |
|---|---|
| `components/Design/placeholderNode.test.ts` | Chip parses from `[viewApproveButton]` and serialises back; unknown brackets stay as text |
| `ApprovalEmailModal.test.tsx` | Loads the draft into subject and editor. Send is disabled when the approve chip is deleted. The quote paste field shows when the link is missing and saves it. Insert menu re-adds chips. Reset asks for confirmation |

## Acceptance criteria (Part B)

- [ ] Modal shows the full subject and body from the template, all editable
- [ ] Approve button, quote link and first name show as `[ ]` chips and are filled per recipient
- [ ] Quote public link stored on `ProjectQuote`, pasteable from the Financial tab and from the modal, and validated
- [ ] Quote link included when the job has one; the line is dropped cleanly when it doesn't
- [ ] `[First name]` falls back to the business name, then "there"
- [ ] Internal Xero quote links are always `/view/` links: new, linked and existing rows
- [ ] Customer link pasteable on the same Financial tab quote row; wrong-kind pastes get a helpful message
- [ ] Can't send without the approve button (UI and server)
- [ ] Scheduled sends use the edited text (fixes the dropped custom message)
- [ ] Staff HTML sanitised server-side; only allowlisted placeholders replaced
- [ ] Template migration respects any live edits
- [ ] Tests as above
- [ ] `CLAUDE.md` updated (design approval send flow, new columns, `utils/approvalEmail.ts`)
- [ ] `ReleasesPanel.tsx` updated in the same PR

## Notes

- The same editor and chip pattern could later serve the Notify Customer email (`job_notification`). It's not in scope here, but build the chip node and `approvalEmail` helpers so they aren't tied to approval-only names where that's cheap.
