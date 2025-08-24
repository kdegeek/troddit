# Troddit PWA + Server-side Preferences Roadmap

## Goals
- Elevate PWA capabilities: offline support, runtime caching, install UX, and predictable updates.
- Persist user preferences server-side (while still working offline), with seamless merge/sync.
- Improve perceived performance and resilience under poor connectivity.

## Scope
- PWA: service worker runtime caching, offline fallback, manifest improvements, optional background sync.
- Data: server storage for user preferences using self-hosted Postgres.
- Client: load/merge/sync preferences, persist query cache for offline reading.
- UX: connectivity awareness and gentle fallbacks.

---

## 1) PWA Enhancements

### 1.1 Runtime caching
- Add `runtimeCaching` to `next-pwa` config in `next.config.js`:
  - Cache Reddit APIs: `https://www.reddit.com/*` and `https://oauth.reddit.com/*` via NetworkFirst.
  - Cache Reddit media: `i.redd.it`, `preview.redd.it`, `external-preview.redd.it`, `*.thumbs.redditmedia.com`, `v.redd.it` via CacheFirst.
  - Cache HLS segments (`.m3u8`, `.ts`) via StaleWhileRevalidate.
  - Cache navigations (document shell) via NetworkFirst with timeout.
- Enable `skipWaiting: true` to activate updates promptly.
- Set `fallbacks: { document: "/fallback.html" }`.

### 1.2 Offline fallback page
- Add `public/fallback.html` with a simple, lightweight offline message and a link to `/`.

### 1.3 Manifest improvements
- Update `public/manifest.json`:
  - Add `scope`, `categories`: `["social","news"]`.
  - Add `shortcuts` for quick access: Home `/`, Search `/search`, My subs `/subreddits`.
  - Keep `display: "standalone"`, `theme_color`, `background_color`.

### 1.4 Image/CDN domains
- Populate `images.domains` in `next.config.js`:
  - `i.redd.it`, `preview.redd.it`, `external-preview.redd.it`, `a.thumbs.redditmedia.com`, `b.thumbs.redditmedia.com`, `styles.redditmedia.com`, `i.imgur.com`, `v.redd.it`.

### 1.5 Head optimizations
- In `src/pages/_document.tsx` `<Head>` add:
  - `preconnect` to `https://i.redd.it`, `https://preview.redd.it`, `https://www.reddit.com`.
  - `dns-prefetch` for `https://v.redd.it`.

### 1.6 Optional: Background Sync
- If needed, add custom Workbox setup with BackgroundSync queue for POST actions (vote/save/comment):
  - Queue name: `troddit-actions`.
  - Replay on regain connectivity.
  - Requires `next-pwa` `swSrc` and a custom SW file.

---

## 2) Persist React Query Cache Offline

### 2.1 Persistor
- Add `@tanstack/react-query-persist-client` and `@tanstack/query-sync-storage-persister`.
- Use `localforage` as storage.
- Wrap providers inside `PersistQueryClientProvider` in `src/pages/_app.tsx`.
- Set a sensible `maxAge` (e.g., 5–15 minutes) to avoid stale data.

### 2.2 Cache strategy
- Persist for feed and thread queries to allow basic reading while offline.
- Keep mutation handling as-is; pair with background sync later if necessary.

---

## 3) Server-side User Preferences (self-hosted Postgres)

### 3.1 Storage engine
- Use Postgres (e.g., `postgres:15-alpine`) self-hosted via Docker Compose.
- App connects over the internal compose network using service name `db`.
- Node library: `pg` (lightweight). Optionally Prisma if you prefer an ORM.

### 3.2 Schema
```sql
create table if not exists user_prefs (
  user_id text primary key,
  data jsonb not null,
  updated_at timestamptz not null default now()
);
```

### 3.3 API endpoint
- Create `src/pages/api/user/prefs.ts`:
  - `GET`: returns `{}` if none exist.
  - `POST`: upserts JSON prefs for the user.
  - Identify user via NextAuth Reddit username (`session.user.name`) or Clerk ID; fallback to `401` if missing.
  - Use a shared `pg` Pool; keep queries parameterized.
  - Validate payload size (e.g., < 32KB) to avoid abuse.

### 3.4 Minimal DB client
- File: `src/server/db.ts`
- Expose a singleton `Pool` from `pg` initialized with `process.env.DATABASE_URL` OR discrete envs.
- Ensure `ssl` off/on per deployment needs (off for local Docker network by default).

### 3.5 Docker & persistence
- Extend `docker-compose.yml` to add Postgres and a named volume:
```yaml
version: '3'
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: troddit
      POSTGRES_PASSWORD: troddit
      POSTGRES_DB: troddit
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U troddit -d troddit"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      dockerfile: Dockerfile
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://troddit:troddit@db:5432/troddit
    ports:
      - "3000:3000"

volumes:
  pgdata:
```

### 3.6 Env vars
- `DATABASE_URL=postgres://USER:PASS@HOST:PORT/DB` (recommended)
  - Or discrete vars: `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`.

### 3.7 Migration/init
- Option A: One-off `psql` against the `db` service to run the schema.
- Option B: Add an init script mounted at `/docker-entrypoint-initdb.d/` for Postgres.
- Option C: Run a tiny Node script at app start to `create table if not exists`.

---

## 4) Client Preference Sync

### 4.1 Collect prefs
- In `src/MainContext.tsx`, create a compact object of persistable settings (booleans/numbers/strings), e.g.:
  - `nsfw`, `autoplay`, `hoverplay`, `volume`, `wideUI`, `saveWideUI`, `syncWideUI`, `postWideUI`, `mediaOnly`, `cardStyle`, `autoCollapseComments`, `collapseChildrenOnly`, `defaultSortComments`, etc.
- Avoid large/ephemeral data (e.g., massive caches); those can stay local.

### 4.2 Hook for load/merge/save
- New hook `src/hooks/useUserPrefs.ts`:
  - On mount, `GET /api/user/prefs`, merge with local (`localforage.getItem("userPrefs")`), then set local with merged result.
  - Watch selected prefs; debounce 500ms; `POST` merged prefs to server and save to localforage.
  - If offline, best-effort save to local; server sync will occur when back online (or via Background Sync if implemented).

### 4.3 Merge precedence
- Server overwrites local on first load (assume server is source of truth if present).
- Afterwards, local changes are sent upstream; last-write-wins.

---

## 5) UX and Connectivity

### 5.1 Offline page & UI cues
- Show a simple banner/toast when offline.
- Disable actions that require live Reddit POSTs, or queue them if Background Sync is active.
- Keep optimistic UI for votes but mark as “pending” if queued.

### 5.2 Performance
- Lazy-load heavy components (`next/dynamic`) where helpful.
- Consider lowering initial data payload where possible.

---

## 6) Security & Privacy

- Do not store Reddit access tokens or PII in prefs.
- Keep Supabase service role key on server only.
- If moving to RLS with end-user JWTs, implement robust claim checks for `user_id`.
- Avoid over-collecting preferences; store only what is necessary.

---

## 7) Testing Plan

### 7.1 Unit/Integration
- API: `GET/POST /api/user/prefs` with/without auth.
- Client hook: merges correctly, debounces saves, handles offline.
- MainContext: initial load waits on prefs when user is authenticated.

### 7.2 E2E Scenarios
- First-time user: local defaults only, then sign-in → server sync.
- Returning user: server prefs load and override local.
- Offline:
  - Load cached feed and navigate pages.
  - Attempt vote/save/comment → disabled or queued; resume online → replay.
  - View fallback page on navigation miss.

### 7.3 Lighthouse
- Run PWA audits on desktop/mobile; aim for:
  - Installable = true
  - Offline page = working
  - Service worker controlling the page

---

## 8) Rollout Plan

- Phase 1: PWA runtime caching + fallback + manifest; React Query persistence.
- Phase 2: Server prefs API + client sync for a safe subset of prefs.
- Phase 3 (optional): Background Sync for mutations.
- Staged rollout:
  - Feature flag background sync.
  - Monitor Supabase writes and error rates.
- Rollback:
  - Disable `next-pwa` via `disable: true` if needed.
  - Revert API route to return local-only if Supabase unavailable.

---

## 9) Acceptance Criteria

- PWA:
  - [ ] App passes Lighthouse PWA installability.
  - [ ] Offline fallback renders when offline on fresh navigation.
  - [ ] Media/APIs are cached per strategy.
  - [ ] SW updates apply on next navigation (skipWaiting enabled).

- Preferences:
  - [ ] Authenticated user’s prefs persist across devices.
  - [ ] Local defaults apply when server prefs are absent.
  - [ ] Changes debounce and save without UI jank.
  - [ ] Works when offline (local persistence), syncs on reconnect.

- React Query:
  - [ ] Feed/comments viewable from cache after reload offline.
  - [ ] Cache ages out per configuration.

---

## 10) Effort & Timeline (est.)

- PWA runtime caching + fallback + manifest: 0.5–1 day
- React Query persistence: 0.5 day
- API route + Supabase table + envs: 0.5 day
- Client prefs hook + wiring in `MainContext`: 1 day
- Testing/Lighthouse/e2e validation: 0.5–1 day
- Optional background sync: 1–1.5 days

Total: ~3–5 days without background sync; ~5–6.5 days with it.

---

## 11) Implementation Checklist (ordered)

- [ ] `next.config.js`: add runtimeCaching, fallbacks, skipWaiting, image domains.
- [ ] `public/fallback.html`.
- [ ] `public/manifest.json`: add `scope`, `categories`, `shortcuts`.
- [ ] `_document.tsx`: add `preconnect`/`dns-prefetch`.
- [ ] `_app.tsx`: wrap with `PersistQueryClientProvider` using `localforage`.
- [ ] Supabase: create `user_prefs` table (+ RLS if desired).
- [ ] API: `src/pages/api/user/prefs.ts` (GET/POST).
- [ ] Hook: `src/hooks/useUserPrefs.ts` (load/merge/save).
- [ ] `src/MainContext.tsx`: select prefs and call `useUserPrefsSync`.
- [ ] QA: offline behavior, multi-device sync, Lighthouse PWA.
- [ ] (Optional) Background Sync: custom SW and queue POSTs.

---

## 12) Risks & Mitigations

- SW cache bloat: set expiration limits, max entries.
- Stale content confusion: keep NetworkFirst for HTML/API, show refresh prompts.
- Pref schema drift: include a `schemaVersion` in prefs; migrate on load if needed.
- Auth identity ambiguity: prefer Reddit username via NextAuth; if using Clerk, use Clerk `user.id` consistently.

---

## 13) References (in-repo touchpoints)

- `next.config.js` (PWA config, `images.domains`)
- `public/manifest.json`, `public/fallback.html`
- `src/pages/_document.tsx` (head tags)
- `src/pages/_app.tsx` (providers)
- `src/MainContext.tsx` (prefs states)
- `src/hooks/useUserPrefs.ts` (new)
- `src/pages/api/user/prefs.ts` (new)


