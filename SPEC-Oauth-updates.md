# SPEC-oauth-staff-gating — OAuth Login Gating via StaffMember.isActive

**Status:** 📋 Ready to Build  
**Priority:** High  
**Labels:** `auth`, `staff`, `security`, `backend`, `frontend`

**Overview:** Extend Google OAuth login to non-`@vil.nz` users (e.g. `brenmurrell@gmail.com`) by replacing the hardcoded email allowlist with a dynamic lookup against the existing `StaffMember` table. The `StaffMember.isActive` flag becomes the auth gate — inactive staff cannot log in, even if they have an account.

---

## Backend Changes

### Route: Google OAuth Callback (`src/routes/authRoutes.ts` or `index.ts`)

**Current behavior:**
- Hardcoded `allowedEmails` array gates the OAuth callback
- Only `@vil.nz` addresses allowed

**New behavior:**
1. After Passport retrieves Google profile, query `StaffMember` by email
2. Check that staff record exists AND `isActive === true`
3. Deny access with `401 Unauthorized` if either check fails
4. Proceed with session creation if both checks pass

**Implementation notes:**
- Query: `prisma.staffMember.findUnique({ where: { email: profile.emails[0].value } })`
- Error response: `return done(null, false, { message: 'Email not authorised or staff account inactive' })`
- The callback should log failed auth attempts (warn level) for audit purposes
- Do NOT create a new `User` for emails not in `StaffMember` — deny immediately
- Remove the old hardcoded `allowedEmails` array entirely

**Test cases:**
- ✅ Active staff member (email in `StaffMember`, `isActive=true`) → login succeeds, session created
- ❌ Inactive staff member (email in `StaffMember`, `isActive=false`) → login denied, no session
- ❌ Email not in staff table → login denied, no session
- ❌ External email (e.g. `user@gmail.com`) with no corresponding `StaffMember` → login denied
- ✅ Admin staff (Bren) can still log in (pre-existing `User` + `StaffMember` linked)

---

## Frontend Changes

### Component: `Settings/StaffPanel.tsx`

**Current:**
- `isActive` toggle shows only the checkbox label

**New:**
- Add descriptive text explaining that `isActive` controls login access
- Update label and description to be explicit

**Code example:**
```tsx
<Checkbox
  label="Active (can log in)"
  description="Unchecked = blocks this staff member from signing in"
  checked={editing.isActive}
  onChange={(e) => setEditing({ ...editing, isActive: e.currentTarget.checked })}
/>
```

---

## Database

**No schema changes required.** `StaffMember` already has:
- `email String @unique` — used as the lookup key
- `isActive Boolean @default(true)` — gates login

---

## Implementation Order

1. **Backend: Update OAuth callback** (`authRoutes.ts` or `index.ts`)
   - Remove hardcoded `allowedEmails` array
   - Add Prisma `StaffMember` lookup in the callback
   - Add logging for denied access attempts
   - Verify locally with test staff member

2. **Frontend: Update StaffPanel** (`StaffPanel.tsx`)
   - Add descriptive label + description to `isActive` checkbox
   - Verify visually in Settings → Staff tab

3. **Backend: Add unit tests** (`src/routes/authRoutes.test.ts`)
   - Test all four scenarios above
   - Mock Prisma `staffMember.findUnique`

4. **Frontend: No new tests needed** — StaffPanel is already tested in the existing suite

5. **Update CLAUDE.md** — remove reference to hardcoded `allowedEmails`

6. **Open a PR** and verify:
   - Try logging in as an external user with a corresponding active `StaffMember` (should succeed)
   - Try logging in as an external user with an inactive `StaffMember` (should fail)
   - Try logging in as an email not in the staff table (should fail)
   - Existing `@vil.nz` users still work

---

## Acceptance Criteria

- [ ] Hardcoded `allowedEmails` array removed from OAuth callback
- [ ] OAuth callback queries `StaffMember` by email before allowing login
- [ ] `isActive=false` blocks login (returns 401 or redirect to login page)
- [ ] Email not in `StaffMember` table blocks login
- [ ] Existing admin staff (Bren, Bev) can still log in
- [ ] Failed login attempts logged at warn level with email + reason
- [ ] `StaffPanel` UI clarifies that `isActive` controls login access
- [ ] Unit tests cover all four scenarios (active/inactive/not-found/etc)
- [ ] CLAUDE.md updated to remove hardcoded allowlist mention
- [ ] ReleasesPanel.tsx updated with a release entry

---

## Open Questions

None — implementation is straightforward using existing `StaffMember` table.

---

## Notes

- This change allows VIL to onboard external contractors or remote staff by creating a `StaffMember` record with their email (any domain) and setting `isActive=true`
- The `User` model will still be created on first login (standard Passport flow), but only if the email passes the `StaffMember` gate
- If an external staff member leaves, just set `isActive=false` in Settings → Staff; their session will expire naturally, and next login will be denied
- No environment variable changes needed — all auth control is now in the database

---

**Related backlog items:** None  
**Related specs:** None  
**Last updated:** August 19, 2026