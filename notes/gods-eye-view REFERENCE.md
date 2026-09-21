---
title: gods-eye-view — Reference
created: 2026-09-20 13:40 PDT
updated: 2026-09-21 09:52 PDT
status: settled knowledge as of 2026-09-21; #677/#678 merged upstream, #680 open
tags:
  - project/gods-eye-view
  - reference
  - cesium
  - architecture
aliases:
  - gods-eye-view Reference
  - GEV Reference
---

# gods-eye-view — Reference

> [!abstract] Authoritative, consolidated reference — the clean how-it-works.
> Companion notes: [[gods-eye-view HANDOFF]] (resume here) · [[gods-eye-view STATUS]] (chronological log) · [[gods-eye-view APPENDIX]] (deep detail).

---

## 1. Goal & final state

**Goal:** Run God's Eye View locally on an Intel-iGPU Windows machine, key every free data layer, understand the codebase, and contribute verified improvements upstream — with zero credential leakage.

**Current/achieved:**
- Public fork `GGPOShadows/gods-eye-view` of `bilawalsidhu/gods-eye-view`, clone at `C:\Claude\Projects\gods-eye-view`, `upstream` remote wired.
- App runs keyless and keyed; 6 of 8 providers live (AISStream, NASA FIRMS, TomTom, Cesium ion, OpenSky, Google Maps). Photorealistic Google 3D tiles active.
- Upstream: **#677** (TomTom budget) and **#678** (dotenv ignore) **MERGED** 2026-09-20 by samehkhamis; **#680** (render-quality presets) open, one commit, checks clean, awaiting review. Comments posted on **PR #298** (verified eviction bug) and **issue #8** (full iGPU profiling); no replies yet.
- Secrets: upstream `.gitignore` now covers the whole dotenv ladder (via #678) plus a fork-local extras block; pre-commit guard; Windows DACL on `.env`; history scanned clean. Fork `main` synced with upstream 2026-09-20 (0 behind).

## 2. What the upstream project is

A browser-based "spy-satellite simulator" — a CesiumJS photorealistic 3D globe with ~20 live public-data layers: civil + military aircraft (OpenSky / adsb.lol), vessels (AISStream), satellites (Celestrak), launches (Launch Library 2), earthquakes (USGS), active fires (NASA FIRMS), traffic (TomTom or built-in simulation over Overpass roads), public traffic cameras (Austin, Caltrans, TfL, Ontario 511, Fintraffic, DriveBC, TxDOT, Tallinn, NSW, Calgary…), radio, bikeshare (GBFS), transit (GTFS), military installations, submarine cables, plus voice control via OpenAI Realtime and a scene "Director". Created 2026-06-22; #1 on GitHub Trending Aug 2026.

| Fact | Value (2026-09-19/20) |
|---|---|
| Stars / forks / open issues | 38,484 / 7,766 / 217 |
| Upstream HEAD when forked | `0d41b6b` 2026-09-16 (PR #626 merge) |
| Code | ~280,659 lines of JS, no framework, Vite 6 + CesiumJS ^1.124 |
| Tests | 325 `*.test.mjs` files, Node's built-in runner; 4,145 tests at fork time |
| Node | `>=24.14.0 <25 \|\| >=26 <27` |
| License | MIT for code; bundled data carries own terms, two are **non-commercial** (TeleGeography cables CC BY-NC-SA 3.0, Bhote Koshi imagery CC BY-NC 4.0) |
| Maintainers | Bilawal Sidhu, Sameh Khamis (Halfpixel) |

## 3. Key file locations

| Path | What |
|---|---|
| `src/main.js` (17 lines) | Bootstrap only: `createStandaloneApplication(...).start()` |
| `src/app/application.js` | Generic 4-phase lifecycle: `START_ORDER = ['scene','controls','data','tools']`, symmetric teardown, abort-safe |
| `src/app/viewer.js` | The **only** `new Cesium.Viewer` call. `msaaSamples: 4`, `contextOptions.webgl.preserveDrawingBuffer: true`, `targetFrameRate = 60`, `globe.show = false`, sky atmosphere tuned |
| `src/app/scene.js`, `src/mapStartup.js`, `src/maps/google3d.js` | Viewer + Google 3D tiles or keyless-globe fallback; ion tileset `cacheBytes` 1536 MB |
| `src/app/constructCatalog.js` | The list of ~20 layer factories (`createApplication<Name>()` wrappers in `src/app/layers/*.js`) |
| `src/data/layerState.js` (`LAYER_STATE_REGISTRY` ~L304) | **Second** layer registry for share-URL tokens; `finalizeRegistrations()` throws if it disagrees with the catalog |
| `src/data/lifecycle.js` (~2,300 lines) | Epoch/intent-based layer visibility state machine — the heaviest machinery in the client |
| `src/layers/<family>/` | Layer implementations; flights is ~7,200 LOC across 16 files, earthquakes ~250 LOC (the template) |
| `src/sources/live/standalone.js`, `contract.js` | Client fetch factories → same-origin `/api/*`; shared `readResponse`/`LiveSourceError` |
| `src/renderGovernor.js` (installed `src/app/tools.js:61`) | Ref-counted `holdContinuousRender()` toggling `requestRenderMode`. Parked scene = **0 renders/s** |
| `src/app/tools.js:103` | `window.__godsEyeView` debug handle (`viewer`, `dataManager`, `styleManager`…) — used by every perf harness |
| `server/providers/local.js` | Single registration point for ~20 Vite middleware plugins = the `/api/*` proxies |
| `server/providers/*.js` | Per-provider proxies: key brokering, memory→disk→upstream caching, single-flight, serve-stale |
| `server/providers/common/rate-limit.js` → `src/sources/rateLimit.js` | Per-IP sliding 60 s window, in-memory, capped at 2,000 keys |
| `server/standalone/key-setup.js`, `key-setup-hardening.mjs` | POWER UP panel: `GET /api/setup/status` (presence only), `POST /api/setup/keys` → writes `.env` + `icacls` DACL, loopback-only, dev-server only |
| `scripts/setup-doctor.mjs` | `npm run doctor`; L92 reads `.env`, `.env.local`, `.env.<mode>`, `.env.<mode>.local` |
| `scripts/run-unit-tests.mjs` | Discovers `src/**/*.test.mjs`; runs 2 GC-bracketed allocation gates separately (Node-24-calibrated) |
| `scripts/track-regression.mjs`, `scripts/qa-perf.mjs`, ~59 `qa-*.mjs` | Puppeteer harnesses; `test:track` = 109 checks against a live dev server |
| `scripts/package-boundaries.json`, `scripts/format-scope.json` | Module-ownership groups and explicit format list — **edit as text** |
| `docs/CURRENT-STATE.md` (364 KB) | Authoritative runtime reference; must be updated with any runtime PR |
| `.github/workflows/ci.yml` | `verify` on ubuntu (Node 24.14.0 + 26.x): doctor, format, boundaries, test, build; `windows-onboarding` on windows-latest: Pinokio install path + 7 named tests + build only |

## 4. Architecture in brief

1. **Boot:** `main.js` → `createStandaloneApplication` → phases **scene** (Viewer, 3D tiles/imagery, `MapStackController`) → **controls** (`StyleManager`, search) → **data** (catalog → `LayerLifecycle`/`DataLayerManager`, layer panel) → **tools** (Director, annotations, voice, **render governor last**).
2. **A layer** is a plain object `{ id, name, icon, updateInterval, init, enable, disable, update, getStats? }` registered in **two** places (`constructCatalog.js` + `LAYER_STATE_REGISTRY`). Adding one touches ~7 files (see APPENDIX §E).
3. **Data path:** browser never calls a third party. Everything is same-origin `/api/<provider>` served by Vite middleware in `server/providers/`. Keys stay in `process.env` on the server; exceptions are the two client-side keys (Google Maps, Cesium ion) injected into the bundle by `build/vite.js:33`.
4. **State:** explicit DI through phase factories; `createStateChannel` pub/sub for product state; `LayerLifecycle` events; `StyleManager` instance fields. Visual settings persist only in the share-link URL hash (`src/sharelink.js`), not localStorage (one exception: detection allocation).
5. **Rendering:** `requestRenderMode` governed by ref-counted holds. Idle parked scene costs nothing. Moving-object layers hold continuous render.

## 5. Setup recipe (Windows, this machine)

```bash
gh repo fork bilawalsidhu/gods-eye-view --clone      # origin=fork, upstream=parent
cd gods-eye-view
npm ci                                                # 123 pkgs; allow-scripts warnings are harmless
npm run doctor                                        # "Ready"
npm run dev                                           # http://localhost:4173  (use localhost, not 127.0.0.1)
```
Then in the app: **POWER UP** → paste free keys → **SAVE KEYS**. If a Google key is added, append the spend throttle to `.env` (Add-Content preserves the ACL) and restart:
```
GEV_RATELIMIT_GOOGLE_PER_MIN=60
```
(`TOMTOM_DAILY_TILE_BUDGET` now defaults to 6000 upstream since #677 merged; set it only to override.) These are app-side, per-IP, in-memory guards — **not billing caps**; provider-side budget alerts are the real protection.
Optional perf preset (branch `pr/render-quality-presets`): `http://localhost:4173/?quality=balanced|performance`.

## 6. Providers — what each key does and costs

| Provider | Env var(s) | Tier | Notes |
|---|---|---|---|
| Cesium ion | `CESIUM_ION_TOKEN` | free signup | **Client-exposed.** Use an `assets:read` token. Ion-hosted Google 3D + Bing + world terrain |
| AISStream | `AISSTREAM_API_KEY` | free signup | Live vessels via server WebSocket; client cap `VITE_AIS_LIVE_MAX_ROWS=12000` |
| NASA FIRMS | `FIRMS_MAP_KEY` | free, email only | Active fires; ~36 MB payload |
| TomTom | `TOMTOM_API_KEY` | free, **200K tiles/month**, no card | Traffic Flow & Incidents Raster Tiles. Key = "My first API key" at `my.tomtom.com`. The old 40k/day default exhausted the month in 5 days; **#677 (merged 2026-09-20) made 6k/day the upstream default** |
| OpenSky | `OPENSKY_CLIENT_ID` + `_SECRET` | free signup | Only raises polling credits; anon works. Secret is shown once, in a downloaded `credentials.json`. On Windows, paste both values into the panel — the file path setting is bash-only |
| Google Maps | `GOOGLE_MAPS_API_KEY` | **metered**, billing account required; 1,000 photorealistic-3D sessions/month currently free | **Client-exposed.** Restrict by HTTP referrer + API in Cloud Console. Set `GEV_RATELIMIT_GOOGLE_PER_MIN` (default unlimited) |
| OpenAI | `OPENAI_API_KEY` | **metered** — "the one that costs real money" | Realtime voice, cents/minute, $5/session app cap. **Unset.** |
| Launch Library 2 | `LL2_API_TOKEN` | no free token exists (Patreon) | App caches 15 min → 4 req/h vs 15 req/h anon limit. **Unset, correctly.** |

## 7. Contribution rules (from `CONTRIBUTING.md`, verified in practice)

1. Branch off `main` (in practice: off `upstream/main`, one commit per PR).
2. `npm run build`, `npm test`, `npm run test:track` — **all three green** (plus `format:check`, `check:boundaries`).
3. Runtime behaviour change ⇒ update `docs/CURRENT-STATE.md` **and** `CHANGELOG.md` (newest entry on top, bullet style) in the same PR.
4. New data source ⇒ update `DATA_SOURCES.md` with license/attribution; never bundle data you can't redistribute.
5. Describe what changed and how it was verified.
6. Style: ES modules, 2-space, single quotes, semicolons, JSDoc on exports, conventional-commit prefixes appreciated.
7. **Unwritten but essential:** search open/closed PRs and issues before writing a line (217 open issues, ~7.7k forks — duplicates are likely). Keep mechanical formatting out of behavioural diffs.

## 8. Performance — the settled facts (Intel UHD 770)

Full methodology and tables in [[gods-eye-view APPENDIX#A. Performance measurement methodology]] and [[gods-eye-view APPENDIX#B. Measured results (Intel UHD 770, Chrome 152, ANGLE D3D11)]].

- **Valid measurement requires:** headed Chrome with the real GPU, frames counted from `scene.postRender`, canvas dimensions asserted unchanged, warm tile cache, interleaved A/B/A arms. Anything else produced wrong numbers in this project — twice.
- **Shipped config on this iGPU:** ~17.9 fps at 1264×705, keyless, no layers, parked. ~15 fps with photorealistic 3D + flights/vessels/installations at 1518×1082 on `balanced`.
- **What works:** `msaaSamples` 4→1 = **+65%**; `viewer.resolutionScale` 1.0→0.5 = **+99%** (never set upstream); combined = **+160%**.
- **What doesn't (measured, not guessed):** `preserveDrawingBuffer:false` = no effect; removing `backdrop-filter` = ±noise; no single HUD element is responsible.
- **The floor:** frame time ≈ **22 ms fixed + 40 ms/megapixel**. The fixed term is outside JS (main thread 72% idle, Cesium render phase ~11 ms, frame interval ~46 ms) and caps this machine near ~44 fps regardless of settings. Unresolved.
- **Idle is free:** the render governor really does hit 0 renders/s on a parked scene.

## 9. Critical gotchas (clean version)

- `localhost`, never `127.0.0.1` — IPv6-only bind.
- Never measure perf in the Claude browser pane (canvas → 0×0 when hidden).
- Never count rAF ticks as fps; count `postRender` and check the canvas.
- Sync `scene.render()+gl.finish()` timing misses MSAA resolve/present — don't use it for user-facing fps.
- `OPENSKY_CREDENTIALS_FILE`, `npm run dev:secure`, `npm run opensky:import` are macOS/Linux-only (bash + `security` keychain). Windows CI never exercises them.
- Upstream `.gitignore` used to ignore only bare `.env` while the app reads `.env.local` and `.env.<mode>[.local]` too; **#678 (merged 2026-09-20) closed that** — upstream now ignores the whole dotenv ladder. Fork `main` adds only `*.pem`, `*.crt`, `*.key`, `credentials.json`. Hooks do not clone: recreate `.git/hooks/pre-commit` from APPENDIX §C on a fresh checkout.
- App-side rate limits and budgets are not billing caps.
- Hooks (`.git/hooks/pre-commit`) don't clone — recreate on a fresh checkout (APPENDIX §C).
- Issue #8's file references are stale: `src/main.js` and `style.css` were refactored; `backdrop-filter` count is 64 across `src/ui/styles/*.css`; its item 5 is already done.
