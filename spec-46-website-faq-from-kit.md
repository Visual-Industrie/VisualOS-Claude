# Spec #46 — Website FAQ Page Fed from Kit's Knowledge Pack

**Status:** 📋 Specced — ready to build
**Priority:** High
**Labels:** `feature`, `website`, `wordpress`, `api`, `webhook`, `faq`
**Branches:** `feature/PublicKnowledgeApi` (projects-backend, plus a small frontend status line), and a `kit-faq` plugin in the website repo
**Deliberately separate from:** #43 (Website Chatbot). This needs no AI and no API credits, so it can ship now.

---

## Goal

Kit's knowledge pack (`KnowledgeEntry`, shipped under #43) is the single source of truth. The public website (beta.vil.nz, WordPress) shows a plain FAQ page built from the same published entries. Nobody maintains a second copy. When the chatbot ships, it reads the same entries, so an answer edited once in Kit updates both.

```
Settings → Chatbot → Knowledge
        │ save / reorder / delete / approve suggestion
        ▼
KnowledgeEntry ──► notifyKnowledgeUpdated() ──signed ping──► WP: POST /wp-json/kit/v1/refresh
        ▲                                                         │
        └──────────── GET /api/public/knowledge ◄─────── re-fetch ┘
                                                                  │
                                                     cache in wp_options
                                                                  │
                                                         [kit_faq] shortcode
```

### Already in place (no work needed)

- `KnowledgeEntry` model, `knowledge_category` taxonomy, authed CRUD at `/api/knowledge` (`knowledgeRoutes.ts`)
- Settings → Chatbot → Knowledge UI, `settings.tab.chatbot.view/.edit`
- 47 imported entries (two unpublished pending a decision)
- `/api/public` router (`publicRoutes.ts`, currently serves `/notice`), mounted before auth in `mountRoutes.ts`
- `buildKnowledgePack()` in `utils/knowledgePack.ts` — groups published entries by category for the chatbot prompt

### Non-goals

- Search, "was this helpful?" voting or comments on the FAQ page
- Editing FAQs from WordPress. WordPress is read-only; Kit is the editor.
- Anything AI

---

## Part A — Kit backend

### `GET /api/public/knowledge`

Add to the existing `publicRoutes.ts`.

- No auth, read-only, **published entries only**. Select only the fields below — never `updatedById`, `source`, `sourceSuggestionId` or anything internal.
- Rate limited (60 / IP / hour via `express-rate-limit`)
- `Cache-Control: public, max-age=300`
- `ETag` from the newest `updatedAt` + entry count + newest category change; honours `If-None-Match` → 304
- No CORS header needed: WordPress calls it server-to-server. If a browser client is ever added, lock it to `PUBLIC_ALLOWED_ORIGINS` (the `publicOrigin` util planned in #43).

**Response:**

```json
{
  "updatedAt": "2026-10-20T01:23:45.000Z",
  "categories": [
    {
      "slug": "services",
      "label": "Services",
      "entries": [
        { "id": "uuid", "title": "Do you do vehicle wraps?", "bodyHtml": "<p>…</p>", "bodyMarkdown": "…" }
      ]
    }
  ]
}
```

- Categories in taxonomy `sortOrder`, entries in `sortOrder`; empty and archived categories omitted
- `bodyHtml` rendered from markdown and **sanitised** (e.g. `marked` + `sanitize-html`; allow p, a[href], strong, em, ul, ol, li, br). WordPress then needs no markdown parser, and runs it through `wp_kses_post` as well.
- Pure util `utils/publicKnowledge.ts` → `buildPublicKnowledge(entries, categories)`. Reuse the grouping from `buildKnowledgePack()` where it fits rather than duplicating it.

### Webhook to WordPress

`utils/knowledgeWebhook.ts` → `notifyKnowledgeUpdated()`, called after every successful write that can change what's public:

- `knowledgeRoutes.ts`: `POST /`, `POST /reorder`, `PATCH /:id`, `DELETE /:id`
- FAQ suggestion approve (creates or merges an entry)
- `knowledge_category` taxonomy create / rename / archive / reorder in `taxonomyRoutes`

Behaviour:

- **Debounced 10 seconds**, so a drag-reorder of 20 items sends one webhook, not 20
- Payload is a ping, not the data: `{ "event": "knowledge.updated", "updatedAt": "…" }`. WordPress always pulls from the public route, so there's one read path to trust.
- Headers `X-Kit-Timestamp` and `X-Kit-Signature` = hex HMAC-SHA256 of `timestamp + "." + rawBody` with `KNOWLEDGE_WEBHOOK_SECRET`
- Target from env `KNOWLEDGE_WEBHOOK_URL` (e.g. `https://beta.vil.nz/wp-json/kit/v1/refresh`). Unset = do nothing (dev).
- Fire-and-forget with one retry after 30s. Failures are logged and never block or fail the save.
- Store the outcome on `SystemSettings`: `knowledgeWebhookLastAt DateTime?`, `knowledgeWebhookLastOk Boolean?` (migration)
- `GET /api/knowledge` response gains `website: { lastNotifiedAt, ok, configured }`

**New env vars:** `KNOWLEDGE_WEBHOOK_URL`, `KNOWLEDGE_WEBHOOK_SECRET` — backend `.env`, the NAS, **and the `environment:` list in `docker-compose.yml`** (new vars in `.env` are silently dropped until they're added there — the #42 gotcha).

## Part B — Kit frontend

- Knowledge section header shows "Website updated 2 min ago ✓", "Website update failed ✗" or "Website sync not configured", from the `website` field
- Pure helper `formatWebsiteSyncStatus()`

## Part C — WordPress plugin (`kit-faq`, website repo)

Small and theme-independent.

**Settings → Kit FAQ**
- Kit API base (default `https://vis.vil.nz/api`)
- Webhook secret, or a `KIT_FAQ_WEBHOOK_SECRET` constant in `wp-config.php`, which wins if set
- Last synced time + status, and a **Resync now** button

**Cache**
- `wp_options` key `kit_faq_cache` (`autoload = no`): the API response, `fetchedAt` and `etag`
- On fetch failure, **keep the existing cache**. The page never goes blank because Kit is down.
- Daily WP-cron re-fetch as a fallback (conditional request with the stored ETag) in case a webhook is missed
- Fetch once on activation

**Webhook endpoint** — `POST /wp-json/kit/v1/refresh` via `register_rest_route` in `rest_api_init`
- `permission_callback` verifies the HMAC with `hash_equals` and rejects timestamps more than 5 minutes old (replay protection)
- Valid: re-fetch, update the cache, purge the FAQ page from any page-cache plugin, return 200
- Bad signature → 401, no work done

**Rendering**
- Shortcode `[kit_faq]`, optional `category="services"` for a single category
- Category headings; each entry is a native `<details><summary>` accordion (accessible, no JS)
- **FAQPage JSON-LD** structured data for search engines
- Minimal CSS that inherits the theme's colours and fonts

---

## Testing

### projects-backend (Vitest)

| File | Covers |
|---|---|
| `utils/publicKnowledge.test.ts` | Published only; category and entry ordering; empty and archived categories dropped; markdown → sanitised HTML (`<script>`, `onerror=` stripped); no internal fields in output |
| `utils/knowledgeWebhook.test.ts` | Signature matches a known vector; debounce collapses a burst into one call; unset URL → no call; failure logs and doesn't throw; retry once |
| `routes/publicRoutes.test.ts` | Extend: `/knowledge` needs no auth, ETag → 304, unpublished never leaks, rate limit |
| `routes/knowledgeRoutes.test.ts` | Extend: each write calls `notifyKnowledgeUpdated`; failed writes don't; `website` status in the list response |

### projects-frontend (Vitest)
- `formatWebsiteSyncStatus` — never notified, ok, failed, not configured

### WordPress plugin
- PHPUnit (WP test scaffold): signature valid / wrong secret / stale timestamp; cache kept on fetch failure; shortcode output for a fixture response, including the JSON-LD

---

## PR plan

| # | Repo | Branch | Scope |
|---|---|---|---|
| 1 | projects-backend + frontend | `feature/PublicKnowledgeApi` | Public route, `publicKnowledge` util, webhook notifier + debounce, hooks into the knowledge / suggestion / taxonomy writes, settings fields + migration, env vars + compose list, status line in the Knowledge section |
| 2 | website | `kit-faq` plugin | Settings, cache, webhook endpoint, shortcode, JSON-LD, daily fallback |

PR 1 includes its `ReleasesPanel.tsx` entry.

**Rollout:** publish or review the two held entries in Kit → merge PR 1, set env vars on the NAS → install the plugin on beta.vil.nz, set the same secret, add `[kit_faq]` to an FAQ page → edit an entry in Kit and confirm the page updates within a minute.

## Multi-tenancy (#32)

The public route becomes per-org (API key or subdomain). The webhook URL and secret move from env to per-org settings.
