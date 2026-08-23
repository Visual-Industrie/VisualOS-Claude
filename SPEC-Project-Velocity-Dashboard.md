# SPEC — Project Velocity Dashboard

## Overview

A new admin-visible-to-all-staff page, `/project-velocity`, showing how long projects spend in
each stage of the pipeline (New → Design → Production → Invoice → Completed, or whichever
stages are flagged for tracking). Built entirely from data already captured in
`ProjectStatusLog` — no new tracking required. Goal: surface bottlenecks (which stage is slow)
and currently-stuck projects (which specific jobs are overdue relative to typical).

This follows the same pattern as the Financial Overview page: a pure, unit-tested maths module
(`utils/projectVelocity.ts`) + a thin route + a Recharts-based frontend page.

---

## Data Model Changes

### `TaxonomyItem.meta` (no migration — JSON field already exists)

Add a new optional flag to `project_stage` taxonomy items, alongside the existing
`showInKanban` / `closesXero` / `sendNotification` / `canNotifyCustomer` / `showOnOverview` /
`expectsInvoice` flags:

- `includeInVelocity: boolean` (default/absent = `false`) — this stage counts toward the
  velocity dashboard's stage list. Bren picks which stages matter (e.g. skip "With Customer" if
  it's not a meaningful checkpoint).

Surfaced in **Settings → Lists → Project stages** as a new toggle, same UI pattern as the other
stage flags. No backend route changes needed — existing `PATCH /api/taxonomy/:type/:id` already
accepts arbitrary `meta` updates.

**No other schema changes.** `ProjectStatusLog` (projectId, status, userId, userName, createdAt)
already has everything needed.

---

## Backend

### `utils/projectVelocity.ts` (pure functions, fully unit-tested — mirror `financialOverview.ts`)

**Core idea:** for each project, walk its `ProjectStatusLog` rows in `createdAt` order. Each
consecutive pair `(logA, logB)` means the project was in `logA.status` for
`logB.createdAt - logA.createdAt`. The project's *current* stage duration (no `logB` yet) is
`now - logA.createdAt`.

**Eligibility filter (apply before any calculation):**
- A project is only included in the dataset if its **earliest** `ProjectStatusLog` row has
  `status` equal to the taxonomy stage with the lowest `sortOrder` among stages currently
  flagged `includeInVelocity` (in practice, "New"). This is a proxy for "we have complete
  history for this project." `ProjectStatusLog` only started recording in March 2026, so
  projects created before then either have no rows or a log history that starts mid-pipeline —
  both are excluded entirely, not just for the stages they're missing. This avoids silently
  understating a stage's duration from partial data.
- This filter is applied once per project, not per stage — a project either has a clean "New"
  start and is included for all its stage-transitions, or it's excluded completely.

**Exported functions:**

```ts
interface StatusLogRow {
  projectId: number;
  status: string;
  createdAt: Date;
}

interface StageDuration {
  projectId: number;
  stage: string;
  days: number;       // fractional days
  isCurrent: boolean;  // true if this is the project's present stage (no exit log yet)
}

// Filters to eligible projects (has a "New"-first log) and returns one row per
// stage-occupancy interval, including the current in-progress interval.
function computeStageDurations(
  logs: StatusLogRow[],
  firstTrackedStage: string
): StageDuration[]

interface StageStats {
  stage: string;
  sortOrder: number;
  medianDays: number;
  p25Days: number;
  p75Days: number;
  sampleSize: number; // completed intervals only — excludes the still-in-progress ones
}

// One entry per tracked stage, ordered by taxonomy sortOrder.
// Only uses *completed* stage intervals (isCurrent: false) for the stats —
// in-progress durations would bias toward "fast" since they're truncated at "now".
function computeStageStats(durations: StageDuration[], trackedStages: TaxonomyStage[]): StageStats[]

interface StuckProject {
  projectId: number;
  projectName: string;
  customerName: string | null;
  stage: string;
  daysInStage: number;
  p75ForStage: number; // the benchmark it's exceeding
}

// Projects whose *current* stage duration exceeds that stage's P75 (from computeStageStats).
// Sorted worst-first (days over benchmark, descending).
function findStuckProjects(
  durations: StageDuration[],
  stats: StageStats[],
  projects: { id: number; name: string; customerName: string | null; status: string }[]
): StuckProject[]

// median/percentile helpers (reuse or extract from financialOverview.ts if overlap)
function median(values: number[]): number
function percentile(values: number[], p: number): number
```

### `GET /api/project-velocity`

Query params:
- `startDate`, `endDate` (optional, ISO date) — filters which *stage-entry* events count toward
  the stats (i.e. a stage interval counts if the entry into that stage falls in the range).
  Unbounded if omitted.
- `includeAbandoned` (optional boolean, default `false`) — when `false`, exclude projects whose
  current `status` is an "Abandoned"-type taxonomy stage (or any non-`includeInVelocity` stage
  reached after their last tracked stage — i.e. they left the tracked pipeline). When `true`,
  include their completed-interval data in the stats (their current/final stage doesn't produce
  a "stuck" entry either way, since it's not a tracked stage).

Response:
```json
{
  "stages": [
    { "stage": "New", "sortOrder": 0, "medianDays": 2.1, "p25Days": 0.5, "p75Days": 5.0, "sampleSize": 84 },
    { "stage": "Design", "sortOrder": 1, "medianDays": 6.4, "p25Days": 3.0, "p75Days": 12.0, "sampleSize": 79 }
  ],
  "stuckProjects": [
    { "projectId": 412, "projectName": "Acme Fascia Sign", "customerName": "Acme Corp", "stage": "Production", "daysInStage": 18.2, "p75ForStage": 9.0 }
  ],
  "excludedProjectCount": 37,
  "trackedStages": ["New", "Design", "Production", "Invoice", "Completed"]
}
```

`excludedProjectCount` = total projects filtered out for lacking a "New"-first log history, so
the frontend caption can say "excludes N pre-March projects" honestly rather than silently
dropping them.

Route lives in a new `projectVelocityRoutes.ts`, mounted at `/api/project-velocity`. No
admin-gating (see decision below — visible to all authenticated staff, same as My Tasks).
Fetches: `TaxonomyItem` (type=`project_stage`, ordered by `sortOrder`, filter `meta.includeInVelocity`),
`ProjectStatusLog` (all rows, or date-bounded), `Project` (id, name, status, contact name) for
the stuck-projects join.

---

## Frontend

### `ProjectVelocity.page.tsx` — new page at `/project-velocity`, sidebar link

- **Filter bar:** date range (`DatePickerInput` range mode, Auckland-default like Financial
  Overview), "Include abandoned projects" switch (default off).
- **Stage bar chart** (Recharts `BarChart`, horizontal or vertical — vertical to match sortOrder
  left-to-right like a funnel): median days per stage, with a range indicator (P25–P75) per bar.
  Caption under the chart: *"Based on N projects with tracked history since March 2026 — M
  earlier projects excluded (no recorded start date)."* using `sampleSize` totals and
  `excludedProjectCount`.
- **Stuck projects table:** project name (link to project), customer, stage, days in stage vs
  P75 benchmark, sorted worst-first. Empty state: "No projects currently exceeding typical time
  in stage 🎉".
- Stage multi-select is **not** a frontend control — it reads whatever's flagged
  `includeInVelocity` in taxonomy (edited in Settings → Lists, not on this page). Keeps the
  page itself simple; the "configuration" lives where the other stage flags already live.

### `Settings/TaxonomyPanel.tsx` (existing) — small addition

Add "Include in velocity dashboard" toggle to the `project_stage` `TaxonomySection`, alongside
the existing stage flag toggles.

### Types

`types/projectVelocity.ts` — `IStageStats`, `IStuckProject`, `IProjectVelocityResponse`.

---

## Testing

- Backend: `utils/projectVelocity.test.ts` — unit tests for `computeStageDurations`,
  `computeStageStats`, `findStuckProjects`, `median`/`percentile`, and the "New"-first
  eligibility filter (including edge cases: project with only one log row, project with no log
  rows, project whose first row isn't "New").
- Backend: route test for `GET /api/project-velocity` covering date filtering and
  `includeAbandoned`.
- Frontend: Vitest test for the stage-stats formatting/caption logic and the stuck-projects
  sort order.

---

## Implementation Order

1. Taxonomy: add `includeInVelocity` toggle to `TaxonomySection` (project_stage) in Settings →
   Lists. No backend change needed (existing PATCH already accepts arbitrary `meta`).
2. `utils/projectVelocity.ts` — pure functions + full unit test suite first (this is the part
   worth getting right before wiring up routes/UI).
3. `projectVelocityRoutes.ts` — `GET /api/project-velocity`, mounted in `index.ts`.
4. `types/projectVelocity.ts` on the frontend.
5. `ProjectVelocity.page.tsx` — filter bar, stage chart, stuck-projects table.
6. Sidebar link.
7. Update `ReleasesPanel.tsx` per house rule.

---

## Acceptance Criteria

- [ ] `includeInVelocity` toggle added to project_stage taxonomy sections in Settings → Lists
- [ ] `utils/projectVelocity.ts` with full unit test coverage, including the pre-March
      exclusion filter
- [ ] `GET /api/project-velocity` returns stage stats, stuck projects, and
      `excludedProjectCount`
- [ ] Dashboard page shows a stage chart (median + P25–P75 range) ordered by taxonomy
      `sortOrder`
- [ ] "Based on N projects... M excluded" caption is accurate and visible
- [ ] Stuck-projects table lists projects exceeding their current stage's P75, linking to the
      project
- [ ] Date range + include-abandoned filters both work and are reflected in the caption/counts
- [ ] Projects with no "New"-first log history are excluded from all stats (not just the
      stages they're missing)
- [ ] Sidebar link added; releases entry added

---

## Open Questions

1. **Access control** — proposed: visible to all authenticated staff (no `ensureAdmin`), since
   it's operational/workflow data rather than financial. Confirm before building, or gate it
   admin-only like Financial Overview if preferred.
2. **"Abandoned" detection** — is there a reliable taxonomy signal for "this project left the
   pipeline and isn't coming back" beyond just checking whether its current `status` is a
   non-tracked stage? (e.g. is there already an "Abandoned" stage in your taxonomy, or does this
   need a `meta.isAbandoned` flag added alongside `includeInVelocity`?)
3. **Multiple stage re-entries** — if a project bounces back to an earlier stage (e.g.
   Production → Design for a rework) and then forward again, should both Design intervals count
   toward the Design stats, or only the first? Proposed: count both — a bounce-back is real time
   spent in that stage — but flag this so it's a conscious choice, not an accident.