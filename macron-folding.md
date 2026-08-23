# Spec — Diacritic-insensitive (macron-folding) Home search

**Branch:** `feature/MacronSearch`
**Codebase:** frontend (React/Mantine/Vite) — client-side, assuming the Home search filters an already-loaded list. See the *DB-query fallback* section at the end if `Home.page.tsx` turns out to filter via a backend query instead.

---

## Problem

The Home (`/`) project search is substring-based on lowercased text. A contact/project whose name contains a macron (e.g. **Kōhanga**) can't be found by typing the plain form (**Kohanga**), and vice versa. This affects every te reo name in the client base — Kaitāia, Ōtāhuhu, Whangārei, kōhanga reo, etc.

## Approach

Unicode **diacritic folding**: decompose to NFD (so `ō` → `o` + combining macron), strip the combining marks, lowercase. Apply the same fold to **both** the query and every searched field, so macrons stop mattering in either direction. Purely in-memory, per-keystroke — trivial at a few hundred projects, no precomputed index needed.

---

## 1. New util — `src/utils/foldText.ts`

```ts
/**
 * Diacritic-insensitive text folding for search.
 * NFD-decomposes, strips combining marks (U+0300–U+036F), lowercases.
 * "Kōhanga" -> "kohanga", so a search for "kohanga" (or "Kōhanga") matches either form.
 */
export const foldText = (s: string): string =>
  s.normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase();
```

## 2. Apply in `Home.page.tsx`

Find the current search filter — it will look roughly like:

```ts
const q = query.toLowerCase();
projects.filter((p) => p.name.toLowerCase().includes(q) /* || other fields */);
```

Replace `.toLowerCase()` with `foldText()` on the query **and on every field currently searched** (name, customer/contact name, and any others already in the predicate — don't drop any). Result should read like:

```ts
const q = foldText(query);
projects.filter(
  (p) =>
    foldText(p.name).includes(q) ||
    foldText(p.contact?.name ?? '').includes(q)
  // ...fold every other field this predicate already checks
);
```

Import ordering (per `.prettierrc.mjs`): `foldText` is a local `@/`-or-relative import, so it sorts after third-party imports. Let prettier place it.

No behaviour change other than macron-insensitivity — a plain query still matches plain records exactly as before.

## 3. Unit tests — `src/utils/foldText.test.ts`

```ts
import { describe, expect, it } from 'vitest';
import { foldText } from './foldText';

describe('foldText', () => {
  it('strips a macron so it matches the plain form', () => {
    expect(foldText('Kōhanga')).toBe('kohanga');
  });

  it('folds multiple macrons across words', () => {
    expect(foldText('Kaitāia Ōtāhuhu')).toBe('kaitaia otahuhu');
  });

  it('lowercases and leaves plain ASCII otherwise unchanged', () => {
    expect(foldText('Acme Signs Ltd')).toBe('acme signs ltd');
  });

  it('returns empty string for empty input', () => {
    expect(foldText('')).toBe('');
  });

  it('is symmetric — a folded query matches a folded macronised source', () => {
    const source = foldText('Te Kōhanga Reo');
    expect(source.includes(foldText('kohanga'))).toBe(true);
    expect(source.includes(foldText('Kōhanga'))).toBe(true);
  });
});
```

Run: `yarn vitest --reporter=verbose` (happy-dom env; test file sits alongside the util per convention).

## 4. Releases entry (required — same PR)

Prepend to the current date block in `src/components/Settings/ReleasesPanel.tsx`, plain English from the user's view, e.g.:

> **Macron-friendly search** — the project search on the Home page now ignores macrons, so typing "Kohanga" finds "Kōhanga" (and the other way round). Works for all te reo names — Kaitāia, Ōtāhuhu, Whangārei, etc.

---

## Acceptance criteria

- [ ] `foldText` util added, exported, prettier/eslint clean
- [ ] Home search folds both the query and every currently-searched field
- [ ] Typing `Kohanga` finds a `Kōhanga` record; typing `Kōhanga` still finds it too
- [ ] Plain-text searches behave exactly as before
- [ ] `foldText.test.ts` passing under `yarn vitest`
- [ ] `ReleasesPanel.tsx` updated in the same PR
- [ ] PR opened on `feature/MacronSearch` — **stop and wait** (no merge)

---

## DB-query fallback (only if the search hits Postgres)

If `Home.page.tsx` turns out to send the search term to a backend endpoint that does an `ILIKE`, do the folding in SQL instead of shipping `foldText`:

1. Enable the extension once (in a migration): `CREATE EXTENSION IF NOT EXISTS unaccent;`
2. Change the predicate to `unaccent(name) ILIKE unaccent($1)` on each searched column. Its default rules cover macrons.
3. Note: `unaccent()` is not `IMMUTABLE`, so this can't back an index as-is — at current data size a seq scan is fine; wrap in an `IMMUTABLE` SQL function later if a functional index is ever wanted.
4. Add a backend `*.test.ts` covering the macron case for the route.

Keep the same branch, releases entry, and "open PR then wait" rule.