# Spec #49 — SMS History on the Contact Page

**Status:** 📋 Specced — ready to build
**Priority:** Medium
**Labels:** `feature`, `sms`, `contacts`, `ux`
**Branches:** `feature/ContactSmsHistory` (projects-backend + projects-frontend)
**Builds on:** #42 (Two-Way SMS, live)
**Source:** Daily Log, 20 Oct 2026 — "Show the history of all SMS messages in the customer's contact page"

---

## Goal

A **Messages** tab on the contact detail page showing every SMS conversation we've had with that customer, across all their numbers and jobs, with the same reply box as the inbox. Staff answering the phone or looking up a customer shouldn't have to hunt through `/messages` or open each project.

### Non-goals

- New SMS behaviour. Sending, attribution, read state and actioned state all stay exactly as #42 built them.
- Merging threads. A thread is still one per phone number.
- Showing email history here (separate idea).

---

## Which threads belong to a contact

A thread shows on a contact's Messages tab when **either**:

1. `SmsThread.contactId = contact.id` (auto-linked or assigned via Assign customer), **or**
2. any of its messages has `SmsMessage.projectId` in the contact's projects (`Project.contactId = contact.id`)

Rule 2 catches threads that were never linked to the contact but were attributed to one of their jobs by the two-day rule or a manual Move.

Within a thread the tab shows **all** messages, not just the attributed ones. Each bubble keeps its project tag, so a message about another customer's job (rare, e.g. a shared phone) is still visibly tagged.

Pure util in `utils/smsRules.ts` (or a new `utils/contactSms.ts`):

```ts
contactThreadWhere(contactId: string, projectIds: number[]): Prisma.SmsThreadWhereInput
```

Returns `{ OR: [{ contactId }, { messages: { some: { projectId: { in: projectIds } } } }] }`, omitting the second branch when `projectIds` is empty. Unit-tested without a DB.

---

## Backend

### `GET /api/sms/contacts/:contactId`

In `smsRoutes.ts`.

- Permission: `contact.tab.messages.view` (new, see below)
- 403 while `SystemSettings.smsEnabled` is off, same as the other SMS routes
- 404 when the contact doesn't exist
- Response: threads sorted by `lastMessageAt` desc, each with `id`, `phoneNumber`, `displayName`, `status`, `optedOut`, `lastMessageAt`, `unread` (via `isUnreadFor`), a `lastMessage` preview, and `projects` (distinct `{ id, name }` the thread's messages are attributed to)
- Thread detail and messages reuse the existing `GET /api/sms/threads/:id` — no new detail route

### Replies

Reuse `POST /api/sms/threads/:id/messages` (`action.sms.send`). Replies from the contact tab are **not** attributed to a project (same as replying from the inbox). If staff want a reply tied to a job they send it from the project Messages tab — keep it that way so the two-day rule stays predictable.

### Permission keys

Add to `permissions/registry.ts` (contact-tabs group) and mirror in `types/permissions.ts`:

| Key | Grants |
|---|---|
| `contact.tab.messages.view` | See the Messages tab on a contact |
| `contact.tab.messages.edit` | Reply and use thread actions from that tab (still also needs `action.sms.send` to send) |

Default off for non-admin roles.

---

## Frontend

### Contact Messages tab

- New `components/Contact/Tabs/MessagesTab.tsx`, added to the contact tab config array with the `.view` key. Hidden while SMS is off (`useMessaging().enabled`).
- Layout follows the project Messages tab:
  - **One thread** → show the conversation directly (`ThreadView`)
  - **Several threads** (different numbers / people) → a compact list (number or display name, project tags, last message, time, unread dot) with the selected thread below or beside it on desktop. Selected thread in the query string (`?thread=`) so deep links work.
- Reuse `ThreadView`, `MessageBubble`, `ReplyBox` unchanged. Reply box disabled rules come from `utils/sms.ts` as today (opted out, empty, SMS off, no permission), plus disabled without `contact.tab.messages.edit`.
- Opening a thread marks it read (`markThreadRead`), so the sidebar badge stays right.
- Empty state: "No texts with this customer yet." plus, if they have a mobile on file and the user can send, a **New text** button that opens the same composer the project tab uses.
- Live updates: `useRealtimeEvent('sms.received', …)` — refetch when the event's `threadId` is in the list **or** its `contactName`/project belongs to this contact. Simplest correct version: refetch on any `sms.received` while the tab is open (cheap, one query).
- A count badge on the tab label when any thread is unread for the current user.

### Contact page header

- Small "Text" icon button next to the phone number when the contact has a mobile and the user has `action.sms.send`, linking to the Messages tab with a draft composer open. Optional — drop if it crowds the header after #50.

---

## Tests

### Backend (Vitest)

| File | Covers |
|---|---|
| `utils/contactSms.test.ts` (or `smsRules.test.ts`) | `contactThreadWhere`: contactId only; with projects; empty project list drops the OR branch |
| `routes/smsRoutes.test.ts` (extend) | 403 without `contact.tab.messages.view`; 403 when SMS off; 404 unknown contact; linked thread returned; unlinked thread with an attributed message returned; another customer's thread not returned; sort order; `unread` per user |

### Frontend (Vitest)

- `MessagesTab`: single thread renders the conversation directly; several threads render the list; empty state with and without a mobile/permission; reply box disabled without `.edit`
- Any new pure helper (e.g. grouping project tags per thread) gets its own test

---

## Acceptance criteria

- [ ] `GET /api/sms/contacts/:contactId` with the two-rule thread match
- [ ] `contactThreadWhere` pure util + tests
- [ ] `contact.tab.messages.view/.edit` in registry + frontend types
- [ ] Messages tab on the contact page, hidden when SMS is off
- [ ] Reply, mark read and live refresh work from the tab
- [ ] Backend + frontend tests as above
- [ ] `CLAUDE.md` updated (smsRoutes list, contact tabs)
- [ ] `ReleasesPanel.tsx` updated in the same PR

## Notes

- Ties into #29 (contact opt-outs): once `doNotContactVia` lands, the reply box should also respect `sms` in the contact's list. Not needed for this PR.
- Multi-tenant (#32): no new models; scoping comes with the SMS models.
