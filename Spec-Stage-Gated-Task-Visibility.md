# Spec: Stage-Gated Task Visibility

**Feature branch:** `feature/StageGatedTasks`

---

## Overview

Tasks with a type linked to a project stage should only become "actionable" once the project reaches (or passes) that stage. Before that point, the task exists in the DB but is hidden from all action-oriented views and counts. A visual indicator on the task itself signals it's pending a stage transition.

---

## Behaviour Rules

| Scenario | Visible? |
|---|---|
| Task type has no `activationStage` | Always visible |
| Task type has `activationStage`, project is at or past that stage | Visible |
| Task type has `activationStage`, project is before that stage | Hidden from action views |
| Task type has `activationStage`, task has no project (standalone) | Always visible |
| Hidden task — Timesheets tab (project) | Always visible (timesheet entry still valid) |

"At or past" is determined by `sortOrder` on the `TaxonomyItem` for `project_stage`. If the project's current stage has a `sortOrder >= activationStage.sortOrder`, the task is visible.

---

## 1. Data Model Changes

### `TaxonomyItem` — add `activationStage` field

```prisma
model TaxonomyItem {
  // ... existing fields ...
  activationStage   String?   // Only used when type = 'task_type'.
                              // Stores the `name` of a project_stage TaxonomyItem.
}
```

Store the stage `name` (the DB key, not `label`) as a plain string FK-equivalent — consistent with how `Project.status` references taxonomy values. No hard FK; avoids circular dependency and matches the existing pattern.

**Migration name:** `add_activation_stage_to_taxonomy_item`

---

## 2. Backend Changes

### 2a. Taxonomy routes — `taxonomyRoutes.ts`

`PATCH /api/taxonomy/task_type/:id` already accepts arbitrary fields via the existing update handler. Ensure `activationStage` is included in the allowed update payload (add to the Zod/pick schema if one exists, otherwise it passes through).

Validation: if `activationStage` is provided, confirm a `TaxonomyItem` of `type = 'project_stage'` with that `name` exists. Return 400 if not found.

### 2b. Task visibility utility — `src/utils/taskVisibility.ts` (new file)

```typescript
/**
 * Returns true if the task should be visible in action-oriented views.
 *
 * Rules:
 * - No task type → visible
 * - Task type has no activationStage → visible
 * - Task is standalone (no projectId) → visible
 * - Project stage sortOrder >= activationStage sortOrder → visible
 * - Otherwise → hidden
 */
export async function isTaskVisible(
  task: { taskType?: string | null; projectId?: string | null },
  projectStatus: string | null | undefined,
  prisma: PrismaClient
): Promise<boolean>
```

This function is the single source of truth. All routes that filter tasks call it (or its bulk equivalent).

Also export a bulk variant for list queries:

```typescript
export async function filterVisibleTasks<T extends {
  taskType?: string | null;
  projectId?: string | null;
  project?: { status: string } | null;
}>(tasks: T[], prisma: PrismaClient): Promise<T[]>
```

The bulk variant fetches all relevant taxonomy items once (not per-task) to avoid N+1 queries.

### 2c. Routes to update

Apply `filterVisibleTasks` **after** the existing DB query in each of the following. Do not change the DB query itself — keep returning all tasks from Postgres, then filter in application layer so the Timesheets tab (which uses a separate route) is unaffected.

| Route | File |
|---|---|
| `GET /api/tasks/mine` | `taskRoutes.ts` |
| `GET /api/projects/:id/tasks` | `projectRoutes.ts` or `taskRoutes.ts` |
| `GET /api/shopfloor/tasks` | `shopfloorRoutes.ts` |

**Sidebar badge count** is driven by `GET /api/tasks/mine` — filtering there is sufficient, no separate change needed.

### 2d. Include `taskType` on task responses

Ensure every task-list response includes the task's `taskType` string field. It's already on the model — confirm it's included in all `select` / `include` clauses touched above.

### 2e. Expose `activationStage` and `sortOrder` in taxonomy responses

`GET /api/taxonomy/task_type` and `GET /api/taxonomy/project_stage` must include `activationStage` and `sortOrder` respectively. These are already on the model — confirm they're not being excluded by any field-level select.

---

## 3. Frontend Changes

### 3a. Taxonomy editor — `TaxonomyPanel` / `TaxonomySection`

In the `task_type` section, add a **Stage activation** `Select` dropdown to the add/edit modal:

- Options: all active `project_stage` taxonomy items (fetch via `useTaxonomy('project_stage')`)
- Label: **"Show tasks from stage"**
- Placeholder: `"Always visible"`
- Value: the stage `name` string, or `null` to clear
- Renders below the existing fields (colour, label, etc.)

On save, include `activationStage` in the PATCH/POST payload.

### 3b. `useTaxonomy` hook — no changes needed

The hook already caches taxonomy items including all fields. `activationStage` will be available once the backend returns it.

### 3c. Task visibility helper — `src/utils/taskStageVisibility.ts` (new file)

Pure client-side helper used to compute whether a task is "pending" (not yet active) for the purposes of the badge indicator. This runs on already-fetched data — it does not make API calls.

```typescript
/**
 * Returns the stage name that must be reached before this task becomes active,
 * or null if the task is already active / has no gate.
 */
export function getPendingStage(
  task: ITask,
  projectStatus: string | null | undefined,
  taskTypes: ITaxonomyItem[],
  projectStages: ITaxonomyItem[]
): string | null
```

Logic:
1. Look up the task's `taskType` in `taskTypes`. If not found or `activationStage` is null → return `null`.
2. Look up `activationStage` name in `projectStages` to get its `sortOrder`.
3. Look up `projectStatus` in `projectStages` to get its `sortOrder`.
4. If project stage `sortOrder >= activation sortOrder` → return `null` (task is active).
5. Otherwise → return the `label ?? name` of the activation stage (for tooltip text).

### 3d. `TaskItem` component

Import `getPendingStage` and both taxonomy lists (via `useTaxonomy`).

When `getPendingStage` returns a non-null string, render an orange clock icon to the right of the task type badge:

```tsx
// Pseudocode
const pendingStage = getPendingStage(task, project?.status, taskTypes, projectStages);

{pendingStage && (
  <Tooltip label={`Available once this project progresses to ${pendingStage}`}>
    <ThemeIcon color="orange" variant="light" size="sm">
      <IconClock size={12} />
    </ThemeIcon>
  </Tooltip>
)}
```

Use `IconClock` from `@tabler/icons-react` (already in stack).

The task remains rendered — it's shown with the badge, but interactive elements (complete checkbox, start timer, etc.) should be **disabled** when `pendingStage` is non-null.

### 3e. `ShopFloorTaskCard`

Stage-gated tasks are filtered server-side before reaching Shop Floor, so they won't appear. No UI change needed here — the filtering is handled in `shopfloorRoutes.ts`.

### 3f. `TaskModal` — task creation

When creating a task on a project that is currently before the activation stage, no blocking or warning is needed — the badge on `TaskItem` is sufficient feedback once created.

---

## 4. Unit Tests

### Backend — `src/utils/taskVisibility.test.ts`

```
- isTaskVisible: task type with no activationStage → true
- isTaskVisible: task type with activationStage, project at exact stage → true
- isTaskVisible: task type with activationStage, project past stage (higher sortOrder) → true
- isTaskVisible: task type with activationStage, project before stage → false
- isTaskVisible: standalone task (no projectId) with activationStage → true
- filterVisibleTasks: mixed list returns only visible tasks
- filterVisibleTasks: does not N+1 (taxonomy fetched once)
```

### Frontend — `src/utils/taskStageVisibility.test.ts`

```
- returns null when task has no taskType
- returns null when taskType has no activationStage
- returns null when project is at the activation stage
- returns null when project is past the activation stage
- returns stage label when project is before activation stage
- returns stage name when label is absent
- returns null for standalone task (no project status)
```

---

## 5. Migration

```sql
-- Migration: add_activation_stage_to_taxonomy_item
ALTER TABLE "TaxonomyItem" ADD COLUMN "activationStage" TEXT;
```

No data migration required — all existing task types default to `null` (always visible).

---

## 6. Implementation Order

1. Prisma migration + `npx prisma generate`
2. `taskVisibility.ts` utility + tests
3. Update `taskRoutes`, `projectRoutes`, `shopfloorRoutes`
4. Update taxonomy PATCH validation to accept `activationStage`
5. `taskStageVisibility.ts` frontend utility + tests
6. `TaxonomyPanel` editor — add stage dropdown to task_type section
7. `TaskItem` — pending badge + disabled state
8. ReleasesPanel entry

---

## 7. Out of Scope

- Retroactive visibility changes when a project moves *backwards* in stage (edge case — not handled; task visibility recalculates live from current status)
- Per-task override of the stage gate (not needed)
- Notifying assignees when a gated task becomes active (future enhancement)