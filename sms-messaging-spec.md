# 42. Two-Way SMS Messaging + Live Notifications (Prototype)

**Status:** 🚧 Built — Prototype (ships to production, gated off until tested). See backlog #42 for the remaining live-test steps.
**Priority:** Medium
**Labels:** `feature`, `sms`, `notifications`, `customer-facing`, `architecture`, `prototype`
**Parent:** #29 (Client Communication Preferences)
**Advances:** #11 (Global App State) — builds the foundation, ships with SMS as the only slice

---

## Goal

Two-way SMS with customers, tied to projects. Staff send a job notification (e.g. "Visual Industrie: your vehicle is ready to collect"), customer replies are captured and threaded, every user can see the conversation on the project, and anyone can reply from inside the app. New inbound messages raise a live, sticky toast.

Built into production but gated behind a feature flag so it stays dormant until live-tested.

---

## Gateway

- **sms-gate.app** — spare Android phone with a dedicated SIM as the gateway; self-hosted web dashboard on the NAS (Docker).
- Inbound texts and delivery events reach the backend via an HMAC-signed webhook.
- **SMS carries no thread metadata.** The gateway's message ID is for delivery-state tracking only. Threading is done in the app, keyed on the customer's normalised phone number (E.164).
- All sending goes through a thin `sendSms()` util so the provider can be swapped later (e.g. a short-code provider if volume grows).

---

## Data model (Prisma)

### `SmsThread`
- `id`
- `phoneNumber` — normalised E.164, `@unique`
- `contactId?` → `Contact` (`onDelete: SetNull`)
- `projectId?` → `Project` (`onDelete: SetNull`)
- `status` — `open` | `handled` | `archived` | `spam` (default `open`)
- `handledByUserId?`, `handledAt?` — who marked it actioned, and when
- `optedOut` — `Boolean @default(false)` (set when the customer texts STOP)
- `lastMessageAt`
- `createdAt`, `updatedAt`

### `SmsMessage`
- `id`, `threadId` → `SmsThread` (`onDelete: Cascade`)
- `direction` — `inbound` | `outbound`
- `body`
- `gatewayId?` — `@unique`, for webhook dedupe
- `status?` — `received` | `queued` | `sent` | `delivered` | `failed`
- `sentByUserId?` — null for inbound
- `projectId?` — the project this message was attributed to (supports per-message move)
- `occurredAt`, `createdAt`
- `@@index([threadId])`

### `SmsThreadRead`
- `threadId`, `userId`, `lastReadAt`
- `@@unique([threadId, userId])`
- Per-user read state. A thread is unread for a user when `lastMessageAt > lastReadAt` (or no row exists).

All three models must be ready to take an `organisationId` FK when #32 lands.

---

## Inbound attribution — the two-day rule

On an inbound webhook:

1. Verify HMAC signature; reject on failure.
2. Ignore if `gatewayId` already exists (gateway retries).
3. Normalise the number; find or create the `SmsThread`.
4. If no `contactId`, auto-link when the number matches **exactly one** `ProjectContact` / `Contact`. Zero or multiple matches → leave unlinked.
5. **Project attribution:** if an **outbound** message was sent to this number within the **last 2 days**, attribute the message (and thread) to that outbound's project. Otherwise the message is **unassigned** — it sits under the customer (if known) for a person to file.
6. If the body is `STOP` (case-insensitive, trimmed), set `optedOut = true`.
7. Insert the message, bump `lastMessageAt`, and **reopen** the thread if it was `handled` (a new message needs attention again).
8. Publish a `sms.received` event to the live notification layer.

Also handle `sms:sent` / `sms:delivered` / `sms:failed` to update outbound `status`.

The 2-day window lives in one constant/helper so it's easy to test and change.

---

## Reassignment

- **"Move to another project"** on a message — pick from the customer's other projects (or "Unassigned"). Correct-by-default, cheap manual override; no auto-guessing engine.
- **Assign customer** on unlinked threads — contact picker, same pattern as the `TaskModal` project picker.
- Never auto-create contacts or projects from unknown numbers.

---

## Read state vs. actioned state

Two separate concepts:

- **Read (per user)** — "have *I* seen this?" Opening a thread upserts `SmsThreadRead.lastReadAt` for the current user. Drives each user's personal unread badge.
- **Actioned (per thread, team-wide)** — "does *anyone* still need to deal with this?" A **Mark as actioned** button sets `status = handled` and stamps `handledByUserId` + `handledAt`. The thread drops out of everyone's default inbox view, showing "Actioned by Bren · 2:41pm". A new inbound message reopens it.

**Later (optional):** an avatar "seen by" row on a thread, derived from existing `SmsThreadRead` rows — no new data needed.

---

## SMS Inbox UI

New sidebar page (`/messages`), email-inbox style:

- Thread list showing customer name (or the number if unknown), project tag where attributed, last message preview, time, and bold/dot for unread-for-me.
- **Filters:** Open (default) · Unassigned · Actioned · Archived · Spam.
- Conversation view: chronological bubbles, inbound/outbound, delivery status badge on outbound (Sent / Delivered / Failed, error on hover), sender name on outbound.
- Reply box → `sendSms()` → appends an outbound message. Disabled when the thread has `optedOut`, the body is empty, or the feature is off.
- Actions: Mark as actioned, Move to another project, Assign customer, Archive, Mark as spam.
- Sidebar **unread badge** (per user), like the My Tasks count.

**Project page:** a small Messages panel or tab showing that project's messages, with the same reply box and actions. Sending from a project attributes the outbound to it (which starts the 2-day window).

---

## Live notifications (SSE)

New inbound messages are pushed to the browser — no polling.

**Backend**
- `GET /api/events` — Server-Sent Events stream, session-authenticated, permission-checked.
- A small in-process event bus: the webhook publishes `sms.received`; the SSE route fans it out to connected clients.
- Event payload: `{ type, threadId, messageId, contactName | null, projectId | null, projectName | null, preview }`.
- Heartbeat comment every ~25s to keep proxies from dropping the connection.
- Generic event envelope (`type` + payload) so other features can publish later (task counts, etc).

**Frontend**
- Connect once, app-wide, via `EventSource` with credentials; auto-reconnect.
- On `sms.received`: show a **sticky toast** (`autoClose: false`, same pattern as the new-version prompt) that stays until closed:
  - "New message from **Jane Smith** about **Hilux wrap**" — when attributed
  - "New message from **Jane Smith**" — known customer, unassigned
  - "New message from **an unknown number**" — no match
  - **View** button → the project's Messages panel if attributed, otherwise the thread in the inbox.
- Update the unread badge immediately.
- Don't toast for messages the current user just sent.

**Gotcha:** the NAS reverse proxy may buffer responses — set `X-Accel-Buffering: no` and `Cache-Control: no-cache` on the SSE response, and check the proxy timeout.

---

## Global state foundation (slice of #11)

Build the frame, fill one panel:

- An app-wide `RealtimeProvider` (or similar) owning the single SSE connection and dispatching events by `type` to subscribers.
- A `MessagingContext` holding the unread count and exposing `markThreadRead()` / refresh — the first slice.
- Structured so #11's other candidates (user object, task count badge) slot in later without a rebuild.
- Update #11 in `backlog.md` to note the foundation exists and SMS is the first slice.

---

## Permissions

Per CLAUDE.md, every new page/action gets a key in `permissions/registry.ts`, mirrored in `types/permissions.ts`:

- `page.messages` — inbox page + `/api/events` SMS events
- `project.tab.messages.view` / `.edit` — project panel (view vs reply/actions)
- `action.sms.send` — send outbound / reply

All default **off** for non-admin roles. Enforced server-side with `requirePermission()`; UI gated with `<Can>`.

---

## Gating (prototype)

- `SystemSettings.smsEnabled` (default `false`) — master switch for the send path, inbox, and toasts. Combined with the permission keys above, the feature can be limited to test users first.
- `canNotifySms` flag on `project_stage` taxonomy (mirrors `canNotifyCustomer`) — which stages offer an SMS send.
- The webhook still logs inbound messages while disabled, so nothing is lost during testing.

---

## Gotchas

- **Spam/junk:** anyone can text the number — archive + mark-as-spam, no auto-created records.
- **Privacy:** authenticated-only. The HMAC-verified webhook is the only unauthenticated route.
- **Opt-out:** honour `optedOut`; block sends to that thread (UEM Act good practice).
- **Identify the business** in every outbound text.
- **Phone reliability:** the gateway phone is a single point of failure — surface send failures in the UI rather than failing silently.
- **Multi-tenancy (#32):** design all models to accept `organisationId` later.

---

## Tests

**Backend** (needs #12 — vitest setup — done first, or as part of this)
- Phone normaliser: `021 123 4567`, `+64 21 123 4567`, `0064…`, landlines, junk input.
- Webhook: bad signature rejected; dedupe on `gatewayId`; thread find-or-create.
- Contact auto-link with 0, 1, and 2+ matches.
- **Two-day attribution:** inside window → attributed; outside → unassigned; most recent outbound wins.
- STOP sets `optedOut`; send to opted-out thread rejected.
- New inbound reopens a `handled` thread.
- Status updates from `sms:delivered` / `sms:failed`.
- Unread derivation from `SmsThreadRead`.
- SSE event bus publishes on inbound; `/api/events` requires auth.
- Permission checks on all new routes.

**Frontend (Vitest)**
- Inbox filters (Open / Unassigned / Actioned).
- Unread badge count updates on event and on open.
- Toast text for attributed / known-unassigned / unknown; View routes correctly.
- Reply disabled states (empty, opted out, feature off, no permission).
- Mark as actioned and Move to another project flows.
- `RealtimeProvider` dispatches events to subscribers.

---

## Acceptance criteria

- [ ] `SmsThread`, `SmsMessage`, `SmsThreadRead` models + migration
- [ ] HMAC-verified webhook with dedupe, contact auto-link, two-day attribution, STOP handling
- [ ] `sendSms()` util behind a thin provider interface
- [ ] Outbound send from a project, gated by `smsEnabled`, `canNotifySms`, and permissions
- [ ] SMS Inbox page with filters, per-user unread badge, and thread actions
- [ ] Mark as actioned (team-wide, who/when stamped, reopens on new inbound)
- [ ] Move to another project + assign customer
- [ ] Messages panel on the project page
- [ ] SSE `/api/events` + app-wide `RealtimeProvider` + `MessagingContext`
- [ ] Sticky toast with View deep-link
- [ ] Permission keys added to registry + frontend types
- [ ] Backend + frontend tests as above
- [ ] #11 updated in `backlog.md`
- [ ] `ReleasesPanel.tsx` updated in the same PR

---

## Suggested build order

1. Models + migration, normaliser, `sendSms()`, webhook (with tests)
2. Inbox page + project panel (polling-free, read-only first)
3. Reply, actioned, move/assign
4. SSE + `RealtimeProvider` + sticky toast
5. Gating, permissions, live test on the phone
