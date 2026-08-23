# VisualOS — Verifone Transactions Feature Spec

**Status:** 🟡 Ready to Build  
**Priority:** Medium  
**Labels:** `feature`, `verifone-api`, `admin-only`, `reporting`  
**Last Updated:** March 2026

---

## Overview

Add an admin-only Verifone Transactions screen to VisualOS. The backend polls the Verifone Reporting API for scheduled daily reports, downloads each unprocessed report (raw CSV), parses it, and stores each transaction row in a local database table. Admins can then view and filter transactions by date range via a clean table UI.

This screen is entirely read-only — no writes back to Verifone.

---

## Scope

- New env variables: `VERIFONE_USER_UUID`, `VERIFONE_API_KEY`
- Prisma schema: `VerifoneReport` and `VerifoneTransaction` models
- Backend service: `verifoneService.ts` — list reports (RSQL filter), identify unprocessed, download CSV, parse, upsert
- Backend route: `POST /api/verifone/sync` — trigger manual sync (admin only)
- Backend route: `GET /api/verifone/transactions` — query transactions by date range (admin only)
- Scheduled daily sync via `node-cron`
- Frontend: `/verifone` admin-only page with date range picker and transaction table
- Sidebar nav link visible to admins only

## What We Are NOT Doing

- No writing transactions back to Verifone
- No refund or void actions
- No non-admin access
- No real-time webhook integration (scheduled reports only)
- No per-terminal breakdown (not in scope for v1)

---

## Verifone API

### Base URL (NZ Production)

```
https://nz.gsc.verifone.cloud/oidc/report-engine/api/v1
```

### Authentication — `ApiKeyAuth` (HTTP Basic)

The `ApiKeyAuth` scheme uses HTTP Basic Authentication. The credential is `UserUUID:APIKEY` encoded as base64.

**Construction:**
1. Concatenate: `<VERIFONE_USER_UUID>:<VERIFONE_API_KEY>` (e.g. `f8811dd2-3667-4a1e-be8b-b42b63743254:ABCheDEF...`)
2. Base64-encode the full string
3. Send as: `Authorization: Basic <base64string>`

**Store in `.env`:**
```
VERIFONE_USER_UUID=your-user-uuid-here
VERIFONE_API_KEY=your-api-key-here
```

**Helper in `verifoneService.ts`:**
```typescript
function getAuthHeader(): string {
  const credential = `${process.env.VERIFONE_USER_UUID}:${process.env.VERIFONE_API_KEY}`;
  return `Basic ${Buffer.from(credential).toString('base64')}`;
}
```

---

### Endpoint 1 — List Reports (`GET /reports`)

Returns metadata for reports matching an RSQL filter. Paginated.

**Query parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `pageNumber` | integer | 1 | Page number (min: 1) |
| `pageSize` | integer | 50 | Results per page (max: 1000) |
| `orderBy` | string | `DESC` | `ASC` or `DESC` |
| `orderCriteria` | string | `createdOn` | Sort field: `createdOn`, `modifiedOn`, `status`, `mimeType`, `reportType`, `reportUid` |
| `search` | string | — | RSQL filter expression (see below) |

**RSQL filter — key criteria for this feature:**

| Criteria | Description | Operators |
|---|---|---|
| `createdOn` | When the report record was created | `==`, `=lt=`, `=le=`, `=gt=`, `=ge=`, `=in=`, `=out=` |
| `reportType` | Report type enum | `==`, `=in=`, `=out=` |
| `status` | Report status | `==`, `=in=`, `=out=` |
| `reportParameter.reportableDay` | The business date the report covers | `==`, `=lt=`, `=le=`, `=gt=`, `=ge=` |

RSQL logical operators: `;` or `&` for AND, `,` for OR.

**Example search strings:**
```
# All successful CSV daily transaction reports
reportType==DAILY_TRANSACTION_REPORT;status==SUCCESSFUL

# Reports for a specific business day
reportType==DAILY_TRANSACTION_REPORT;status==SUCCESSFUL;reportParameter.reportableDay==2026-03-05

# Reports created after a given date
reportType==DAILY_TRANSACTION_REPORT;status==SUCCESSFUL;createdOn=gt=2026-03-01T00:00:00.000Z
```

**Response schema (`ReportsResponse`):**
```typescript
interface ReportsResponse {
  totals: number;           // total matching records (may exceed current page)
  reports: ReportRecord[];
}

interface ReportRecord {
  reportUid: string;        // UUID — used to download the report
  reportType: string;       // e.g. 'DAILY_TRANSACTION_REPORT'
  reportEntityUid: string;  // UUID of the merchant entity
  fileName: string;         // e.g. 'daily_report_cb941e2d-....csv'
  mimeType: string;         // 'text/csv' | 'application/pdf' | 'text/plain'
  status: 'INITIATED' | 'PROCESSING_STARTED' | 'SUCCESSFUL' | 'FAILED' | 'INVALID_INPUT_PARAMETERS';
  createdOn: string;        // ISO 8601 datetime
  modifiedOn: string;       // ISO 8601 datetime
  reportParameter: {
    reportableDay?: string; // date string — the business day this report covers
    reportDescription?: string;
    clearingDate?: string;
    // ... other parameters
  };
}
```

> **Implementation note:** Filter for `status == 'SUCCESSFUL'` and `mimeType === 'text/csv'` in the RSQL search. Skip any other status or mime type.

---

### Endpoint 2 — Download Report (`GET /reports/{reportUid}`)

Downloads the raw content of a specific report.

**Path parameter:** `reportUid` (UUID) — taken from `ReportRecord.reportUid`.

**Response:** Raw `text/csv` content. No JSON wrapper — the response body is the CSV string directly.

**Example request:**
```
GET https://nz.gsc.verifone.cloud/oidc/report-engine/api/v1/reports/04f5ea24-cc4f-11e8-a8d5-f2801f1b9fd1
Authorization: Basic Zjg4MTFkZD...
```

---

## CSV Format

The exact column names depend on your scheduled report configuration in Verifone Central. The fields you want to capture are:

| CSV Column (expected) | Stored as | Notes |
|---|---|---|
| `created_at_date` | part of `transactionDate` | e.g. `2026-03-05` |
| `created_at_time` | part of `transactionDate` | e.g. `14:32:07` |
| `masked_card_number` | `maskedCard` | e.g. `****1234` |
| `Orig.amount` | `origAmount` | Strip currency symbols + commas before parsing |
| `Surcharge_amount` | `surchargeAmount` | Strip currency symbols + commas before parsing |

> **Critical:** On first deployment, download one report manually and log the exact header row before running a full sync. Column names in Verifone CSV exports are case-sensitive and may differ from documentation. Update the column mappings in `verifoneService.ts` to match actual headers.

---

## Data Model

### `VerifoneReport`

Tracks which reports have been fetched and processed. Acts as the deduplication guard.

```prisma
model VerifoneReport {
  id                Int      @id @default(autoincrement())

  reportUid         String   @unique  // Verifone reportUid (UUID) — deduplication key
  reportType        String?           // e.g. 'DAILY_TRANSACTION_REPORT'
  reportableDay     String?           // from reportParameter.reportableDay (date string)
  fileName          String?
  mimeType          String?
  verifoneStatus    String?           // Verifone's status: SUCCESSFUL, FAILED, etc.
  verifoneCreatedOn DateTime?         // createdOn from Verifone

  processedAt       DateTime?         // null = not yet parsed; set on success
  rowCount          Int?              // number of transactions parsed
  errorMessage      String?           // set if parsing failed; cleared on successful retry

  transactions      VerifoneTransaction[]

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```

### `VerifoneTransaction`

One row per transaction line from the CSV.

```prisma
model VerifoneTransaction {
  id               Int      @id @default(autoincrement())

  reportId         Int
  report           VerifoneReport @relation(fields: [reportId], references: [id], onDelete: Cascade)

  transactionDate  DateTime         // combined created_at_date + created_at_time, stored as UTC
  maskedCard       String?
  origAmount       Decimal?  @db.Decimal(10, 2)
  surchargeAmount  Decimal?  @db.Decimal(10, 2)

  rawData          Json?             // full parsed CSV row — safety net for future fields

  createdAt        DateTime @default(now())

  @@index([transactionDate])         // for efficient date-range queries
}
```

> **Design decisions:**
> - `transactionDate` is a proper `DateTime` combining the date + time CSV columns. Prisma `gte`/`lte` date range queries work cleanly against this.
> - `rawData` stores the full original CSV row as JSON. Zero cost, and gives you access to any extra fields later without a migration.
> - `reportableDay` is stored as a plain string on `VerifoneReport` (matching Verifone's format) — useful for cross-referencing which business day was processed.

---

## Backend Implementation

### File Structure

```
src/
  routes/
    verifoneRoutes.ts       # Express routes: sync + query transactions
  services/
    verifoneService.ts      # Core logic: list, filter, download, parse, upsert
  utils/
    csvParser.ts            # CSV string → array of row objects
```

### `verifoneService.ts` — Pseudocode

```typescript
const VERIFONE_BASE_URL = 'https://nz.gsc.verifone.cloud/oidc/report-engine/api/v1';

function getAuthHeader(): string {
  const credential = `${process.env.VERIFONE_USER_UUID}:${process.env.VERIFONE_API_KEY}`;
  return `Basic ${Buffer.from(credential).toString('base64')}`;
}

// listAllReports(): paginate GET /reports with RSQL filter
//   search = 'reportType==DAILY_TRANSACTION_REPORT;status==SUCCESSFUL'
//   pageSize = 1000, increment pageNumber until page results < pageSize
//   Returns all matching ReportRecord[]
async function listAllReports(): Promise<ReportRecord[]> { ... }

// fetchAndStoreNewReports(): main entry point
//   1. Call listAllReports()
//   2. For each report where mimeType === 'text/csv':
//      a. Look up VerifoneReport by reportUid
//      b. If found AND processedAt is set → skip (idempotent)
//      c. Upsert VerifoneReport (clear errorMessage on retry)
//      d. GET /reports/{reportUid} → raw CSV string
//      e. parseCSV(rawCsv) → row objects
//      f. Map rows → VerifoneTransaction data (parseAmount, combineDateTime)
//      g. prisma.$transaction:
//           - deleteMany transactions for this reportId (safe re-run)
//           - createMany new transactions
//           - update VerifoneReport: processedAt = now(), rowCount, errorMessage = null
//      h. On error: update VerifoneReport: errorMessage = err.message (leave processedAt null)
//   3. Return { processed, skipped, errors }
export async function fetchAndStoreNewReports(): Promise<SyncResult> { ... }

// parseAmountString(): strip '$', ',', whitespace → parseFloat → null if NaN
function parseAmountString(value: string): number | null { ... }

// combineDateTime(): '2026-03-05' + '14:32:07' → Date in UTC (apply NZ timezone offset)
function combineDateTime(dateStr: string, timeStr: string): Date { ... }
```

### `csvParser.ts`

```typescript
import { parse } from 'csv-parse/sync';

export function parseCSV(rawCsv: string): Record<string, string>[] {
  return parse(rawCsv, {
    columns: true,          // first row becomes object keys
    skip_empty_lines: true,
    trim: true,
    bom: true,              // handle BOM characters if present in Verifone exports
  });
}
```

Install: `npm install csv-parse` — remember to run both locally and inside the container per CLAUDE.md.

### Routes — `verifoneRoutes.ts`

All routes protected by `ensureAuthenticated` + `ensureAdmin`.

```
POST /api/verifone/sync
  Body: none
  Triggers fetchAndStoreNewReports()
  Returns: { processed: number, skipped: number, errors: number }

GET /api/verifone/transactions
  Query params:
    from       — ISO date string, required (e.g. '2026-03-01')
    to         — ISO date string, required (e.g. '2026-03-31')
    page       — integer, default 1
    pageSize   — integer, default 50
  Filters: transactionDate >= start of `from` day AND <= end of `to` day
  Order: transactionDate DESC
  Returns:
    {
      transactions: [{ id, transactionDate, maskedCard, origAmount, surchargeAmount, report: { reportUid, reportableDay } }],
      total: number,
      page: number,
      pageSize: number
    }
```

### Scheduled Daily Sync

```typescript
// In index.ts, after app setup:
import cron from 'node-cron';
import { fetchAndStoreNewReports } from './services/verifoneService';

// Adjust to run after Verifone has generated the previous day's reports
// Confirm the exact generation time in your Verifone Central Report Scheduler
// 18:00 UTC ≈ 6am–7am NZT (safe default — adjust once confirmed)
cron.schedule('0 18 * * *', async () => {
  console.log('[Verifone] Running scheduled daily sync...');
  try {
    const result = await fetchAndStoreNewReports();
    console.log('[Verifone] Sync complete:', result);
  } catch (err) {
    console.error('[Verifone] Sync failed:', err);
  }
});
```

Install: `npm install node-cron` + `npm install --save-dev @types/node-cron`.

---

## Frontend Implementation

### Route

```
/verifone   → VerifoneTransactionsPage (admin only)
```

Redirect non-admins to `/` on mount (same pattern as other admin-gated pages).

### Sidebar Nav

Add a "Transactions" link in the admin section of the sidebar, rendered only when `user.isAdmin === true`.

### `VerifoneTransactionsPage`

**Layout:**

```
┌─────────────────────────────────────────────────────────────┐
│  Verifone Transactions                    [Sync Now]         │
│                                                              │
│  From [date picker]   To [date picker]   [Apply]             │
│                                                              │
│  Date        Time      Card        Amount    Surcharge       │
│  05 Mar 26   14:32:07  ****1234    $125.00   $3.75           │
│  05 Mar 26   11:14:22  ****5678    $89.50    $2.69           │
│  ...                                                         │
│                                                              │
│                              [Pagination]                    │
└─────────────────────────────────────────────────────────────┘
```

**Date pickers:** Mantine `DatePickerInput` — default range is last 7 days on mount. "Apply" re-fetches with the new range.

**Sync Now button:**
- `POST /api/verifone/sync`
- Shows Mantine `Loader` while in-flight
- On success: toast "Sync complete — X report(s) processed"
- On error: error toast

**Transaction table columns:**

| Column | Value | Format |
|---|---|---|
| Date | `transactionDate` | `DD MMM YYYY` |
| Time | `transactionDate` | `HH:mm:ss` |
| Card | `maskedCard` | Monospace |
| Amount | `origAmount` | Right-aligned, `$0.00` |
| Surcharge | `surchargeAmount` | Right-aligned, `$0.00` |

**Loading state:** Mantine `Skeleton` rows during fetch.  
**Empty state:** "No transactions found for the selected date range."

---

## Environment Variables

Add to backend `.env` and `.env.example`:

```
VERIFONE_USER_UUID=your-verifone-user-uuid
VERIFONE_API_KEY=your-verifone-api-key
```

---

## Migration Plan

1. Add `VERIFONE_USER_UUID` + `VERIFONE_API_KEY` to `.env` and document in `.env.example`
2. Install packages: `csv-parse`, `node-cron`, `@types/node-cron` (local + docker container)
3. Prisma migration: `npx prisma migrate dev --name add_verifone_tables`
4. Deploy backend — cron starts automatically
5. Download one report manually from Verifone Central and log the header row — confirm CSV column names before running full sync
6. Update column mappings in `verifoneService.ts` to match actual headers
7. Trigger a manual backfill: `POST /api/verifone/sync`
8. Verify rows in Prisma Studio
9. Deploy frontend

> **Backfill scope:** The initial sync will pull all `SUCCESSFUL` daily transaction reports available in your account. If you have extensive history, add a `createdOn=gt=<date>` RSQL filter to limit scope. The daily cron handles everything going forward.

---

## Unit Tests

### Backend (`verifoneService.test.ts`)

- [ ] `getAuthHeader()` returns correct `Basic <base64>` string from UUID + API key
- [ ] `listAllReports()` paginates correctly when `totals > pageSize`
- [ ] `fetchAndStoreNewReports()` skips reports where `processedAt` is already set
- [ ] `fetchAndStoreNewReports()` creates `VerifoneReport` + `VerifoneTransaction` rows for a new report
- [ ] `fetchAndStoreNewReports()` retries a previously failed report (clears `errorMessage`, sets `processedAt`)
- [ ] `fetchAndStoreNewReports()` sets `errorMessage` and leaves `processedAt` null on CSV parse error
- [ ] `fetchAndStoreNewReports()` skips non-CSV mime types
- [ ] `parseAmountString()` strips `$` and `,` and returns correct float
- [ ] `parseAmountString()` returns `null` for empty or non-numeric input
- [ ] `combineDateTime()` returns correct UTC `Date` from NZ local date + time strings
- [ ] `parseCSV()` correctly parses a multi-row CSV string using headers as keys

### Backend (`verifoneRoutes.test.ts`)

- [ ] `POST /api/verifone/sync` returns 403 for non-admin users
- [ ] `POST /api/verifone/sync` returns 200 with sync result for admin users
- [ ] `GET /api/verifone/transactions` returns 403 for non-admin users
- [ ] `GET /api/verifone/transactions` returns 400 if `from` or `to` are missing
- [ ] `GET /api/verifone/transactions` returns only rows within the date range
- [ ] `GET /api/verifone/transactions` paginates correctly

### Frontend (`VerifoneTransactionsPage.test.tsx`)

- [ ] Page redirects non-admins to `/`
- [ ] Date pickers default to last 7 days on mount
- [ ] "Sync Now" button shows loading state during request
- [ ] Table renders correct columns from mock API response
- [ ] Empty state shown when API returns zero transactions

---

## Acceptance Criteria

- [ ] `VERIFONE_USER_UUID` and `VERIFONE_API_KEY` documented in `.env.example`
- [ ] Prisma migrations for `VerifoneReport` and `VerifoneTransaction` pass cleanly
- [ ] `VerifoneReport.reportUid` has a unique index (deduplication)
- [ ] Auth header constructed correctly as `Basic base64(UUID:APIKEY)`
- [ ] RSQL search correctly targets `DAILY_TRANSACTION_REPORT` + `status==SUCCESSFUL`
- [ ] List endpoint paginates until all matching reports are retrieved
- [ ] Already-processed reports are skipped (idempotent)
- [ ] Failed reports retry on next sync
- [ ] Daily cron triggers at configured time without manual intervention
- [ ] `GET /api/verifone/transactions` returns correct paginated results for a date range
- [ ] All routes return 403 for non-admin users
- [ ] Frontend page redirects non-admins
- [ ] Date range filter and table render correctly in UI
- [ ] "Sync Now" button triggers sync and shows result toast
- [ ] No TypeScript errors, ESLint clean per project config
- [ ] Unit tests written and passing

---

## Open Questions

1. **CSV column names:** Download one report manually from Verifone Central before implementing the parser. Log the exact header row and update the column mappings in `verifoneService.ts` accordingly before running a full sync.

2. **Timezone of CSV timestamps:** Confirm whether the date/time values in the CSV are NZT or UTC. `combineDateTime()` needs to apply the correct offset when constructing the stored `DateTime`.

3. **Cron timing:** Confirm when Verifone generates your daily reports (check the Report Scheduler in Verifone Central). Set the cron to run at least 30 minutes after that time.

4. **`reportEntityUid`:** The API spec notes that if `reportEntityUid` is not included in the RSQL filter, only reports for the current user's assigned entity are returned (not descendants). For a single-entity setup this should be fine — confirm whether you need to explicitly include it.

5. **Backfill scope:** Decide how far back to pull history on the first manual sync. You can limit with `createdOn=gt=<date>` in the RSQL filter if needed.