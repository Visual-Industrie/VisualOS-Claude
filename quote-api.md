# Spec #30 — WordPress Quote Request Form → VisualOS API

**Status:** 📋 Planned  
**Priority:** High  
**Labels:** `feature`, `api`, `intake`, `contacts`  
**Branch:** `feature/QuoteRequestIntake`

---

## Overview

A public, unauthenticated `POST /api/quote-requests` endpoint that receives submissions from the Visual Industrie WordPress contact/quote form. It performs fuzzy contact matching against existing Xero-synced contacts, creates or reuses a `Contact` record, and scaffolds a new `Project` in `New` status — ready for the team to pick up in VisualOS.

No frontend changes required. This is entirely a backend feature.

---

## Behaviour Summary

1. Request arrives from `visualindustrie.co.nz` — origin is validated, others rejected with `403`.
2. Payload is validated — required fields checked, `400` returned on failure.
3. Fuzzy contact match attempted (see matching logic below).
4. If a match is found → reuse that `Contact`.  
   If no match → create a new local `Contact` record (no Xero write at this stage).
5. A new `Project` is created with status `New`, linked to the contact.
6. A `Note` is created on the project containing the full submission detail (message, job type, dimensions, etc.) so nothing is lost.
7. A new `Task` is created: "Follow up quote request" assigned to no one, due in 1 business day, priority `high`.
8. A notification email is sent to `studio@vil.nz` via `sendEmail.ts` summarising the lead.
9. `201` response returned — no sensitive data echoed back.

---

## Origin Locking

The endpoint is public (no session auth). It must be protected by `Origin` header validation.

```
Allowed origins:
  - https://visualindustrie.co.nz
  - https://www.visualindustrie.co.nz
  - http://localhost:5173  (dev only, when NODE_ENV !== 'production')
```

Reject any request where `Origin` does not match → `403 Forbidden`.

> **Note:** Do not use `cors()` middleware for this route — implement a dedicated inline check so the logic is explicit and auditable.

---

## Request Payload

```ts
// POST /api/quote-requests
// Content-Type: application/json
// No auth required

interface QuoteRequestPayload {
  // Contact identity
  businessName: string;        // required
  contactName: string;         // required — first + last name of the person submitting
  email: string;               // required
  phone?: string;              // optional

  // Job details (all optional — captured in note)
  jobType?: string;            // e.g. "Vehicle Wrap", "Shop Signage", "Event Banners"
  description?: string;        // free-text message / job description
  dimensions?: string;         // e.g. "3m x 1.2m"
  quantity?: number;
  deadline?: string;           // ISO date string or free text — stored as-is in the note
}
```

All fields beyond `businessName`, `contactName`, and `email` are optional but should be captured and written to the project note if present.

---

## Contact Matching Logic

Fuzzy match is performed in this order. First match wins — do not apply multiple strategies.

### Strategy 1 — Exact email match
```sql
SELECT * FROM "Contact" WHERE "emailAddress" = lower(trim(:email)) LIMIT 1
```
If found → use this contact. High confidence.

### Strategy 2 — Fuzzy business name match
- Normalise both sides: lowercase, strip punctuation, collapse whitespace.
- Compare incoming `businessName` against `Contact.name` for all contacts.
- Use a simple token overlap score: split both strings into words, count shared tokens.
- Threshold: **≥ 70% of the shorter string's tokens must match**.
- If multiple contacts score above threshold → pick the highest score. Tie-break: most recently updated.

```ts
// Normalise helper
function normalise(s: string): string[] {
  return s
    .toLowerCase()
    .replace(/[^a-z0-9\s]/g, '')
    .split(/\s+/)
    .filter(Boolean);
}

function tokenOverlapScore(a: string, b: string): number {
  const tokensA = new Set(normalise(a));
  const tokensB = new Set(normalise(b));
  const shared = [...tokensA].filter(t => tokensB.has(t)).length;
  const shorter = Math.min(tokensA.size, tokensB.size);
  return shorter === 0 ? 0 : shared / shorter;
}
```

### Strategy 3 — No match
Create a new local `Contact` record. Do **not** write to Xero at this point — a human should review before promoting to Xero. Set a `source` field (see schema below) to `'quote_request'` so staff know this contact came in via the web form.

---

## Prisma Schema Changes

### `Contact` model — add `source` field

```prisma
model Contact {
  // ... existing fields ...
  source  String?  // e.g. 'xero', 'quote_request' — null for legacy records
}
```

Migration name: `add_source_to_contact`

### `QuoteRequest` model — audit log

Store every inbound submission regardless of outcome. Useful for debugging and duplicate detection.

```prisma
model QuoteRequest {
  id            String    @id @default(uuid())
  createdAt     DateTime  @default(now())
  businessName  String
  contactName   String
  email         String
  phone         String?
  jobType       String?
  description   String?
  dimensions    String?
  quantity      Int?
  deadline      String?
  matchStrategy String?   // 'email', 'fuzzy_name', 'none'
  contactId     String?   // FK to Contact if matched/created
  projectId     String?   // FK to Project created
  contact       Contact?  @relation(fields: [contactId], references: [id])
  project       Project?  @relation(fields: [projectId], references: [id])
}
```

Migration name: `add_quote_request`

> Also add the `quoteRequests` relation to `Contact` and `Project` models.

---

## New Route File

Create `src/routes/quoteRequestRoutes.ts`.

Mount in `index.ts`:
```ts
import quoteRequestRoutes from './routes/quoteRequestRoutes';
app.use('/api/quote-requests', quoteRequestRoutes);
```

> This route must be mounted **before** `ensureAuthenticated` is applied globally — or excluded from it. Check `index.ts` mount order carefully.

### `POST /api/quote-requests`

```
Public — no session auth
Rate limit: 10 requests per IP per hour (use express-rate-limit)
```

**Steps:**

```
1. Validate Origin header → 403 if not allowed
2. Validate required fields (businessName, contactName, email) → 400 with field errors if missing
3. Validate email format → 400 if invalid
4. Run contact matching (Strategy 1 → 2 → 3)
5. If new contact: INSERT into Contact with source='quote_request'
6. Create Project:
   - name: `{businessName} — {jobType ?? 'Quote Request'}`
   - status: 'New'
   - contactId: matched/created contact id
   - source: 'quote_request'  (see note below)
7. Create Note on project with full submission details (see note template below)
8. Create Task:
   - title: 'Follow up quote request'
   - priority: 'high'
   - dueDate: next business day (skip Sat/Sun from now())
   - projectId: new project id
   - staffMemberId: null (unassigned — let team pick it up)
9. Create QuoteRequest audit record
10. Send internal notification email to studio@vil.nz
11. Return 201 { success: true }
```

### Note Template

```
## Quote Request — {contactName} ({businessName})

Received: {createdAt in NZ local time}

**Contact**
- Name: {contactName}
- Email: {email}
- Phone: {phone ?? '—'}

**Job Details**
- Type: {jobType ?? '—'}
- Dimensions: {dimensions ?? '—'}
- Quantity: {quantity ?? '—'}
- Deadline: {deadline ?? '—'}

**Message**
{description ?? '(no message provided)'}

---
*Submitted via visualindustrie.co.nz*
```

Store as plain HTML (convert markdown → HTML using a simple template string, not an external lib).

### Internal Notification Email

Send via existing `sendEmail.ts` to `studio@vil.nz`:

```
Subject: New Quote Request — {businessName}

Body (plain HTML):
  Contact match: {matchStrategy} ({contact name / new})
  Business: {businessName}
  Person: {contactName} <{email}>
  Job type: {jobType ?? '—'}
  Message: {description ?? '—'}

  View project: https://visualos.vil.nz/projects/{projectId}
```

Do **not** label this email with the job label (project was just created — consistent with other new-project flows).

---

## `Project` model — add `source` field

```prisma
model Project {
  // ... existing fields ...
  source  String?  // e.g. 'xero', 'quote_request' — null for legacy records
}
```

This lets the Home page and Kanban eventually filter/badge leads that came in via the web form (out of scope for this PR, but design for it now).

Migration: include in `add_quote_request` migration above.

---

## Rate Limiting

Install `express-rate-limit` if not already present:

```bash
npm install express-rate-limit
```

Apply to this route only:

```ts
import rateLimit from 'express-rate-limit';

const quoteRateLimit = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10,
  message: { error: 'Too many requests. Please try again later.' },
  standardHeaders: true,
  legacyHeaders: false,
});

router.post('/', quoteRateLimit, async (req, res) => { ... });
```

---

## Error Responses

| Condition | Status | Body |
|---|---|---|
| Origin not allowed | `403` | `{ error: 'Forbidden' }` |
| Missing required fields | `400` | `{ error: 'Validation failed', fields: ['businessName', ...] }` |
| Invalid email format | `400` | `{ error: 'Invalid email address' }` |
| Rate limit exceeded | `429` | `{ error: 'Too many requests. Please try again later.' }` |
| Unhandled server error | `500` | `{ error: 'Internal server error' }` |

Success:
```json
HTTP 201
{ "success": true }
```

No project ID, contact ID, or internal data should be returned to the caller.

---

## Unit Tests

Create `src/routes/quoteRequestRoutes.test.ts`. Cover:

- `POST` with valid payload → `201`
- `POST` from disallowed origin → `403`
- `POST` missing required fields → `400` with correct field list
- `POST` invalid email → `400`
- Contact matching — exact email match uses existing contact
- Contact matching — fuzzy name match above threshold uses existing contact
- Contact matching — no match creates new contact with `source='quote_request'`
- `tokenOverlapScore` helper — export and test directly:
  - identical strings → `1.0`
  - completely different → `0.0`
  - partial match → correct ratio
  - handles punctuation and case differences

Export `tokenOverlapScore` and `normalise` from a `src/utils/fuzzyMatch.ts` utility file so they can be tested in isolation.

---

## Acceptance Criteria

- [ ] `POST /api/quote-requests` is publicly accessible (no session required)
- [ ] Requests from origins other than `visualindustrie.co.nz` are rejected with `403`
- [ ] Missing `businessName`, `contactName`, or `email` returns `400` with field list
- [ ] Exact email match reuses existing `Contact` record
- [ ] Fuzzy business name match (≥ 70% token overlap) reuses existing `Contact`
- [ ] No match creates a new `Contact` with `source = 'quote_request'`
- [ ] New `Project` created with status `New`, linked to contact
- [ ] `Project.source` set to `'quote_request'`
- [ ] `Note` created on project with full submission details
- [ ] `Task` created: "Follow up quote request", priority high, due next business day
- [ ] `QuoteRequest` audit record created with `matchStrategy`, `contactId`, `projectId`
- [ ] Internal notification email sent to `studio@vil.nz`
- [ ] Rate limiting: max 10 requests/IP/hour
- [ ] All unit tests passing
- [ ] `ReleasesPanel.tsx` updated

---

## Out of Scope (This PR)

- Xero contact creation for new web leads (manual promotion — future)
- SMS or webhook notification to staff
- Home page / Kanban badge for `source='quote_request'` leads
- WordPress form implementation (handled separately)
- Admin UI for viewing raw `QuoteRequest` audit log