---
title: gods-eye-view — Technical Log & Hand-off
created: 2026-09-20 13:40 PDT
updated: 2026-09-20 22:14 PDT
status: PRs #677 and #678 MERGED upstream (2026-09-20); #680 rebased and awaiting review; local app fully running with 6 providers keyed
host: M70Q (Windows 11 Pro 10.0.26200, Intel UHD 770)
tags:
  - project/gods-eye-view
  - handoff
  - cesium
  - open-source-contribution
  - performance
aliases:
  - gods-eye-view Handoff
  - GEV Handoff
---

# gods-eye-view — Technical Log & Hand-off

> [!abstract] Last updated **2026-09-20 22:14 PDT**
> Companion notes: [[gods-eye-view REFERENCE]] (clean how-it-works) · [[gods-eye-view APPENDIX]] (deep detail: perf methodology, measurements, security audit) · [[gods-eye-view STATUS]] (chronological log + forensics).
> GitHub (**public** fork): `https://github.com/GGPOShadows/gods-eye-view` · upstream: `https://github.com/bilawalsidhu/gods-eye-view` · local repo: `C:\Claude\Projects\gods-eye-view`.

---

## 🧭 Hand-off prompt (paste this to resume in a new chat)

> [!quote] Copy from here
> I'm continuing my personal project on **gods-eye-view**, a public fork (`GGPOShadows/gods-eye-view`) of Bilawal Sidhu's `bilawalsidhu/gods-eye-view` — a ~280k-LOC vanilla-JS + CesiumJS + Vite browser app that renders live public geospatial data (aircraft, ships, satellites, fires, traffic, CCTV) on a photorealistic 3D globe. The local clone is at `C:\Claude\Projects\gods-eye-view` on Windows 11 (host M70Q, Intel UHD 770 integrated graphics, Node 24.19.0). It runs at `http://localhost:4173` via `npm run dev` — **use `localhost`, not `127.0.0.1`; Vite binds IPv6-only here.** The full knowledge base is in the Obsidian vault at `C:\Obsidian\Personal-Claude\Projects\gods-eye-view\` (four docs: HANDOFF, REFERENCE, STATUS, APPENDIX) and copied under `notes/` on the fork's `main`; read HANDOFF first.
>
> **State:** Phases 0–3 of the plan are done. Six providers are keyed in the git-ignored, ACL-locked `.env` (AISStream, NASA FIRMS, TomTom, Cesium ion, OpenSky, Google Maps); OpenAI and Launch Library are deliberately unset. **Two upstream PRs are MERGED** (2026-09-20, by maintainer samehkhamis): #677 (TomTom daily tile budget 40k→6k) and #678 (gitignore the whole dotenv ladder). #680 (opt-in `?quality=` render presets, measured +113% fps on this iGPU) is open, rebased onto current `main` after those merges, one commit, all gates green. I also posted a verified 502 bug on PR #298 and full profiling data on issue #8. A fourth fix (terrain cache LRU, branch `pr/terrain-cache-bound`) was NOT filed because it duplicates #298.
>
> **Guardrails:** (1) Never commit or push a `.env*` file or key — a pre-commit hook and hardened `.gitignore` guard this; do not bypass them. (2) Before any new upstream work, search existing PRs and issues first — this project has 217 open issues and ~7.7k forks and I already nearly filed a duplicate. (3) Every PR branches off `upstream/main`, one commit, and must pass all three gates: `npm run build`, `npm test`, `npm run test:track` (dev server must be up), plus `format:check` and `check:boundaries`; runtime changes also need `CHANGELOG.md` + `docs/CURRENT-STATE.md` in the same PR. (4) Any performance number is untrustworthy unless frames are counted from `scene.postRender` **and** the canvas size is asserted unchanged — I got burned twice by a hidden browser pane (canvas 0×0) and a free-running rAF after rendering silently stopped. Measure in a headed Puppeteer Chrome, never in the Claude browser pane. (5) Don't do more upstream work until at least one PR gets feedback; build in a style they might reject and it's wasted.
>
> **Next:** check the remaining upstream threads (#680, and the #298 / #8 comments) for responses; then the Windows credentials-file gap (the `OPENSKY_CREDENTIALS_FILE` setting is read only by three macOS/Linux shell scripts, and `npm run dev:secure` / `npm run opensky:import` are broken on Windows); then optionally a DISPLAY-rail UI control for the quality preset as a follow-up to #680. Two user-side TODOs remain: restrict the Google Maps key by HTTP referrer in Cloud Console, and set a Cloud Console budget alert.

---

## 🎯 Project goal & success criteria

- **Goal:** Build out God's Eye View locally on this PC, understand it, get every free data layer live, and find genuine ways to improve it — contributed upstream where they hold up.
- Motivation: it's the #1 GitHub-trending project of August 2026 (38k stars), it's a real, hackable geospatial platform, and the Intel iGPU on this machine makes it a legitimately useful test bed nobody upstream has.
- Constraints / non-goals:
  - No metered spend without explicit decision (OpenAI voice is the one that "costs real money"; Google requires a billing account).
  - Nothing commercial: bundled datasets carry non-commercial licenses (TeleGeography cables CC BY-NC-SA 3.0, Bhote Koshi imagery CC BY-NC 4.0) even though the code is MIT.
  - **No API key ever reaches GitHub.** Standing requirement from the owner.
- Success criteria:
  1. [x] Fork is public, cloned locally, builds and runs keyless.
  2. [x] All free-tier providers keyed and verified returning live data.
  3. [x] At least one verified, non-duplicate improvement filed upstream — three filed.
  4. [x] At least one upstream PR merged — **#677 and #678 merged 2026-09-20** by samehkhamis.
  5. [x] Knowledge base stood up and kept current — this document set.

## 📍 Current state

- [x] **Phase 0** — forked to `GGPOShadows/gods-eye-view` (public, inherited from parent), cloned to `C:\Claude\Projects\gods-eye-view`, `upstream` remote added, codebase mapped (three parallel Explore agents), secrets hardened (gitignore + pre-commit hook), history scanned clean.
- [x] **Phase 1** — `npm ci` (123 packages, 18 s), `npm run doctor` ready, `npm test` 4,149 pass, `npm run build` clean, dev server up, globe renders keyless on Esri imagery.
- [x] **Phase 2** — 6 of 8 providers keyed via the in-app POWER UP panel; each proxy queried directly and returning real payloads; `.env` ACL verified as exactly owner + SYSTEM + Administrators; TomTom budget set to 6000/day; Google throttle set to 60/min.
- [x] **Phase 3, item 1 (terrain memory leak)** — fixed and tested on `pr/terrain-cache-bound`, **not filed** (duplicates upstream PR #298); findings posted as a comment on #298 instead.
- [x] **Phase 3, item 3 (TomTom budget default)** — filed as **PR #677**.
- [x] **Security fix (dotenv ignore)** — filed as **PR #678**.
- [x] **Phase 3, item 2 (GPU performance)** — full investigation done, findings posted on issue #8, opt-in presets filed as **PR #680**.
- [x] **#677 and #678 MERGED** upstream 2026-09-20 (22:43Z / 22:33Z) by samehkhamis, no review comments.
- [x] **#680 rebased** onto post-merge `main` (conflicts in `src/app/viewer.js` header vs the #284 pinch-zoom block, and `CHANGELOG.md` ordering); gates re-run green; force-pushed with lease; rebase note posted.
- [ ] **Awaiting review on #680**, and replies on the #298 and #8 comments. ← **NEXT: check these**
- [ ] **Phase 3, item 4 (Windows credentials gap)** — verified, unclaimed, not started.
- [x] **Phase 4 (knowledge base)** — this doc set, in the vault and under `notes/` on fork `main`; kept current.
- [ ] **User-side:** restrict Google Maps key by HTTP referrer in Cloud Console; set a Cloud Console budget alert. Neither confirmed done.

---

## ✅ What WORKED — with specifics

**Local build, keyless.** From a fresh clone:
```bash
cd C:\Claude\Projects\gods-eye-view
npm ci            # 123 packages, ~18 s, node_modules ≈ 249 MB with Puppeteer's Chrome cached
npm run doctor    # reports Node 24.19.0 supported, npm 11.17.0, dependencies installed, "Ready"
npm run dev       # Vite 6.4.3 ready in ~550 ms → http://localhost:4173
```
npm 11.17's `allow-scripts` guard **blocked** the `esbuild` and `puppeteer` postinstall scripts with a warning. This broke nothing: esbuild's Windows binary arrives via `optionalDependencies` (`@esbuild/win32-x64`), and Puppeteer launched Chrome 152.0.7977.75 from `%USERPROFILE%\.cache\puppeteer\` fine. No action needed.

**Keys via the in-app panel, not by hand.** Click **POWER UP · N KEYS WAITING** (bottom-right) → paste → **SAVE KEYS**. The dev server writes repo-root `.env`, applies a Windows DACL via `icacls` restricted to exactly `BUILTIN\Administrators`, `NT AUTHORITY\SYSTEM`, `M70Q\Medal`, and restarts itself. The endpoint is loopback-only and never echoes a key value (`/api/setup/status` reports presence only). Verified the ACL path with `node --test src/keySetupHardening.test.mjs` (16/16) before pasting anything real.

**Verifying each provider returns real data** (not just a green dot):
```powershell
Invoke-WebRequest 'http://localhost:4173/api/opensky?lamin=30.1&lomin=-97.9&lamax=30.5&lomax=-97.6'  # ~865 KB
Invoke-WebRequest 'http://localhost:4173/api/ais-live'            # ~1.5 MB
Invoke-WebRequest 'http://localhost:4173/api/firms'               # ~36 MB
Invoke-WebRequest 'http://localhost:4173/api/tomtom/status'       # {"hasKey":true,"dailyCount":0,"budget":6000,...}
Invoke-WebRequest 'http://localhost:4173/api/celestrak/stations'  # group is a PATH segment, not ?GROUP=
Invoke-WebRequest 'http://localhost:4173/api/launches'            # ~1 MB, works anonymously
```

**Appending non-secret config to `.env` without breaking the ACL:** `Add-Content -Path .env -Value "..." -Encoding utf8` preserves the DACL. Then restart the dev server (kill the listener on 4173, `npm run dev` again) — a manual `.env` edit is not hot-reloaded.

**Real GPU profiling that can be trusted.** Headed Puppeteer Chrome (`headless: false`, args `--ignore-gpu-blocklist --use-angle=d3d11`), 45 s settle for a warm tile cache, frames counted via `scene.postRender.addEventListener`, canvas `width×height` asserted every arm, arms interleaved A/B/A with order reversed on alternate rounds, median of 4 rounds. Full harness in [[gods-eye-view APPENDIX#A. Performance measurement methodology]].

**Surgical JSON edits for the boundary/format registries.** `scripts/package-boundaries.json` and `scripts/format-scope.json` must be edited as text (Python `str.replace` on a unique anchor line). Loading + `json.dump` reformatted both files and produced a 1,900-line diff for a 3-line change — reverted.

**Independent one-commit PR branches.** Each PR is its own branch cut from `upstream/main` (`git fetch upstream && git checkout -b pr/<name> upstream/main`, then `git cherry-pick <sha>`), so each is mergeable alone and no docs/gitignore hardening leaks into any of them.

## ❌ What did NOT work / dead ends (don't re-try)

| Attempt | Result |
|---|---|
| Measuring frame rate inside the Claude desktop browser pane | **Invalid.** When the pane hides, the canvas collapses to `0x0` and `requestAnimationFrame` pauses. My first published numbers (22.7 fps baseline, "30 fps ceiling") were artifacts of this. Never measure perf in the pane. |
| Counting `requestAnimationFrame` ticks as "fps" | **False positive.** Reported a perfect 60 fps for "HUD hidden" — rendering had silently stopped and rAF was free-running at vsync. Count `scene.postRender`, and assert canvas dims. |
| Timing `scene.render()` + `gl.finish()` in a tight synchronous loop | Measures draw-call execution only (~11 ms) and **misses the MSAA resolve / compositor / present cost**. It showed MSAA as "2% / noise" when the real rAF-driven number is +65%. Useful for isolating draw cost, useless for user-facing fps. |
| `preserveDrawingBuffer: false` (issue #8's #1 suggestion) | **No measurable effect.** 17.9 → 18.2 fps, overlapping samples, two independent full runs. Reverted. |
| Removing all `backdrop-filter` rules (issue #8's #3) | +6% one run, −1% the next. Noise. Also bisected by hiding each overlay container individually — every arm ~17 fps; the HUD cost is diffuse and small. |
| Reducing `viewer.resolutionScale` alone to chase the "30 fps ceiling" | The ceiling didn't exist (see row 1). With valid measurement, resolutionScale is the **strongest** single knob (0.5 → +99%). |
| Filing the terrain-cache LRU fix as a PR | Duplicate of upstream **PR #298** (2026-09-11), which bounds the same cache and adds coordinate validation, a disk byte cap and rate limiting. Also, my commit said `closes #327` but #327's title names the **client** file `src/services/terrainHeights.js` — that's PR #594's territory, not the server proxy I fixed. Branch kept as `pr/terrain-cache-bound`; refinements offered as a comment on #298 instead. |
| `gh repo fork owner/repo --clone=true --remote=true` | `--remote` is rejected when a repo argument is given. Use `gh repo fork bilawalsidhu/gods-eye-view --clone` — it sets `origin`=fork and `upstream`=parent automatically. |
| `preview_start` with `.claude/launch.json` inside the project | The tool looks for `C:\Claude\.claude\launch.json` (the session root), not the project dir. Started the server with `Start-Process npm.cmd run dev` instead and navigated to it. |
| Rewriting `package-boundaries.json` / `format-scope.json` via `json.load`/`json.dump` | Reformatted the entire file: 1,148 insertions / 787 deletions for 3 real lines. CONTRIBUTING forbids mixing mechanical formatting with behavioral edits. Reverted; edited as text. |
| Getting a free Launch Library 2 token | There isn't one. Higher rate limits are Patreon-only. And the app caches LL2 for 15 min (max 4 req/h) against a 15 req/h anonymous limit — a token buys nothing. Leave the field empty. |
| Pointing the app at OpenSky's downloaded `credentials.json` on Windows | `OPENSKY_CREDENTIALS_FILE` is read by **zero** JavaScript — only `scripts/dev-fresh.sh`, `scripts/dev-secure.sh`, `scripts/opensky-import-client.sh`, all bash + macOS `security` keychain. On Windows the only path is opening the JSON and pasting `clientId`/`clientSecret` into the panel. (This is the unfiled Phase 3 item 4.) |
| `try { } catch { }` as an expression in PowerShell 5.1 | Parser error ("The term 'try' is not recognized"). Use `if ((Get-Command x -ErrorAction SilentlyContinue)) {...}` or a statement-level try. |

---

## 🐛 Gotchas & operational procedures

> [!warning] Vite binds IPv6-only here
> `Get-NetTCPConnection -LocalPort 4173` shows `::1` only. `http://localhost:4173` and `http://[::1]:4173` work; `http://127.0.0.1:4173` is refused. Always use `localhost`. Don't bind `0.0.0.0` casually — `SECURITY.md` warns it exposes every key-brokering proxy to the LAN.

> [!warning] The Claude browser pane is not a measurement instrument
> Hidden pane ⇒ canvas `0x0`, rAF paused, `javascript_tool` times out at 45 s on any promise that awaits a frame. For anything performance-related use the headed Puppeteer harness (APPENDIX §A). For simply *looking* at the app, `tabs_select` fronts the pane.

> [!danger] Keys
> `.env` is git-ignored (hardened rule set on fork `main`), ACL-locked, and guarded by `.git/hooks/pre-commit`, which refuses any staged `.env*`/`*.pem`/`*.key`/`*.crt`/`credentials.json` and any live-looking `sk-…`/`AIza…`/`ghp_…` pattern (excluding `src/data/local_data/**` and the `fixture-only-not-a-real-key` test string). Bypass only with `--no-verify`, and only if you have personally verified the diff. Two keys are **client-exposed by design** (Google Maps, Cesium ion) and visible in devtools — restrict them provider-side, don't try to hide them.

- **Cost guards are app-side, not billing caps.** `TOMTOM_DAILY_TILE_BUDGET=6000` and `GEV_RATELIMIT_GOOGLE_PER_MIN=60` are in-memory, per-IP, reset on restart. Provider-side budget alerts are the real protection. TomTom's free tier has no card, so an overrun cuts the layer off rather than billing; Google's does bill.
- **Restarting the dev server:** kill whatever owns port 4173, then `npm run dev`. The Provider Settings panel restarts it for you; manual `.env` edits don't.
- **Branch discipline:** `main` on the fork = `upstream/main` + one gitignore-hardening commit (`45a2477`) + (after this checkpoint) the `notes/` doc copies. Never cut a PR branch from `main`; always from `upstream/main`.
- **Checking for duplicates before filing** (mandatory — see dead-ends):
  ```bash
  gh pr list --repo bilawalsidhu/gods-eye-view --state all --limit 500 --json number,title,state --jq '.[] | "#\(.number) [\(.state)] \(.title)"' | grep -iE 'keyword1|keyword2'
  gh issue list --repo bilawalsidhu/gods-eye-view --state all --limit 500 --json number,title,state --jq '.[] | "#\(.number) [\(.state)] \(.title)"' | grep -iE 'keyword'
  ```
- **The three CONTRIBUTING gates**, all must be green before a PR (`test:track` needs the dev server up and takes ~5.4 min):
  ```bash
  npm run format && npm run format:check && npm run check:boundaries
  npm test          # expect 4,1xx pass / 0 fail (+ 1 + 13 in the two allocation gates)
  npm run build     # "✓ built in ~4 s"; chunk-size warnings are pre-existing
  npm run test:track  # expect "RESULT: 109 passed, 0 failed, 0 skipped"
  ```
- **New files:** a new runtime `.js` under `src/` is auto-discovered by the formatter; a new `*.test.mjs` **must** be added to `scripts/format-scope.json`; a new module imported by a file that belongs to a `scripts/package-boundaries.json` group must be added to that group's `modules` (e.g. `src/app/viewer.js` is in both `viewer` and `application-components`).
- **Puppeteer on Windows:** headless mode uses SwiftShader (software GL). For real-GPU work launch `headless: false`. `puppeteer.executablePath()` returns a **Promise** in v25 — `await` it.
- **PowerShell 5.1 here:** no `&&`/`||` chain operators, no `try` as an expression, no ternary. Native-exe stderr lines get wrapped as `NativeCommandError` noise even on exit 0 — harmless.

## 🔧 Environment & access

- **Host M70Q** — Windows 11 Pro 10.0.26200 (x64). Intel Core i7-12700T (35 W), 15.7 GB RAM, **Intel UHD Graphics 770** integrated (reported by WebGL as `ANGLE (Intel, Intel(R) UHD Graphics 770 (0x00004680) Direct3D11 vs_5_0 ps_5_0, D3D11)`). No discrete GPU. This is the *point* — it's the low-end test bed.
- **Toolchain:** Node **v24.19.0** (project requires `>=24.14.0 <25 || >=26 <27`), npm **11.17.0**, git **2.55.0.windows.3**, GitHub CLI **2.100.0** (on PATH in this session; historically at `C:\Program Files\GitHub CLI\gh.exe`), Python 3 (used for surgical file edits), PowerShell **5.1** primary + Git Bash.
- **Browser for perf:** Chrome for Testing **152.0.7977.75** via Puppeteer 25.10.0, cached at `%USERPROFILE%\.cache\puppeteer\chrome\`.
- **Accounts:** GitHub **GGPOShadows** (token scopes `gist, read:org, repo, workflow, write:packages`); git identity `GGPOShadows <187427758+GGPOShadows@users.noreply.github.com>`. Provider accounts created by the owner: AISStream, NASA FIRMS, TomTom (`my.tomtom.com`, key "My first API key"), Cesium ion, OpenSky (OAuth client), Google Cloud (Maps key, billing-enabled). Not created: OpenAI, TheSpaceDevs Patreon.
- **Key paths:**
  - Repo: `C:\Claude\Projects\gods-eye-view` — **deliberately local, not the NAS** (`A:\Claude\Projects`): Vite's file watcher has the same UNC-path failure class that forced webpack+polling on the Next.js project, and there's no upside with 248 GB free on C:.
  - Vault: `C:\Obsidian\Personal-Claude\Projects\gods-eye-view\`
  - Secrets: `C:\Claude\Projects\gods-eye-view\.env` (host-only, never committed)
  - Server caches: `C:\Claude\Projects\gods-eye-view\.gev-cache\` (git-ignored; terrain heights, TomTom tiles/budget, overpass)
  - Session scratch logs: `C:\Users\Medal\AppData\Local\Temp\claude\C--Claude\d54f371c-…\scratchpad\` (volatile)
- **Networking:** dev server `http://localhost:4173` (IPv6 `::1` only). Remote Control is enabled for the Claude session so it can be steered from claude.ai/code or the phone.

---

## 📁 File inventory

> [!note] What lives where. Items marked **host-only** are not in the repo and are not recoverable from GitHub.

### Repo `C:\Claude\Projects\gods-eye-view` (fork; git-tracked on `main` unless noted)
| File | What |
|---|---|
| `README.md` | **Upstream's** README — untouched, do not overwrite with a project README |
| `.gitignore` | Hardened on fork `main` (commit `45a2477`): `.env`, `.env.*`, `!.env.example`, `pinokio/ENVIRONMENT`, `*.pem`, `*.crt`, `*.key`, `credentials.json`, `.gev-cache/`. The upstream PR #678 version is scoped tighter (dotenv ladder only). |
| `notes/gods-eye-view {HANDOFF,REFERENCE,STATUS,APPENDIX}.md` | Committed copies of this doc set (vault is source of truth) |
| `src/app/renderQuality.js` + `.test.mjs` | **On branch `pr/render-quality-presets` only** — the `?quality=` preset module (PR #680) |
| `src/app/viewer.js` | Only `new Cesium.Viewer` call in the codebase; `msaaSamples: 4`, `preserveDrawingBuffer: true`, `targetFrameRate = 60` |
| `server/providers/traffic.js` | TomTom proxy + daily budget governor; `DEFAULT_DAILY_BUDGET` = 6000 on `pr/tomtom-budget` (PR #677), still 40000 upstream |
| `server/providers/terrain.js`, `src/data/terrainHeightsProxy.js` | The unbounded cache (fixed on `pr/terrain-cache-bound`, unfiled) |
| `server/standalone/key-setup.js`, `key-setup-hardening.mjs` | The POWER UP panel backend + Windows `icacls` DACL hardener |
| `scripts/setup-doctor.mjs` | `npm run doctor`; line 92 enumerates the dotenv ladder the app reads |
| `scripts/package-boundaries.json`, `scripts/format-scope.json` | Registries that new modules/tests must be added to (edit as text!) |
| `CONTRIBUTING.md`, `docs/CURRENT-STATE.md`, `CHANGELOG.md` | The rules; the 364 KB authoritative runtime reference; the changelog every runtime PR must touch |

### Not in the repo (host-only / volatile)
| Path | What |
|---|---|
| `.env` **(host-only, secret)** | 9 vars: `AISSTREAM_API_KEY`, `FIRMS_MAP_KEY`, `TOMTOM_API_KEY`, `CESIUM_ION_TOKEN`, `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET`, `TOMTOM_DAILY_TILE_BUDGET=6000`, `GOOGLE_MAPS_API_KEY`, `GEV_RATELIMIT_GOOGLE_PER_MIN=60`. If lost, every key must be regenerated at its provider (OpenSky's secret is shown once). |
| `.git/hooks/pre-commit` **(host-only)** | The secret guard. Hooks aren't cloned — recreate from [[gods-eye-view APPENDIX#C. The pre-commit secret guard]] on any fresh clone. |
| `.git/info/exclude` | Contains `.claude/` so the launch config never shows as untracked |
| `.claude/launch.json` | Dev-server launch config (`gev-dev`, `npm run dev`, port 4173) — not actually used by `preview_start`, see dead-ends |
| `.gev-cache/` | Server-side caches; safe to delete; regenerates |
| `node_modules/` (249 MB) | `npm ci` regenerates |
| `%USERPROFILE%\.cache\puppeteer\` | Chrome for Testing 152; Puppeteer re-downloads if missing |
| `%USERPROFILE%\Downloads\credentials.json` | OpenSky OAuth download — **already deleted** (values live in `.env`) |

---

## ⏭️ Next steps

- [ ] **Check the remaining upstream threads**: PR #680 (rebased, awaiting review); comments on PR #298 and issue #8. (#677 and #678 are merged — done.) (`gh pr view <n> --repo bilawalsidhu/gods-eye-view --json reviews,comments`)
- [ ] **User-side, not yet confirmed done:** in Google Cloud Console restrict the Maps key by HTTP referrer (`http://localhost:4173/*`) and by API (Map Tiles + Geocoding); set a Billing → Budgets alert. Also OpenAI usage limits if a key is ever added.
- [ ] **Phase 3 item 4 — Windows credentials gap** (verified, unclaimed, owner personally hit it): teach the Node server to read `OPENSKY_CREDENTIALS_FILE` (accepting `clientId`/`clientSecret` or `client_id`/`client_secret`, per `docs/opensky-auth.md`), and make `npm run dev:secure` / `npm run opensky:import` not shell out to bash + macOS `security`. Check PRs/issues for `opensky|credential|windows` first.
- [ ] **Follow-up to #680, only if the maintainer wants it:** a DISPLAY-rail control for the quality preset. Touches `src/ui/templates/display-controls.html`, `shellElements.js`, `displayControls.js`, `displayBindings.js`, `visualSettings.js`, `sharelink.js`, and the whole-file invariant test `src/sharelink.celestial.test.mjs`. Deliberately deferred — large surface, high conflict risk.
- [ ] **Bring fork `main` up to date:** it is still upstream-as-of-2026-09-16 + gitignore hardening + `notes/`. Upstream has moved 41 commits including #677/#678. `git merge upstream/main` will likely auto-merge, leaving `.gitignore` with duplicated `.env`/`.env.*` lines (fork block appended at the end, #678 edited in place) — dedupe by hand. Then decide whether to also merge `pr/render-quality-presets` so `?quality=` is always available locally.
- [ ] **Open investigation (hard):** the ~22 ms per-frame fixed cost outside JS. Main thread idle 72%, Cesium render phase ~11 ms, yet frames arrive every ~46–54 ms. Not the HUD, not `preserveDrawingBuffer`, not fill rate. Compositor/present path is the remaining suspect. Would need Chrome tracing (`chrome://tracing` / `--trace-startup`), not JS-side timers.
- [ ] Keep this doc set current: append to STATUS as work happens, bump `updated:`.

---

## 🩹 Command crib

```bash
# ---- run ----
cd /c/Claude/Projects/gods-eye-view
npm run dev                                   # http://localhost:4173  (NOT 127.0.0.1)
# with the perf preset (only on pr/render-quality-presets or after merging it):
#   http://localhost:4173/?quality=balanced     http://localhost:4173/?quality=performance

# ---- restart dev server (PowerShell) ----
# Get-NetTCPConnection -LocalPort 4173 -State Listen | % { Stop-Process -Id $_.OwningProcess -Force }; npm run dev

# ---- key/provider status (presence only, never values) ----
curl -s http://localhost:4173/api/setup/status | python -m json.tool | grep -E '"title"|"set"'
curl -s http://localhost:4173/api/tomtom/status

# ---- the gates (all must pass before any PR) ----
npm run format && npm run format:check && npm run check:boundaries
npm test
npm run build
npm run test:track                            # dev server must be up; ~5.4 min; expect 109/0/0

# ---- upstream hygiene ----
git fetch upstream
git log --oneline HEAD..upstream/main         # has upstream moved?
git checkout -b pr/<name> upstream/main       # every PR branch starts here
gh pr list  --repo bilawalsidhu/gods-eye-view --state all --limit 500 --json number,title,state --jq '.[] | "#\(.number) [\(.state)] \(.title)"' | grep -iE '<kw>'
gh issue list --repo bilawalsidhu/gods-eye-view --state all --limit 500 --json number,title,state --jq '.[] | "#\(.number) [\(.state)] \(.title)"' | grep -iE '<kw>'
for n in 677 678 680; do gh pr view $n --repo bilawalsidhu/gods-eye-view --json number,state,mergeable,reviews,comments --jq '"#\(.number) \(.state) \(.mergeable) reviews=\(.reviews|length) comments=\(.comments|length)"'; done

# ---- secret safety sweep ----
git status --short                            # must be empty of .env*
git check-ignore -v .env .env.local           # both must print a rule
git add -A --dry-run | grep -iE '\.env|credential|secret' && echo LEAK || echo clean
git diff upstream/main..HEAD | grep -E '^\+' | grep -oE 'sk-[A-Za-z0-9_-]{20,}|AIza[A-Za-z0-9_-]{35}|ghp_[A-Za-z0-9]{36}' | grep -v fixture-only

# ---- Windows ACL on the secret file ----
# icacls .env      -> expect exactly: BUILTIN\Administrators:(F), NT AUTHORITY\SYSTEM:(F), M70Q\Medal:(F)
```
