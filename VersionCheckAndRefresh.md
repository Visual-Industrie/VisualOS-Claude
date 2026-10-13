# #39 — Version Check & Refresh Prompt

## Backend (projects-backend)
- Read `GIT_SHA` from env (fallback `'dev'`).
- Deploy workflow: `GIT_SHA=$(git rev-parse --short HEAD)` passed as a docker build arg → ENV in Dockerfile; add to docker-compose.yml build args.
- Middleware `utils/versionHeader.ts`: sets `X-App-Version` on all responses. Add `X-App-Version` to CORS `exposedHeaders`.
- `GET /api/version` → `{ sha, startedAt }` (no auth).
- Tests: `versionHeader.test.ts` (header present, fallback to 'dev'); route test for `/api/version`.

## Frontend (projects-frontend)
- `vite.config.mjs`: `define: { __APP_VERSION__: JSON.stringify(process.env.GIT_SHA ?? 'dev') }`
  + small plugin emitting `dist/version.json` `{ sha, builtAt }`.
- GH Actions: `GIT_SHA: ${{ github.sha }}` (short to 7 chars).
- Declare `__APP_VERSION__` in `vite-env.d.ts`.
- `utils/version.ts` (pure, unit-tested):
  - `shouldPrompt({ current, latest, dismissed }): boolean` — true if latest !== current, latest !== dismissed, neither 'dev'.
- `hooks/useVersionCheck.ts`:
  - Frontend: fetch `/version.json?t=${Date.now()}` (`cache: 'no-store'`) on visibilitychange + every 5 min.
  - Backend: axios response interceptor records first `x-app-version` seen; prompts on change.
  - `window.addEventListener('vite:preloadError', …)` → prompt.
  - Option `{ autoReload: true }` for Shop Floor.
- Toast via `notifications.show`:
  - `id: 'app-update'` (no stacking), `autoClose: false`, `withCloseButton: true`
  - Title "New version available", message + "Refresh" button → `window.location.reload()`
  - Optional "What's new" link → Settings → Releases
  - On dismiss: store dismissed sha in a ref; re-prompt only for a newer sha.
- Mount in `Shell.page.tsx` and `ShopFloor.page.tsx` (autoReload). NOT in Portal.
- Optional: show the short sha in the sidebar footer / Releases panel.
- Tests: `version.test.ts` (shouldPrompt cases), `useVersionCheck.test.ts` (mock fetch: same sha → no toast; new sha → toast; dismissed sha → no repeat).

## Infra
- Web Station / nginx: `Cache-Control: no-cache` on `index.html` and `version.json`.
  Done in `projects-frontend/public/.htaccess` (Web Station is Apache, so this
  deploys with `dist/` — no manual server config).

## Done when
- [x] Both deploys stamp the sha
- [ ] Toast appears within 5 min (or on refocus) after a FE deploy, on next request after a BE deploy
- [x] Shop Floor auto-reloads
- [x] Tests pass in both repos
- [x] ReleasesPanel.tsx entry

## Built in

- projects-backend **#161** — build stamp, `X-App-Version`, `GET /api/version`
- projects-frontend **#215** — `version.json`, `useVersionCheck`, toast, Shop Floor auto-reload

Merge the backend first: until it is live the header is absent, so the
backend half of the check simply stays quiet (it never prompts on a missing
header).

### Deviations from the spec above

- `hooks/useVersionCheck.ts` is `.tsx` — the toast renders a Refresh button.
- The hook takes an optional `current` sha. `__APP_VERSION__` is `'dev'` under
  vitest, and `'dev'` never prompts, so without an override there is nothing
  the hook tests could assert.
- The backend deploy passes the sha with `docker compose build --build-arg`
  rather than exporting `GIT_SHA`. `sudo` resets the environment, and the
  sudoers rule allows only `/usr/local/bin/docker`, so `sudo -E` and
  `sudo env ...` are both unavailable. `build` and `up` are therefore separate
  steps, since `up --build` takes no `--build-arg`.
- No permission key: the prompt is passive and app-wide, and `/api/version` is
  deliberately open. The one gated piece is the "What's new" link, which the
  Shell offers only to someone holding `settings.tab.releases.view`.

### Verify after the first deploy of each

```
curl https://vis.vil.nz/api/version          # expect a real sha, not 'dev'
curl -sI https://visualos.vil.nz/version.json | grep -i cache-control
curl -s  https://visualos.vil.nz/version.json
```

A `'dev'` sha in production means the build arg did not reach the image, and
nothing will ever prompt.