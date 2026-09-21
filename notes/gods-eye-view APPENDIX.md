---
title: gods-eye-view — Technical Appendix (deep detail)
created: 2026-09-20 13:40 PDT
updated: 2026-09-20 22:14 PDT
tags:
  - project/gods-eye-view
  - appendix
  - performance
  - security
aliases:
  - gods-eye-view Appendix
  - GEV Appendix
---

# gods-eye-view — Technical Appendix

> [!abstract] Granular detail that doesn't fit the main flow.
> Parents: [[gods-eye-view HANDOFF]] · [[gods-eye-view REFERENCE]] · [[gods-eye-view STATUS]].

---

## A. Performance measurement methodology

> [!danger] Two traps that produced false numbers in this project
> 1. **Hidden Claude browser pane** ⇒ `viewer.canvas` is `0x0`, `requestAnimationFrame` never fires, `javascript_tool` times out. Every number captured that way was fiction.
> 2. **Counting rAF ticks** ⇒ rAF free-runs at vsync even when Cesium has stopped drawing. A "60 fps" for hidden-HUD was rendering having silently stopped.
>
> Rule: **count `scene.postRender` events and assert `canvas.width×height` unchanged on every arm.** Anything else is not a measurement.

**Why headed Puppeteer, not headless:** `TESTING.md` notes headless picks Metal on macOS and **SwiftShader (software GL) elsewhere**. Headless on Windows would profile a CPU rasterizer, not the UHD 770. Launch with `headless: false` and check `WEBGL_debug_renderer_info` prints `ANGLE (Intel, Intel(R) UHD Graphics 770 … D3D11)`; abort if it says SwiftShader.

**Why interleaved arms:** the first sequential run drifted 26% between the first and last baseline (22.89 → 18.15 ms) purely from tile-cache warming. Arms must alternate, with order reversed on alternate rounds, after a ≥45 s settle.

**Why not sync `scene.render()+gl.finish()`:** it isolates draw-call cost (~11 ms) but excludes MSAA resolve, compositor and present. It reported MSAA as +2% when the rAF-driven truth is +65%. Use it only to prove "the draw calls themselves are cheap".

**The verified harness** (reconstruct into `<repo>/.gev-final.mjs`; delete before committing — the `.gitignore` does not cover it):

```js
import puppeteer from 'puppeteer';
const browser = await puppeteer.launch({ headless: false, defaultViewport: null,
  args: ['--window-size=1280,800','--ignore-gpu-blocklist','--use-angle=d3d11'] });
const page = (await browser.pages())[0];
await page.goto('http://localhost:4173', { waitUntil: 'domcontentloaded', timeout: 120000 });
await page.waitForFunction(() => !!window.__godsEyeView?.viewer, { timeout: 120000 });
await page.evaluate(() => { document.querySelectorAll('button,[role=button],div,span').forEach(el => {
  if (/EXPLORE MANUALLY/i.test(el.textContent||'') && el.click) el.click(); }); });
await new Promise(r => setTimeout(r, 45000));                     // warm tile cache

async function arm(cfg) {
  return page.evaluate(async (c) => {
    const v = window.__godsEyeView.viewer, s = v.scene;
    document.getElementById('__probe')?.remove();
    if (c.css) { const st = document.createElement('style'); st.id='__probe'; st.textContent=c.css; document.head.appendChild(st); }
    s.msaaSamples = c.msaa ?? 4;  v.resolutionScale = c.scale ?? 1.0;
    s.requestRenderMode = false;  await new Promise(r => setTimeout(r, 1500));
    const dims = `${v.canvas.width}x${v.canvas.height}`;
    let renders = 0; const off = s.postRender.addEventListener(() => { renders++; });
    const t0 = performance.now();
    await new Promise(res => { (function tick(){ performance.now()-t0 < 5000 ? requestAnimationFrame(tick) : res(); })(); });
    off(); s.requestRenderMode = true;
    return { dims, renderFps: +(renders / ((performance.now()-t0)/1000)).toFixed(1) };
  }, cfg);
}
const ARMS = { shipped:{msaa:4,scale:1}, msaa1:{msaa:1,scale:1}, scale05:{msaa:4,scale:0.5} };
const names = Object.keys(ARMS), runs = Object.fromEntries(names.map(n => [n, []]));
for (let r = 0; r < 4; r++) for (const n of (r % 2 ? [...names].reverse() : names)) runs[n].push((await arm(ARMS[n])).renderFps);
const med = a => [...a].sort((x,y)=>x-y)[Math.floor(a.length/2)];
for (const n of names) console.log(n, med(runs[n]), 'fps', runs[n]);
await browser.close();
```

**Cesium phase timing** (to locate time inside a frame): subscribe to `scene.preUpdate / postUpdate / preRender / postRender`, diff `performance.now()`. Observed medians: update 0.1 ms, between 0 ms, render 10.7 ms, **frame-to-frame 46 ms**.

**V8 CPU profile** via CDP (`Profiler.enable`, `setSamplingInterval 200`, `start`/`stop`), aggregate self-time by node id over `profile.samples`. Observed: **72.5% `(idle)`**, 3.1% `(program)`, then Cesium internals ≤1% each (`Cartesian3.subtract`, `CesiumWidget.resize`, `Cesium3DTile.updateVisibility`…). Conclusion: not JS-bound.

## B. Measured results (Intel UHD 770, Chrome 152, ANGLE D3D11)

**B.1 — Verified matrix, empty globe.** 1264×705 canvas, keyless Esri basemap, no layers, parked camera, warm cache, median of 4 interleaved rounds, `postRender`-counted, canvas asserted.

| Arm | fps | vs shipped | samples |
|---|---|---|---|
| as shipped (msaa 4, scale 1.0) | 17.9 | — | 18.9, 16.8, 17.9, 17.7 |
| msaa 2 | 20.5 | +15% | 20.5, 19.7, 19.2, 20.8 |
| **msaa 1** | **29.6** | **+65%** | 29.1, 29.3, 29.8, 29.6 |
| backdrop-filter: none | 18.9 | +6% | 20.4, 18.9, 18.3, 17.9 |
| resolutionScale 0.75 (948×528) | 26.3 | +47% | 26.7, 25.3, 26.3, 24.4 |
| **resolutionScale 0.5** (632×352) | **35.6** | **+99%** | 35.8, 35.1, 35.6, 33.1 |
| msaa 1 + scale 0.5 + no blur | 46.5 | +160% | 46.1, 46.5, 48.4, 46.4 |

Same matrix re-run with `preserveDrawingBuffer: false` in `src/app/viewer.js`: shipped **18.2** (18.2, 17.0, 17.1, 18.3), msaa2 19.8, msaa1 28.9, blur-off 18.1, scale.75 24.2, scale.5 33.1, combo 49.0. Baseline samples overlap the `true` run ⇒ **no effect**.

**B.2 — Resolution scaling (single run, rendered fps, canvas asserted):**

| scale | canvas | MP | fps | throughput MP/s |
|---|---|---|---|---|
| 1.0 | 1264×705 | 0.89 | 17.3 | 15.4 |
| 0.75 | 948×528 | 0.50 | 23.8 | 11.9 |
| 0.5 | 632×352 | 0.22 | 31.3 | 7.0 |
| 0.35 | 442×246 | 0.11 | 37.7 | 4.1 |
| 0.25 | 316×176 | 0.06 | 40.2 | 2.2 |

Two-point fit (1.0 vs 0.25): `57.8 = f + 0.89k`, `24.9 = f + 0.06k` ⇒ **k ≈ 39.6 ms/MP, f ≈ 22.5 ms**. Throughput *falling* as pixels drop is the signature of a fixed per-frame cost.

**B.3 — Real workload (rAF-driven, window visible):** idle governor-on **0 renders/s** at 60 rAF; parked continuous 18.2 fps (median 54.1 ms, p90 66.9, worst 134.8); flying to new terrain 15.5 fps (median 60.3, p90 99.2); parked at destination 20.3 fps. `globe.tilesLoaded` was true in 100% of samples ⇒ not tile-streaming-bound.

**B.4 — Bisects that found nothing (all ≈17 fps):** CSS properties (`backdrop-filter`, `filter`, `box-shadow`, `text-shadow`, `mix-blend-mode`, `opacity`, animations; best combined 21.0). Elements one at a time: `#world-overlay-root`, `#title-bar`, `#style-indicator`, `#top-center-actions`, `#traffic-sync-chip`, `#cctv-sync-chip`, `#toast`, `#command-dock`, `#left-panel-stack`, `#right-context-rail`, `#scene-runtime`, `#first-run-launcher`, `#key-setup-chip`, `#intel-hud`, `#loading-screen`, `#orbit-indicator`, `div.gev-screen-whiteboard` — range 16.5–18.0.

**B.5 — Preset end-to-end (PR #680):** default 16.5 / `high` 15.6 / `balanced` 22.6 (msaa 2, 1074×599) / `performance` 35.1 (msaa 1, 758×423) / `nonsense` → default 15.7. With photorealistic 3D + live layers at 1518×1082 on `balanced`: **15.3 fps**.

**B.6 — Invalidated numbers (do not cite):** pane-based "22.7 baseline / 29.6 msaa-off / 30.0 scale-0.6 / 30.2 atmosphere-off" and the "30 fps ceiling"; sync-render "11.3 ms baseline / msaa1 +2%"; rAF-only "HUD hidden 60.2 fps".

## C. The pre-commit secret guard

Hooks are not cloned. Recreate at `.git/hooks/pre-commit` (Git Bash `chmod +x`) on any fresh checkout:

```sh
#!/bin/sh
# Fork-local secret guard. Blocks commits containing credential files or
# live-looking API keys. Bypass only with an explicit --no-verify.
fail=0
staged=$(git diff --cached --name-only --diff-filter=ACM)
for f in $staged; do
  case "$f" in
    .env.example) ;;
    .env|.env.*|pinokio/ENVIRONMENT|credentials.json|*.pem|*.key|*.crt)
      echo "BLOCKED: credential file staged: $f"; fail=1 ;;
  esac
done
hits=$(git diff --cached -U0 --no-color -- . ':(exclude)src/data/local_data/**' \
  | grep '^+' \
  | grep -Eo 'sk-[A-Za-z0-9_-]{20,}|AIza[A-Za-z0-9_-]{35}|ghp_[A-Za-z0-9]{36}|sk-proj-[A-Za-z0-9_-]{20,}' \
  | grep -v 'fixture-only-not-a-real-key' | sort -u)
if [ -n "$hits" ]; then echo "BLOCKED: live-looking API key in staged content:"; echo "$hits" | sed 's/^/  /'; fail=1; fi
if [ "$fail" -ne 0 ]; then echo; echo "Commit aborted by .git/hooks/pre-commit"; echo "If this is a false positive, re-run with: git commit --no-verify"; exit 1; fi
exit 0
```

Tested 2026-09-19: a staged `.env` containing `OPENAI_API_KEY=sk-proj-AbCd…(fake, 40 chars)` was blocked on both rules; the gitignore commit passed. Known benign matches in history: `sk-fixture-only-not-a-real-key` (`src/**/*.test.mjs`) and the `sk-`-prefixed Kamchatsky–Anadyr cable id (a cable id inside `src/data/local_data/telegeography_submarine_cables/cable-geo.json`).

**Fork `main` `.gitignore` additions** (appended block, commit `45a2477`): `.env`, `.env.*`, `!.env.example`, `pinokio/ENVIRONMENT`, `*.pem`, `*.crt`, `*.key`, `credentials.json`, `.gev-cache/`. The upstream PR #678 version is only `.env` / `.env.*` / `!.env.example`, placed where the original `.env` line was, with a comment naming the two readers.

## D. Upstream contribution ledger

| # | Type | Branch / SHA | Subject | State (2026-09-20 13:40 PDT) |
|---|---|---|---|---|
| [PR #677](https://github.com/bilawalsidhu/gods-eye-view/pull/677) | fix | `pr/tomtom-budget` `5e99adc` | `DEFAULT_DAILY_BUDGET` 40000→6000; corrected `traffic.js` comment, `.env.example`, `DATA_SOURCES.md`, `CHANGELOG.md`, `CURRENT-STATE.md` | **MERGED** 2026-09-20 22:43Z by samehkhamis |
| [PR #678](https://github.com/bilawalsidhu/gods-eye-view/pull/678) | security | `pr/gitignore-env-variants` `82f3516` | `.gitignore`: `.env` → `.env`, `.env.*`, `!.env.example` | **MERGED** 2026-09-20 22:33Z by samehkhamis |
| [PR #680](https://github.com/bilawalsidhu/gods-eye-view/pull/680) | feat/perf | `pr/render-quality-presets` `88b8935` | `src/app/renderQuality.js` + 12 tests, `viewer.js` wiring, boundary/format registration, docs | OPEN; rebased onto post-#677/#678 `main` 2026-09-20 (now `395579a`), gates green, awaiting review |
| [PR #298 comment](https://github.com/bilawalsidhu/gods-eye-view/pull/298#issuecomment-5747992475) | review | — | Eviction-before-rebuild ⇒ 502 (simulated repro); LRU-touch + O(1) eviction; offer of impl + tests; #327↔#594 cross-ref | posted, no reply |
| [Issue #8 comment](https://github.com/bilawalsidhu/gods-eye-view/issues/8#issuecomment-5748116872) | data | — | Full verified matrix; #8 items 1 & 3 don't measure; `resolutionScale` unlisted lever; 22 ms floor; stale refs | posted, no reply |
| *(unfiled)* | fix | `pr/terrain-cache-bound` `56db325` | `TERRAIN_CACHE_MAX_ENTRIES=20_000`, `touchTerrainCacheEntry`, `setTerrainCacheEntry` w/ protected working set, both disk-load paths bounded, 4 tests | duplicate of #298 — held |

**Related upstream context:** issue #8 "Render Performance Suggestions" (open) with unmerged PR #72 (CONFLICTING); issue #327 "Unbounded In-Memory Cache Growth in `terrainHeights.js`" (open; client file); PR #594 (client terrain cache, open); PR #298 (server terrain cache + validation + disk cap + throttle, open, CONFLICTING, 2026-09-11); issue #648 Overpass overload (largely mitigated in current main); issues #661 / #201 TomTom; #362 doctor + `OPENSKY_AUTH_MODE`; #386 AMD GPU in Pinokio.

## E. Adding a new data layer (from the architecture map)

1. `src/layers/<name>/index.js` — `{ id, name, icon, updateInterval, init, enable, disable, update }` (template: `src/layers/earthquakes/index.js:26`).
2. Source: `src/layers/<name>/source.js` or `src/sources/live/*`; if it needs a proxy, `server/providers/<name>.js` + register in `server/providers/local.js`.
3. `src/standalone/layerSources.js` — wire the source factory.
4. `src/app/layers/<name>.js` — `createApplication<Name>({ source, surface })`.
5. `src/app/constructCatalog.js` — push into the catalog (and `SOURCE_METHODS` if validating the source shape).
6. `src/data/layerState.js` `LAYER_STATE_REGISTRY` — add `{ id, token, disposition }` or `finalizeRegistrations()` throws at startup.
7. Optional `CONTROL_LAYER_IDS` in `src/app/catalog.js`.
8. A `*.test.mjs` beside each new file, listed in `scripts/format-scope.json`; new modules imported across a boundary group added to `scripts/package-boundaries.json`.

## F. Server `/api/*` route inventory

`/api/opensky` (aircraft/opensky.js:352) · `/api/opensky-track`, `/api/adsblol/trace` (tracks.js) · `/api/adsblol/mil` · `/api/adsbdb` · `/api/ais-live` (vessels/ais-live.js:74) · `/api/celestrak/<group>` (path segment; `?GROUP=` is a 400) · `/api/launches` · `/api/tomtom/*` incl. `/status` · `/api/firms` · `/api/terrain/heights?points=lon,lat;…` (≤2000 points, 64/chunk upstream) · `/api/overpass` (90/min/IP, 300 global, 6 concurrent, 12° bbox / 50 km radius caps, 4 mirrors) · `/api/military-installations` · `/api/regional-brief`, `/api/geocode`, `/api/weather-effects` · `/api/cctv` · `/api/radio` · `/api/gbfs` · `/api/transit` · `/api/openai/hud-summary`, `/api/realtime/token`, `/api/realtime/debug-log` · `/api/google/nearby-places`, `/api/google/text-search` · `/api/route` · `/api/setup/status`, `/api/setup/keys` (loopback, dev only) · anything else → JSON 404.

## G. Windows-specific gaps in upstream (verified, unfiled)

| Item | Evidence | Windows behaviour |
|---|---|---|
| `npm run dev:secure` → `./scripts/dev-secure.sh` | bash shebang, `security find-generic-password`, bash line-continuation env syntax | "not recognized as an internal or external command" from cmd.exe |
| `npm run opensky:import` → `./scripts/opensky-import-client.sh` | L8-11: exits 1 if `security` not found | non-functional by design on any non-Mac |
| `OPENSKY_CREDENTIALS_FILE` | 20 references, all in `dev-fresh.sh`, `dev-secure.sh`, `opensky-import-client.sh`; **0 in JS** | silently ignored; user must paste `clientId`/`clientSecret` into the panel |
| `scripts/dev-fresh.sh` (426 lines), `dev-cctv.sh` | `pkill`, `lsof`, `ipconfig getifaddr en0`, Keychain | macOS only; equivalent is plain `npm run dev` + `.env` |
| CI `windows-onboarding` | `.github/workflows/ci.yml:57-89` | runs Pinokio install + 7 named tests + build; never the bash launchers, never full `npm test`, never `test:track` |
| `scripts/qa-l9-matrix.mjs:2102-2117` | shells `ls`, reads `$HOME` for nvm | best-effort detection fails silently; not in `package.json` |

What *does* work natively: `doctor` (uses `npm.cmd` + `shell:true` on win32, skips Keychain), `dev`, `build`, `test`, `check:boundaries`, `format`, the Puppeteer harnesses (with Chrome), and `key-setup-hardening.mjs`'s `icacls` path (resolves `whoami.exe`/`icacls.exe`/`powershell.exe` from `%SystemRoot%\System32` without PATH, grants owner SID + SYSTEM + Administrators, re-verifies via `Get-Acl`).
