# Feature Spec: Client Portal — Design Approval Workflow (MVP)

**Status:** 📋 Ready to Build
**Priority:** High
**Labels:** `feature`, `customer-facing`, `magic-link`, `security`, `design-approval`
**Last Updated:** 01 March 2026
**Origin:** Voice conversation — 01 March 2026

---

## Overview

Build a token-based client-facing portal allowing clients to review, approve, and provide feedback on design revisions without requiring usernames or passwords. Each client contact receives a unique, time-limited token link. Approvals are verified via multi-factor email confirmation. Feedback entries flow directly into the internal tasks table.

---

## Goals

- Remove password friction for clients reviewing designs
- Provide a clear, auditable approval workflow per design revision
- Support multiple contacts per project with individual token tracking
- Capture structured, actionable feedback that lands in the tasks table
- Build the data model to support cross-project contact history in future

---

## Scope (MVP)

- Token generation and delivery per contact
- Token expiry and re-authentication via email
- Design PDF display from Google Drive
- Approval flow with MFA confirmation step
- Structured feedback entry (one item per entry)
- Feedback written to tasks table
- Project-level contact management (override Xero default)
- Audit logging: IP address, user agent, timestamps, geolocation

---

## Out of Scope (Future)

- Cross-project contact history / portal dashboard
- Client self-service login
- Notifications beyond approval emails

---

## User Flows

### Internal (Team)

1. Open a project → Design tab
2. Select or upload a PDF from Google Drive
3. Ensure file permissions are set to publicly readable
4. Add or confirm client contacts for the project
5. Click "Send for Approval"
6. System generates a unique token per contact and sends personalised email

### Client

1. Receives email with unique token link
2. Clicks link → lands on approval portal (token validated)
3. Views PDF design
4. Chooses: **Approve** or **Request Changes**
5. If requesting changes: adds structured feedback entries (one per item)
6. On submitting approval: receives MFA confirmation email with one-time code
7. Enters code → approval is locked and recorded

### Token Expiry

1. Client clicks an expired token link
2. Shown "This link has expired" screen with "Request new link" button
3. Email sent to their registered address with fresh token
4. Client clicks new link → continues as normal

---

## Schema Updates

### New Table: `project_contacts`

Per-project contact overrides. If none exist for a project, fall back to the default Xero contact. `xero_contact_id` is nullable.

```sql
CREATE TABLE project_contacts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  first_name      VARCHAR(100) NOT NULL,
  last_name       VARCHAR(100) NOT NULL,
  email           VARCHAR(255) NOT NULL,
  is_primary      BOOLEAN DEFAULT FALSE,
  xero_contact_id VARCHAR(255) NULL,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_project_contacts_project ON project_contacts(project_id);
CREATE INDEX idx_project_contacts_email ON project_contacts(email);
```

### New Table: `design_approval_tokens`

Token = `crypto.randomBytes(48).toString('hex')`. Default expiry 14 days. `contact_id` nullable to support Xero-only contacts.

```sql
CREATE TABLE design_approval_tokens (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id         UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  design_revision_id UUID NOT NULL REFERENCES design_revisions(id) ON DELETE CASCADE,
  contact_id         UUID NULL REFERENCES project_contacts(id) ON DELETE SET NULL,
  xero_contact_id    VARCHAR(255) NULL,
  email              VARCHAR(255) NOT NULL,
  first_name         VARCHAR(100),
  last_name          VARCHAR(100),
  token              VARCHAR(128) NOT NULL UNIQUE,
  expires_at         TIMESTAMPTZ NOT NULL,
  used_at            TIMESTAMPTZ NULL,
  revoked_at         TIMESTAMPTZ NULL,
  created_at         TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_dat_token ON design_approval_tokens(token);
CREATE INDEX idx_dat_project ON design_approval_tokens(project_id);
CREATE INDEX idx_dat_email ON design_approval_tokens(email);
```

### New Table: `design_approvals`

```sql
CREATE TABLE design_approvals (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  design_revision_id  UUID NOT NULL REFERENCES design_revisions(id),
  token_id            UUID NOT NULL REFERENCES design_approval_tokens(id),
  project_id          UUID NOT NULL REFERENCES projects(id),
  email               VARCHAR(255) NOT NULL,
  status              VARCHAR(20) NOT NULL CHECK (status IN ('approved', 'changes_requested')),
  mfa_verified        BOOLEAN DEFAULT FALSE,
  mfa_code            VARCHAR(10) NULL,
  mfa_code_expires_at TIMESTAMPTZ NULL,
  mfa_verified_at     TIMESTAMPTZ NULL,
  ip_address          INET NULL,
  user_agent          TEXT NULL,
  geolocation         JSONB NULL,
  submitted_at        TIMESTAMPTZ NULL,
  created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_da_revision ON design_approvals(design_revision_id);
CREATE INDEX idx_da_project ON design_approvals(project_id);
```

### New Table: `portal_audit_log`

Event types: `token_accessed`, `token_expired`, `token_reissued`, `approval_started`, `mfa_sent`, `mfa_verified`, `mfa_failed`, `approval_submitted`, `feedback_submitted`.

```sql
CREATE TABLE portal_audit_log (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  token_id    UUID REFERENCES design_approval_tokens(id) ON DELETE SET NULL,
  project_id  UUID REFERENCES projects(id) ON DELETE SET NULL,
  email       VARCHAR(255),
  event_type  VARCHAR(50) NOT NULL,
  ip_address  INET NULL,
  user_agent  TEXT NULL,
  geolocation JSONB NULL,
  metadata    JSONB NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pal_token ON portal_audit_log(token_id);
CREATE INDEX idx_pal_project ON portal_audit_log(project_id);
```

### Updates to `design_revisions`

```sql
ALTER TABLE design_revisions
  ADD COLUMN drive_file_id    VARCHAR(255) NULL,
  ADD COLUMN drive_file_url   TEXT NULL,
  ADD COLUMN approval_status  VARCHAR(20) DEFAULT 'pending'
    CHECK (approval_status IN ('pending', 'sent', 'approved', 'changes_requested')),
  ADD COLUMN approval_sent_at TIMESTAMPTZ NULL;
```

### Updates to `tasks`

Add if not already present. Feedback inserts use `source = 'client_feedback'` and `source_reference_id = <design_approval_id>`.

```sql
ALTER TABLE tasks
  ADD COLUMN source              VARCHAR(50) NULL,
  ADD COLUMN source_reference_id UUID NULL;
```

---

## API Routes

### Internal (Authenticated)

| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/api/projects/:id/contacts` | List project contacts |
| `POST` | `/api/projects/:id/contacts` | Add a contact |
| `PUT` | `/api/projects/:id/contacts/:contactId` | Update a contact |
| `DELETE` | `/api/projects/:id/contacts/:contactId` | Remove a contact |
| `POST` | `/api/projects/:id/design-revisions/:revId/send-approval` | Generate tokens, set Drive permissions, send emails |
| `GET` | `/api/projects/:id/design-revisions/:revId/approvals` | View approval status per contact |

### Public (Token-Authenticated)

| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/portal/:token` | Validate token, return revision + PDF details |
| `POST` | `/portal/:token/request-mfa` | Trigger MFA email |
| `POST` | `/portal/:token/verify-mfa` | Verify MFA code |
| `POST` | `/portal/:token/approve` | Submit approval (MFA required) |
| `POST` | `/portal/:token/feedback` | Add single feedback entry |
| `POST` | `/portal/:token/reissue` | Request fresh token (expired only) |

---

## Email Templates

1. **Approval Request** — personalised link to `/portal/:token`, expiry date noted
2. **New Token Issued** — fresh link after expiry
3. **MFA Code** — 6-digit one-time code, expires in 10 minutes
4. **Approval Confirmed (Internal)** — notifies team when approval is locked in

---

## Frontend Components

### Internal

| Component | Description |
|-----------|-------------|
| `ProjectContactsPanel` | Lists primary + supplementary contacts with add/edit/remove |
| `AddContactForm` | First name, last name, email, primary toggle |
| `SendForApprovalButton` | Triggers send flow; shows per-contact status badges |
| `ApprovalStatusBadge` | Pending / Approved / Changes Requested |

### Client Portal (`/portal/:token`)

| Component | Description |
|-----------|-------------|
| `PortalLayout` | Clean, minimal layout — no app shell |
| `DesignViewer` | Embedded PDF via Google Drive |
| `ApprovalActions` | Approve / Request Changes toggle |
| `FeedbackEntryList` | Ordered list of submitted items |
| `FeedbackEntryForm` | Multiline textarea, one entry at a time |
| `MFAConfirmationModal` | 6-digit code entry to confirm approval |
| `TokenExpiredScreen` | Expired message + request new link CTA |

---

## Acceptance Criteria

### Token Generation & Delivery

- [ ] Unique token generated per contact on "Send for Approval"
- [ ] Token is cryptographically random (min 64 hex chars)
- [ ] Token expires after 14 days by default
- [ ] Email sent to each contact with their unique portal link
- [ ] Google Drive file permissions set to publicly readable on send

### Token Access & Expiry

- [ ] Valid token → portal loads with correct PDF and project context
- [ ] Expired token → "Link expired" screen shown, not the portal
- [ ] Expired token → "Request new link" sends fresh token email
- [ ] Token access logged with IP, user agent, timestamp, geolocation

### Project Contacts

- [ ] Contacts can be added, edited, removed on the project details tab
- [ ] One contact can be marked primary per project
- [ ] Falls back to Xero default contact if no project contacts exist
- [ ] UI displays first name, last name, email per contact

### Approval Flow

- [ ] Client can select Approve or Request Changes
- [ ] Submitting triggers MFA email with 6-digit code
- [ ] MFA code expires after 10 minutes
- [ ] Approval not recorded until MFA is verified
- [ ] Approval status stored in `design_approvals`
- [ ] Internal notification sent on successful approval

### Feedback

- [ ] Client adds feedback one item at a time via multiline input
- [ ] Each submitted entry creates a separate row in `tasks`
- [ ] Task linked to correct project and design revision
- [ ] Task `source` = `'client_feedback'`

### Audit Logging

- [ ] All token access attempts logged
- [ ] MFA sent, verified, and failed events logged
- [ ] Approval submission logged
- [ ] Each log entry captures IP, user agent, geolocation, timestamp

### Future-Proofing

- [ ] Token linked to both contact identifier and design revision
- [ ] Email stored on token at time of issue for historical lookups
- [ ] `portal_audit_log` queryable by email across all projects

---

## Implementation Notes

### Relationship to Existing Backlog

This spec supersedes and expands on **Backlog Item #4** (Google Drive Design Approval System — Phase 2). It incorporates the customer-facing flow described there while adding the project contacts, MFA, and audit logging layers.

### Prisma Considerations

The SQL schema above is written for reference. Actual implementation should use Prisma models and migrations, consistent with the rest of the VisualOS codebase. UUIDs in Prisma use `@default(uuid())`. All tables should follow existing naming conventions (camelCase fields in Prisma, snake_case in the database via `@map`).

### Testing

Both backend and frontend should include unit tests for this feature:

- **Backend:** Token generation, HMAC-less public route auth, MFA code generation/expiry, feedback-to-task creation, audit log writes
- **Frontend:** Portal component rendering with valid/expired tokens, MFA modal flow, feedback form submission

---

## Open Questions

1. **Token expiry** — fixed 14 days, or configurable per send?
2. **Portal branding** — client-specific branding, or stay generic for MVP?
3. **Changes requested** — does this block a revision from being marked complete internally?
4. **Multiple revisions** — do we need to support multiple revisions in flight simultaneously per project?
