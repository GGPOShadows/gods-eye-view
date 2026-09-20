---
title: gods-eye-view — Status / Log
created: 2026-09-20 13:40 PDT
updated: 2026-09-20 13:40 PDT
status: three upstream PRs open, no responses yet; knowledge base created
tags:
  - project/gods-eye-view
  - status
aliases:
  - gods-eye-view Status
  - GEV Status
---

# gods-eye-view — Status / Log

> [!abstract] Chronological build log + forensics. Newest on top.
> For the clean consolidated version, read [[gods-eye-view REFERENCE]]. Companion: [[gods-eye-view HANDOFF]] · deep detail in [[gods-eye-view APPENDIX]].

> [!info] Dates
> Local time is PDT. GitHub shows UTC, so comments/PRs made late on 2026-09-19 PDT carry a 2026-09-20 date upstream.

---

## 2026-09-20 13:40 PDT — Knowledge base created; loose ends closed; app running with layers

- Checked all five upstream threads: **#677, #678, #680 — 0 reviews, 0 comments; #298 and #8 comments — no replies.** Decision: no further upstream work until feedback lands.
- Loose ends: `GEV_RATELIMIT_GOOGLE_PER_MIN=60` appended to `.env` (ACL verified intact); confirmed both `OPENSKY_CLIENT_*` values are in `.env`, then found `~/Downloads/credentials.json` already deleted by the owner; deleted redundant branch `fix/terrain-cache-and-tomtom-budget` locally and on origin (its two commits live on `pr/tomtom-budget` and `pr/terrain-cache-bound`).
- Dev server started on `pr/render-quality-presets`; opened `http://localhost:4173/?quality=balanced`; Google photorealistic 3D engaged automatically (credit line switched to "Google Maps"); chose LIVE CONTACTS → Austin airport with real taxiing aircraft (UAL2447 A320 at 0 ft / 14 kts), panel showed 237 flights, 8 military, 4 vessels.
- **First real-workload reading:** canvas 1518×1082, msaa 2, scale 0.85, photorealistic 3D + live layers, pane visible → **15.3 rendered fps**. Lower than the empty-globe 22.6 because the canvas is larger and 3D tiles are heavier.
- Owner flew around, then asked to continue → this doc set (Phase 4).
- Four docs written to the vault and copied to `notes/` on fork `main`. **The pre-commit guard blocked the first docs commit** — the APPENDIX quoted the fake test key and the `sk-`-prefixed cable id verbatim, and both matched the key-pattern rule. Correct behaviour; neutralized the two strings in the docs rather than bypassing the hook. (Side lesson: native Python can't see Git-Bash `/c/…` paths — `glob` silently returned nothing until the path was written as `C:/…`.) Committed as `092d648`, pushed.

## 2026-09-20 ~12:00–13:00 PDT — PR #680: opt-in render-quality presets

- Explore agent (sonnet) mapped the visual-settings architecture: DISPLAY panel is static HTML in `src/ui/templates/display-controls.html` → `shellElements.js` → `displayControls.js` → `displayBindings.js` → `visualSettings.js`/`visualEffects.js`; persistence is share-link hash only; `sharelink.celestial.test.mjs` is a whole-file invariant test enumerating every visual route. Verdict: a UI control is a large, conflict-prone surface. Chose the minimal shape instead: a URL param following the existing `?detectDebug=1` / `?trafficDebug=1` convention (`src/data/detection.js:138-146` is the pure-function template).
- Wrote `src/app/renderQuality.js` (presets `high`=4/1.0 default-identical, `balanced`=2/0.85, `performance`=1/0.6; `Object.hasOwn` lookup so `__proto__`/`constructor` can't smuggle; fails soft on a viewer without a scene) + 12 tests; wired via `applyRenderQuality(viewer, resolveRenderQualityName(globalThis.location?.search))` in `src/app/viewer.js` right after `targetFrameRate`.
- **Mistake caught:** registering the module via `json.load`/`json.dump` rewrote `package-boundaries.json` and `format-scope.json` wholesale (1,148+/787−). Reverted; re-did as three surgical text inserts (3+/1−). Placed `renderQuality.js` beside `viewer.js` in both the `viewer` and `application-components` groups; test added to `format-scope.json` (tests are excluded from runtime auto-discovery).
- End-to-end verification harness (`.gev-verify-preset.mjs`, deleted after) drove the real app at each `?quality=` and asserted the viewer adopted it: default 16.5 fps / high 15.6 / **balanced 22.6 (msaa 2, 1074×599)** / **performance 35.1 (msaa 1, 758×423)** / nonsense → falls back. All PASS.
- Docs: `CHANGELOG.md` entry; `docs/CURRENT-STATE.md` block inserted above the "Detection takes NO continuous-render hold" bullet.
- Gates: format 918 files clean; boundaries clean; build 3.65 s; **`npm test` 4,157 pass (12 new) / 0 fail**; **`test:track` 109/0/0**.
- Commit `88b8935` on `pr/render-quality-presets` (cut from `upstream/main`), pushed, **PR #680 opened** — MERGEABLE.

## 2026-09-20 ~10:00–12:00 PDT — GPU investigation (Phase 3 item 2) and issue #8 comment

Full detail in [[gods-eye-view APPENDIX#A. Performance measurement methodology]] / [[gods-eye-view APPENDIX#B. Measured results (Intel UHD 770, Chrome 152, ANGLE D3D11)]]. Summary of the forensic sequence:

1. Tried to measure in the Claude browser pane → `javascript_tool` timed out; pane was hidden; `canvas: "0x0"`. **All earlier pane-based numbers (22.7 fps baseline, "30 fps ceiling") declared invalid.**
2. Switched to headed Puppeteer Chrome (`.gev-gpu-profile.mjs`). Sequential run showed a 26% drift between first and last baseline (tile-cache warming) → sequential ordering can't be trusted.
3. Interleaved sync `scene.render()+gl.finish()` (`.gev-gpu-ab.mjs`): baseline 11.3 ms, msaa1 11.1 ms (+2%) — **wrong conclusion** ("MSAA is noise"); this method misses resolve/present cost.
4. Real rAF-driven run (`.gev-gpu-real.mjs`): idle 0 renders/s (governor perfect); continuous **18.2 fps, median frame 54 ms**; flying 15.5 fps. Raw render 11 ms vs frame 54 ms → ~43 ms unaccounted.
5. Cesium phase timing: update 0.1 ms, render 10.7 ms, frame interval 46 ms. V8 CPU profile over 8.4 s: **72.5% `(idle)`**. Not a JS bottleneck.
6. Hypothesis `preserveDrawingBuffer:true` → flipped to false → **17.9 fps, no change**. Reverted.
7. Hypothesis HUD DOM compositing → hid all non-canvas DOM → **60.2 fps** … **false positive**: re-ran with render-count + canvas-size verification → 19.4 fps. The 60 was free-running rAF after rendering stopped.
8. CSS-property bisect (`backdrop-filter`, `filter`, `box-shadow`, `text-shadow`, `mix-blend-mode`, `opacity`, animations): best combined 21 vs 17.7. Element-by-element bisect: every container ~17 fps. HUD cost is diffuse.
9. Resolution scaling 1.0→0.25: 17.3→40.2 fps for 14.8× fewer pixels — sub-linear. Fit: **≈22 ms fixed + 40 ms/MP**.
10. Found **issue #8** (proposes exactly these hypotheses) and unmerged **PR #72**; maintainer had asked on #333 for "current-main profiling … and a measured frame-time comparison". Confirmed #8's file references are stale (`src/main.js` 17 lines, `style.css` 21 lines, `backdrop-filter` 64× in `src/ui/styles/`, item 5 done).
11. Final verified matrix (`.gev-final.mjs`, 4 interleaved rounds, postRender-counted, canvas-asserted): shipped 17.9 / msaa2 20.5 / **msaa1 29.6** / blur-off 18.9 / scale.75 26.3 / **scale.5 35.6** / **combo 46.5**. Re-ran entire matrix with `preserveDrawingBuffer:false`: baseline 18.2, overlapping samples → confirmed no effect.
12. Posted the data as a comment on **issue #8** (`issuecomment-5748116872`) with the methodology trap, stale references, and an offer to PR a preset. Cleaned all `.gev-*.mjs` harnesses; tree clean.

## 2026-09-19 ~22:00 PDT — PR #678: dotenv ignore ladder

- Searched PRs/issues for `gitignore|\.env|secret|credential|leak|dotenv` — nothing relevant (only stale-CCTV-flag docs PRs and Windows DACL PRs #243/#318).
- Verified on a clean `upstream/main` checkout: `.gitignore` has bare `.env` only; `scripts/setup-doctor.mjs:92` reads `['.env','.env.local','.env.${mode}','.env.${mode}.local']`; `server/standalone/vite.config.js:11` uses `loadEnv`. `git check-ignore`: `.env` ignored; `.env.local`, `.env.production`, `.env.production.local` **COMMITTABLE**. Demonstrated: wrote a fake key to `.env.local`, `git add -A --dry-run` staged it.
- Fix scoped tighter than fork main: `.env` / `.env.*` / `!.env.example` only, with a comment citing the two readers. Re-verified all variants ignored, `.env.example` still tracked, repro no longer stages.
- Gates: format/boundaries clean, build clean, `npm test` 4,145/0, `test:track` 109/0/0. Commit `82f3516` on `pr/gitignore-env-variants`, **PR #678 opened**.

## 2026-09-19 ~21:00 PDT — PR #677 filed; terrain PR withheld; comment on #298

- Owner instruction mid-turn: read CONTRIBUTING, **check existing PRs for duplicates**. Did so before filing.
- `gh pr list … | grep -iE 'terrain|cache|leak|lru|memory|327'` → **#298** "fix(server): bound terrain-height caches, validate coordinates, throttle requests" (2026-09-11, CONFLICTING, closes #260) — same file, broader scope; **#594** bounds the *client* cache `src/services/terrainHeights.js` (which is what #327's title actually names). Our terrain fix = duplicate of #298 and mis-referenced #327. **Not filed.** Branch `pr/terrain-cache-bound` (`56db325`) kept.
- Nothing upstream touched the TomTom tile budget → novel.
- CONTRIBUTING gaps in my process: hadn't updated `CHANGELOG.md` / `docs/CURRENT-STATE.md`; hadn't run `test:track`. Fixed both on the TomTom branch (CURRENT-STATE line ~3803 "default 40k/day" → "default 6k/day…"). `test:track` first run: **109 passed, 0 failed, 5.4 min**.
- Split into one-commit branches off `upstream/main`: `pr/terrain-cache-bound`, `pr/tomtom-budget`. Amended TomTom commit with docs → `5e99adc`, pushed, **PR #677 opened** — MERGEABLE.
- Read #298's actual diff before commenting: `evictTerrainCache` sorts by `at` (fetch time) and is called **after** the insert loop but **before** the `results` rebuild. Reimplemented it verbatim in a 20-line simulation: with cap 3 and a request of `['hot'(hit, at=1), 'new'(miss)]`, the hit is evicted → rebuild yields `[null, …]` → 502. Posted on **PR #298** (`issuecomment-5747992475`): the ordering bug with repro output, the LRU-touch suggestion (also O(1) vs their O(n log n) sort per fetching request), an offer of the implementation + 4 tests, and the #327-vs-#594 cross-reference.

## 2026-09-19 ~19:00–20:30 PDT — Phase 3 items 1 & 3 implemented

- Branch `fix/terrain-cache-and-tomtom-budget` (later split and deleted).
- **Terrain cache:** confirmed `server/providers/terrain.js:38` `const mem = new Map()` with no eviction; `src/data/terrainHeightsProxy.js:227-231` `cache.set` on every valid fetch; TTL only gates re-fetch, final rebuild serves any valid entry; disk flush every 15 s persists the whole map. Eight other caches in `server/providers` bound themselves (`traffic.js` `memSet`, `overpass/cache.js` `trimOverpassCache`, military-installations `MILITARY_INSTALLATION_MAX_CACHE=80`, nominatim 80, regional brief 120, weather effects 180, aircraft tracks, `rateLimit.js` 2000 keys). Implemented `TERRAIN_CACHE_MAX_ENTRIES=20_000`, `touchTerrainCacheEntry` (LRU on read, incl. stale-but-servable), `setTerrainCacheEntry` (drain-to-limit, skips a `protectedKeys` set). **My own test caught the working-set bug** (evict-oldest FIFO returned 502 when request > ceiling) → added protected-key eviction with bounded overflow. 11/11 tests. Commit `5bb3217`.
- **TomTom:** `DEFAULT_DAILY_BUDGET` 40000→6000; comment in `traffic.js` rewritten (it claimed "~50k/day"); `.env.example` block rewritten (it cited 200K/month two lines below quoting 40000); `DATA_SOURCES.md:79` corrected. Commit `cb97e2c`.
- Full suite 4,149/0; format (had to run `npm run format` — new test file needed wrapping); boundaries; build 4.23 s. Pushed combined branch.

## 2026-09-19 ~18:00–19:00 PDT — Phase 2: keys

- Verified POWER UP endpoint: loopback-only, `no-store`, `X-Frame-Options: DENY`, never returns values; `node --test src/keySetupHardening.test.mjs` **16/16** (incl. the real `icacls` DACL applier).
- Owner created accounts and pasted keys. Panel went 8 → 4 waiting, then OpenSky added. `.env` created 18:14 PDT, 603 bytes, ACL exactly `Administrators / SYSTEM / M70Q\Medal`.
- Provider research (background tab, God's Eye tab untouched): TomTom pricing page = **200K Traffic Flow & Incidents raster tiles/month, no card**; app's `.env.example` and `traffic.js` said ~50k/day and defaulted 40000/day → 5-day exhaustion. LL2 = 15 req/h anon, Patreon for more; app TTL 15 min → token useless. OpenSky secret only in downloaded JSON; `OPENSKY_CREDENTIALS_FILE` appears 20× — all in three `.sh` files, zero JS.
- Set `TOMTOM_DAILY_TILE_BUDGET=6000` via `Add-Content` (ACL preserved), restarted, `/api/tomtom/status` → `budget:6000`.
- Live checks: OpenSky 893 KB → later 865 KB; FIRMS 34.5 → 36 MB; AIS 1.2 → 1.5 MB; LL2 1.07 MB; Celestrak 3.3 KB (`/api/celestrak/stations` — path segment; my first two `?GROUP=` calls were my error, not a fault).
- Owner also added **Google Maps** (metered) against advice — flagged: client-exposed, needs Cloud Console referrer/API restriction; `GEV_RATELIMIT_GOOGLE_PER_MIN` was unset (unlimited). Git sweep: `.env` ignored, dry-run stages nothing, hook armed.

## 2026-09-19 17:26–17:45 PDT — Phase 1: install, test, run, first (invalid) perf numbers

- `npm ci` 17:26:57→17:27:16 (18 s), 123 packages, 0 vulnerabilities, node_modules 198 MB. `allow-scripts` blocked `esbuild@0.25.12` and `puppeteer@25.10.0` postinstall → verified harmless (esbuild.exe 10.1 MB present via optional dep; Puppeteer launched Chrome 152.0.7977.75).
- `npm run doctor`: Node 24.19.0 OK, npm OK, deps OK, "Ready".
- `npm test` 154.8 s: **4,145 tests, 4,135 pass, 10 skipped, 0 fail** + allocation gates 1/1 and 13/13.
- Dev server via `Start-Process npm.cmd run dev` (`preview_start` couldn't find a launch.json at the project path). Vite 6.4.3 ready in 550 ms. First-run dialog → EXPLORE MANUALLY → Esri imagery of the Texas State Capitol. No console errors.
- Measured in the pane: renderer confirmed UHD 770 D3D11; `msaaSamples 4`, `targetFrameRate 60`, `resolutionScale 1`, `globe.maximumScreenSpaceError 2`, `requestRenderMode true`. "Baseline 22.7 fps, msaa-off 29.6, scale 0.6 30.0, atmosphere-off 30.2" → reported a "30 fps ceiling". **Later invalidated** (pane throttling). Idle = 0 renders/s was correct.
- Owner couldn't see the server: pane was hidden + Vite binds `::1` only (127.0.0.1 refused). Fronted the tab.

## 2026-09-19 ~12:00–17:00 PDT — Phase 0: fork, clone, map, harden

- Repo research via WebFetch: README, `package.json`, `.env.example`, LICENSE (MIT + NC data note), API stats (38,484★ / 7,766 forks / 217 issues, pushed 2026-09-17), top issues (#48 Docker, #85 weather, #327 leak, #648 Overpass abuse, #386 AMD/Pinokio).
- Local env: Node 24.19.0, npm 11.17.0, git 2.55, gh 2.100 (auth `GGPOShadows`, scopes incl. `repo`, `workflow`); C: 248 GB free; A: 1,494 GB free; i7-12700T, 15.7 GB, UHD 770. Decision: build on **C:** (Vite/UNC watcher risk), fork not clone.
- `gh repo fork bilawalsidhu/gods-eye-view --clone` (after `--remote` rejected with a repo arg). 1,290 files. HEAD `0d41b6b`.
- Three parallel Explore agents (sonnet): client architecture (4-phase boot, duck-typed layers, two registries, no `resolutionScale` anywhere, no tileset SSE knob, `msaaSamples:4`, `preserveDrawingBuffer:true`, render governor); server + bugs (route inventory, key brokering, three cache tiers, rate limiter; **#327 leak confirmed real** in `terrain.js`/`terrainHeightsProxy.js`; **#648 Overpass abuse largely already fixed** — grid-snapped keys, 12° bbox cap, 50 km radius cap, 24 h memory / 7–30 d disk cache, coalescing, 90/300 per-min limiter, 6 concurrent, 4 mirrors); build/Windows (`dev:secure` + `opensky:import` broken on Windows; `doctor` fine; 325 tests; CI Windows job = Pinokio path only; Puppeteer downloads Chrome; sharp has win32 prebuilt).
- Owner: "fork must be public; NO API keys pushed". Fork already public. Found upstream `.gitignore` ignores only bare `.env` while the app reads `.env.local` etc. Hardened fork `.gitignore` (commit `45a2477`, pushed to `origin/main`); wrote `.git/hooks/pre-commit`; tested it blocks a fake `sk-proj-…` in `.env` and allows the gitignore commit. History scan: only `sk-fixture-only-not-a-real-key` (test fixture) and the `sk-`-prefixed Kamchatsky–Anadyr cable id (submarine cable id).
- Puppeteer question from owner: I had proposed `PUPPETEER_SKIP_DOWNLOAD=1` copying CI. Wrong — CI never runs the Puppeteer harnesses; locally `qa-perf.mjs` (render-governor frame-count gate), `track-regression.mjs`, and ~57 others need Chrome. Corrected: download it.

## 2026-09-19 (earlier) — Session preamble

- Remote Control enabled for the Claude session (CLI needed `claude auth login` first; `/login` is an in-app command, not a shell one). Keep-awake / RC-default app settings left to the owner (settings tool wasn't available in that session).
