# Spec #50 — Simplify the Contact Page (Contact Info Tab)

**Status:** 📋 Specced — ready to build
**Priority:** Medium
**Labels:** `ux`, `contacts`, `cleanup`, `frontend`
**Branches:** `feature/ContactInfoTab` (projects-frontend; backend only for the permission registry)
**Source:** Daily Log, 20 Oct 2026 — "Simplify contacts — remove overview, put ph & address in one tab". Clarified 21 Oct: the tab is called **Contact info** and holds phones, email, addresses etc.

---

## Goal

Fewer clicks to the thing people actually want from a contact: how to reach them. Replace the **Overview** and **Addresses** tabs with one **Contact info** tab that opens by default and holds phones, email and addresses together.

### Before → after

| Before | After |
|---|---|
| Overview · People · Addresses · Projects · Brand Assets (+ Messages from #49) | **Contact info** · People · Projects · Brand Assets (+ Messages) |

URL slug: `contact-info`.

---

## Contact info tab content

Read-only, as today — Xero is the source of truth and the "Edit in Xero" button stays.

**Phone numbers**
- Every non-empty `ContactPhone`, mobile first, then default/DDI, fax last
- Each number is a `tel:` link (tap to call on mobile) with a copy button
- Type shown as a small label (Mobile, Phone, DDI, Fax)

**Email**
- `Contact.emailAddress` as a `mailto:` link with a copy button
- Any other email fields Xero gives us on the contact (e.g. accounts email) listed underneath, labelled

**Addresses**
- Every `ContactAddress` with content, street before postal
- Formatted multi-line; "Open in Maps" link on street addresses; copy button copies the formatted address

**Website** (if Xero has one) as a link.

**Anything else currently on Overview** (payment terms, contact type, Xero link, Drive folder badge, etc.)
- Moves to the **page header** if it's one line (payment terms, type badges, "No client folder" badge)
- Otherwise sits in a short "Account" section at the bottom of the Contact info tab
- Rule for Claude Code: audit the current Overview tab and list every field in the PR description with where it went. **Nothing is dropped silently.**

Mobile: one column, phones first. On desktop, phones/email left and addresses right is fine.

---

## Permissions

Keys are permanent (`CLAUDE.md`), so **reuse the existing Overview key pair** for the new tab and change its label only:

- `contact.tab.overview.view/.edit` → label "Contact info" (key unchanged, so every role keeps its current access with no migration)
- Remove the Addresses key pair from the registry and the frontend `PermissionKey` union. Stale keys on roles are ignored on read and stripped on next write, per the registry rules.

> Check the actual key names in `permissions/registry.ts` before editing — the names above assume the `contact.tab.<slug>` pattern.

---

## Routing

Contact tabs are URL-synced. Old links (bookmarks, links in notes) must still land somewhere sensible:

- Pure util `resolveContactTab(slug)` in `utils/contactTabs.ts`: `overview` → `contact-info`, `addresses` → `contact-info`, unknown → first permitted tab (via the existing `resolveActiveTab()`)
- Default tab is `contact-info`

---

## Pure helpers (`utils/contactFormat.ts`)

| Function | Does |
|---|---|
| `sortContactPhones(phones)` | Drops empty numbers, orders Mobile → Default → DDI → Fax, de-dupes identical numbers |
| `formatContactPhone(phone)` | Joins Xero's country/area/number parts into a readable string (reuse `composeXeroPhone` / `normalisePhone` logic where it fits) and returns a `tel:` href |
| `formatContactAddress(address)` | Multi-line string from the address parts, skipping blanks |
| `hasAddressContent(address)` | False for Xero's empty placeholder address rows |
| `sortContactAddresses(addresses)` | Street before postal, empties removed |

---

## Tests (frontend, Vitest)

| File | Covers |
|---|---|
| `utils/contactFormat.test.ts` | Phone ordering, empty and duplicate numbers, `tel:` hrefs, address formatting with missing parts, empty Xero placeholders dropped |
| `utils/contactTabs.test.ts` | Legacy slugs map to `contact-info`; unknown slug falls back; no permitted tab → null |
| `ContactInfoTab.test.tsx` | Renders phones, email and addresses; empty states ("No phone numbers in Xero", "No address in Xero"); copy buttons present |

Backend: if `registry.ts` has a test asserting group contents, update it for the removed Addresses keys.

---

## Acceptance criteria

- [ ] Overview and Addresses tabs replaced by one Contact info tab, default on open
- [ ] Every Overview field accounted for (header or Account section), listed in the PR
- [ ] Overview key pair relabelled "Contact info"; Addresses keys removed (backend registry + frontend types)
- [ ] Old `overview` / `addresses` URLs land on Contact info
- [ ] Tap-to-call, mailto and Maps links work on mobile
- [ ] Tests as above
- [ ] `CLAUDE.md` updated (contact page tabs)
- [ ] `ReleasesPanel.tsx` updated in the same PR

## Notes

- Do #50 before or alongside #49 and #51 — those both add to the contact tab strip, so it's worth settling the strip first.
