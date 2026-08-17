# PASS 4 — EXPERIMENT STATUS

**Project:** open-media-tools
**Repository:** `Alot1z/open-media-tools`
**Pass:** 4
**Date:** 2026-08-17
**Status:** COMPLETE

This document reports the status of the 8 experiments identified in Pass 2 (§experiment plan) and Pass 3 (§19). **No experiment results are fabricated.** Only Experiment 1 was executed during Pass 4 (it was cheap, safe, and reduced real architectural uncertainty). The others are classified by urgency and given exact specifications for when they will be run.

---

## Classification Scheme (per Pass 4 protocol)

| Classification | Meaning |
|---|---|
| **BLOCKING** | Must be resolved before the architecture can be finalized. If safe to run now, run it. |
| **NON-BLOCKING** | Reduces uncertainty but the architecture has a documented fallback/condition. Can run now if cheap; otherwise document spec. |
| **IMPLEMENTATION-TIME** | Affects an implementation detail; the architecture handles both outcomes. Run during implementation. |
| **DEFERRED** | Optimization or future-scope experiment. Not needed for v1 architecture. |

---

## Summary Table

| Exp | Question | Classification | Executed in Pass 4? | Result | Architectural impact |
|---|---|---|---|---|---|
| 1 | GitHub Releases CORS for browser fetch | NON-BLOCKING (but run) | **YES** | **FAILS** — no CORS headers on release-assets; jsDelivr/unpkg WORK | **NEW-AD-A**: FFmpeg.wasm via jsDelivr npm, not Releases |
| 2 | Mediabunny robustness on real platform media | IMPLEMENTATION-TIME | No | — (spec below) | Escape-hatch frequency only; fallback exists |
| 3 | Cloudflare Workers extractor viability | DEFERRED | No | — (spec below) | Optimization; Node.js is default |
| 4 | TikTok CDN Range-request support | IMPLEMENTATION-TIME | No | — (spec below) | Resume strategy (AD-15 handles both) |
| 5 | iOS Safari FFmpeg.wasm memory cap | IMPLEMENTATION-TIME | No | — (spec below; needs iOS device) | iOS escape-hatch viability; default 384MB cap until measured |
| 6 | Residential-proxy IG extraction | NON-BLOCKING (scope-condition) | No | — (spec below; needs paid proxy) | IG viability in v1; architecture unchanged |
| 7 | Signed-URL TTL per platform | IMPLEMENTATION-TIME | No | — (spec below) | Tunes re-extraction threshold; AD-15 handles arbitrary TTL |
| 8 | iOS navigator.share large-file | IMPLEMENTATION-TIME | No | — (spec below; needs iOS device) | iOS large-file UX; fallback exists |

**No experiment was simulated or guessed.** Exp 1 was empirically run; Exp 2-8 have documented specs and will be run at the appropriate time (implementation or when the required resource is available).

---

## Experiment 1 — GitHub Releases CORS — EXECUTED

**Classification:** NON-BLOCKING (but cheap + safe → run to reduce uncertainty)
**Status:** ✅ EXECUTED (2026-08-17)
**Full evidence:** `.research/experiment-1-gh-releases-cors-RESULT.md`

### Question
Does `release-assets.githubusercontent.com` (and the `github.com/.../releases/download/...` 302 redirect) send `Access-Control-Allow-Origin` allowing a browser `fetch()` from a `*.github.io` origin?

### Setup (what was actually done)
1. Created a temporary GitHub Release on `Alot1z/open-media-tools` (tag `cors-probe`, prerelease).
2. Uploaded a 24-byte test asset `cors-probe.txt`.
3. Probed unauthenticated (as a browser would) with `curl -sIL -H "Origin: https://alot1z.github.io"`.
4. Probed the CORS preflight with `curl -X OPTIONS -H "Origin: ..." -H "Access-Control-Request-Method: GET"`.
5. Probed jsDelivr npm (`cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/umd/ffmpeg-core.wasm`) and unpkg as proposed fallbacks.
6. Cleaned up: deleted the release and tag (verified 0 releases / 0 tags after).

### Result (raw — not fabricated)
- **GitHub Releases direct fetch: FAILS CORS.**
  - `github.com` 302 response: `location: https://release-assets.githubusercontent.com/...`, **no `access-control-allow-origin` header**.
  - `release-assets.githubusercontent.com` 200 response: `content-type: application/octet-stream`, served by Azure Blob / Fastly, **no `access-control-allow-origin` header**.
  - A browser `fetch()` from `*.github.io` would receive the bytes but be blocked by CORS from exposing them to JS.
- **CORS preflight (OPTIONS) on the release-download URL: 404** (github.com doesn't handle OPTIONS there). Non-simple requests (e.g., with `Range` header) are blocked at preflight.
- **jsDelivr npm: WORKS.** `access-control-allow-origin: *`, `access-control-expose-headers: *`, `content-type: application/wasm`, `cache-control: public, max-age=31536000, s-maxage=31536000, immutable`, `x-cache: HIT, HIT`.
- **unpkg npm: WORKS** (Cloudflare-backed, same CORS headers, `access-control-allow-methods: GET, HEAD, OPTIONS`).
- **jsDelivr gh-repo mirror of a release asset: 404** (jsDelivr `/gh/` mirrors repo files at a path, NOT release assets). Cannot use to mirror our own release assets.

### Architectural impact
**NEW-AD-A (introduced in Pass 4):** FFmpeg.wasm escape-hatch is distributed via **jsDelivr npm CDN** (`https://cdn.jsdelivr.net/npm/@ffmpeg/core@<pinned-version>/dist/umd/ffmpeg-core.wasm`), NOT GitHub Releases. GitHub Releases is demoted to source-tarball + release-metadata host only. jsDelivr promoted from OPTIONAL (Pass 3) to USE (Pass 4). unpkg remains OPTIONAL alternative.

### Remaining uncertainty
- Whether GitHub will add CORS headers to release-asset responses in the future (unlikely; re-test periodically).
- Whether a future custom FFmpeg.wasm build (e.g., LGPL-only without x264) can be distributed browser-side: it must be published as an npm package or hosted on a CORS-friendly static host (GitHub Releases alone won't work). USE-LATER concern.

### Confidence
HIGH — empirically measured against the actual user's repo with real release assets and real CDN responses.

---

## Experiment 2 — Mediabunny robustness — NOT EXECUTED

**Classification:** IMPLEMENTATION-TIME
**Status:** ⏳ Not run; spec documented; will run during media-module implementation.
**Why not BLOCKING:** The architecture already has the FFmpeg.wasm escape-hatch fallback [AD-6]. Mediabunny robustness affects *how often* the escape hatch fires (performance/cost), not *whether the architecture works*. The architecture proceeds either way.

### Question
Does Mediabunny parse/mux real TikTok/Instagram/Facebook media (MP4, fragmented MP4, WebM) at ≥90% success?

### Hypothesis
≥90% success on real platform media (Mediabunny is mature, covers common containers).

### Spec (run during implementation)
- **Setup:** Collect 20 sample media files (10 TikTok MP4, 5 IG MP4, 5 FB MP4) via the remote extractor (once it exists). Store locally as test fixtures.
- **Run:** `Mediabunny.read(file)` + `Mediabunny.write(remuxed)` on each; capture success/failure + error type.
- **Measurement:** Success rate per platform; error categorization (unsupported container, parse error, mux error).
- **Success criteria:** ≥90% overall success.
- **Failure criteria:** <90% → investigate failures; may need mp4box.js as primary for MP4-family, or trigger FFmpeg.wasm escape hatch for specific cases.
- **Architectural consequence if fails:** Escape-hatch trigger frequency rises; may shift AD-6 toward FFmpeg.wasm-by-default for specific platforms/containers. Architecture boundary unchanged.
- **When to run:** When implementing `apps/web/src/media/` (the media-processing module). Requires the remote extractor to be working (to obtain real samples).

### Remaining uncertainty (until run)
Exact success rate on real platform media; which specific containers/variants Mediabunny struggles with.

---

## Experiment 3 — Cloudflare Workers extractor viability — NOT EXECUTED

**Classification:** DEFERRED
**Status:** ⏳ Not run; spec documented; deferred (optimization experiment).
**Why not BLOCKING:** The architecture defaults to Node.js + yt-dlp on a VPS/container [AD-7, tech ledger]. CF Workers is an EXPERIMENTAL alternative that would eliminate the yt-dlp dependency and give edge performance. The architecture proceeds with Node.js; CF Workers is a future optimization.

### Question
Can a pure-JS TikTok scraper run within Cloudflare Workers' 128MB memory / 50ms CPU limits?

### Hypothesis
Yes, for a single-URL extraction (HTML fetch + parse + JSON extract + return) — the work is I/O-bound, not CPU-bound.

### Spec (run if/when edge-runtime optimization is pursued)
- **Setup:** Port the TikTok scraper (from `apps/api/src/extractors/tiktok.ts`) to pure JS with no Node-specific deps. Deploy to Cloudflare Workers (free tier).
- **Run:** Test 50 TikTok URLs through the Worker.
- **Measurement:** Success rate; CPU time (ms); memory (MB); wall-clock latency.
- **Success criteria:** ≥80% success; <50ms CPU; <128MB memory.
- **Failure criteria:** Fails → Node.js + yt-dlp on VPS remains the path.
- **Architectural consequence if succeeds:** Remote service may migrate to CF Workers (edge, free tier, no yt-dlp dependency). This would be a Pass-5+ decision, not a v1 change.

### Remaining uncertainty
Whether the pure-JS scraper fits CF Workers' limits; whether residential-proxy egress is possible from CF Workers (it may require a subrequest to a proxy service).

---

## Experiment 4 — TikTok CDN Range-request support — NOT EXECUTED

**Classification:** IMPLEMENTATION-TIME
**Status:** ⏳ Not run; spec documented; will run during download-module implementation.
**Why not BLOCKING:** The large-file state machine [AD-15] handles both outcomes: if Range works, resume from byte offset; if not, restart from zero. The architecture proceeds; the experiment only determines which branch is taken for TikTok.

### Question
Does the TikTok CDN honor `Range: bytes=N-` on signed video URLs (returning 206 Partial Content)?

### Hypothesis
Yes for progressive MP4s; uncertain for signed URLs (the signature may cover the full request, not a byte range).

### Spec (run during implementation)
- **Setup:** Extract a TikTok video URL via the remote service. Start a download, abort at 50% (track bytesWritten to OPFS). Attempt resume with `Range: bytes=N-`.
- **Measurement:** HTTP status (206 vs 200 vs 403); `Content-Range` header presence; whether resumed bytes match.
- **Success criteria:** 206 Partial Content; resume succeeds.
- **Failure criteria:** 200 (full restart) or 403 (URL rejects Range) → resume restarts from zero.
- **Architectural consequence:** Determines the resume branch in AD-15's state machine for TikTok. The state machine already handles both; this just selects the path.

### Remaining uncertainty
Per-CDN Range behavior; may differ between TikTok's progressive MP4s and HLS fragments.

---

## Experiment 5 — iOS Safari FFmpeg.wasm memory cap — NOT EXECUTED

**Classification:** IMPLEMENTATION-TIME
**Status:** ⏳ Not run; spec documented; **cannot run in this environment** (no iOS device). Will run when an iOS device is available during implementation.
**Why not BLOCKING:** The architecture caps WASM memory defensively at 384MB by default [AD-6, §5.5] and uses the pure-TS path as the default. Even if FFmpeg.wasm is unusable on iOS, the architecture holds (pure-TS only on iOS; escape hatch disabled). The experiment refines the cap, doesn't change the boundary.

### Question
What is the practical WASM linear-memory cap on iOS Safari before a tab crash?

### Hypothesis
~256-512MB is safe; the emscripten default of 2GB crashes iOS Safari.

### Spec (run when iOS device available)
- **Setup:** Load FFmpeg.wasm with `MAXIMUM_MEMORY` set to 256MB / 512MB / 1GB / 2GB on iOS Safari (real device). Process a 500MB file.
- **Measurement:** Tab crash / success per cap; identify the threshold.
- **Success criteria:** Identify a safe cap (≥256MB) that doesn't crash.
- **Failure criteria:** Even 256MB crashes → FFmpeg.wasm unusable on iOS; disable escape hatch on iOS entirely (pure-TS only).
- **Architectural consequence:** Sets the iOS WASM memory budget; may disable the escape hatch on iOS. Default 384MB cap until measured.
- **Why not run now:** No iOS device in this environment. The defensive default (384MB) is safe; the experiment refines it.

### Remaining uncertainty
Exact iOS tab memory limit (no published number; varies by device). The architecture treats iOS WASM as best-effort with a defensive cap.

---

## Experiment 6 — Residential-proxy Instagram extraction — NOT EXECUTED

**Classification:** NON-BLOCKING for architecture; **scope-condition for IG in v1**
**Status:** ⏳ Not run; spec documented; **cannot run in this environment** (requires a paid residential proxy service). Will run before IG is confirmed as a v1 target.
**Why not BLOCKING for architecture:** The architecture is the same whether IG is in v1 or not. But this experiment IS blocking for the **v1 platform scope** (whether IG ships in v1). If it fails, IG drops from v1 (the architecture unchanged, just fewer adapters).

### Question
Does Instagram extraction succeed at ≥70% with a residential proxy and ≤20% with a datacenter IP?

### Hypothesis
Residential ≥70%; datacenter ≤20% (IG aggressively blocks datacenter IPs).

### Spec (run before IG v1 ship)
- **Setup:** Sign up for BrightData or Oxylabs (residential). Run the IG extractor via residential egress vs a datacenter IP (e.g., DigitalOcean droplet). Test 30 IG URLs.
- **Measurement:** Success rate per egress; error categorization (403, challenge, timeout).
- **Success criteria:** Residential ≥70% success; datacenter ≤20%.
- **Failure criteria:** Residential <70% → IG extraction is unreliable; **drop IG from v1 platform list** (ship with TikTok + Facebook + direct-URL; add IG later when proxy reliability improves or an extension path is available).
- **Architectural consequence:** Determines whether the InstagramAdapter ships in v1. Architecture boundary unchanged; platform scope changes.
- **Why not run now:** Requires a paid proxy subscription (~$50+ minimum). Not available in this environment. The architecture is designed to ship with or without IG.

### Remaining uncertainty
Actual residential-proxy success rate; cost per successful extraction (bandwidth + per-request).

---

## Experiment 7 — Signed-URL TTL per platform — NOT EXECUTED

**Classification:** IMPLEMENTATION-TIME
**Status:** ⏳ Not run; spec documented; will run during download-module implementation.
**Why not BLOCKING:** The resume-on-expiry state machine [AD-15] handles arbitrary TTLs (re-extract on failure). The exact TTL only tunes the *proactive* re-extraction threshold (re-extract before expiry to avoid mid-download failure). The architecture proceeds.

### Question
How long are TikTok/Instagram/Facebook CDN URLs valid (TTL)?

### Hypothesis
TikTok ~1-2 hours; Instagram ~5-15 minutes; Facebook ~1 hour. (Based on Pass 1 cluster D; not precisely measured.)

### Spec (run during implementation)
- **Setup:** Extract URLs via the remote service for each platform. Attempt `fetch()` (via `/proxy`) at 0 / 5 / 10 / 30 / 60 / 120 minutes after extraction.
- **Measurement:** Time of first 403/failure per platform.
- **Success criteria:** Measure per-platform TTL.
- **Failure criteria:** N/A (measurement, not pass/fail).
- **Architectural consequence:** Sets the proactive re-extraction threshold per platform in AD-15's state machine (re-extract if `time-since-extraction > TTL/2`). The state machine already handles reactive re-extraction on failure.

### Remaining uncertainty
Exact per-platform TTL; whether TTL varies by content type (video vs image).

---

## Experiment 8 — iOS navigator.share large-file — NOT EXECUTED

**Classification:** IMPLEMENTATION-TIME
**Status:** ⏳ Not run; spec documented; **cannot run in this environment** (no iOS device). Will run when an iOS device is available during implementation.
**Why not BLOCKING:** The architecture uses `navigator.share({files})` as the iOS save path with `<a download>` as fallback [AD-22]. If share fails for large files, the fallback (chunk into zip, or advise smaller files) doesn't change the architecture.

### Question
Can `navigator.share({files: [500MB video]})` save to Files on iOS Safari without failure?

### Hypothesis
Yes, within ~30 seconds (iOS Share Sheet handles large files, but may be slow).

### Spec (run when iOS device available)
- **Setup:** On iOS Safari, download a 500MB MP4 to OPFS. Invoke `navigator.share({files: [new File([opfsBlob], 'video.mp4')]})` from a user gesture.
- **Measurement:** Save success/failure; time to save; any iOS prompt.
- **Success criteria:** Save succeeds within ~30s; file appears in Files app.
- **Failure criteria:** Fails → fall back to `<a download>` (less reliable on iOS for media) OR advise "file saved in app storage — use a smaller file or desktop."
- **Architectural consequence:** Affects iOS large-file UX; may add a "chunk to zip" fallback for very large files. Architecture unchanged.

### Remaining uncertainty
Practical iOS Share Sheet file-size limit; behavior on older iOS versions.

---

## Why only Experiment 1 was executed

The Pass 4 protocol states: *"If an experiment is BLOCKING and can safely be executed now, perform it."* Of the 8 experiments:

- **None are BLOCKING** for the architecture. Each has either a documented fallback (Exp 1, 2, 4, 7), a defensive default (Exp 5), a scope-condition rather than architecture condition (Exp 6), or a non-architecture UX detail (Exp 8). Exp 3 is a deferred optimization.
- **Exp 1 was NON-BLOCKING but cheap, safe, and uncertainty-reducing** — so I ran it. It required only: creating a temporary release, curling with an Origin header, inspecting response headers, cleaning up. No paid services, no iOS device, no real platform media. The result (FAILS → jsDelivr is required) had a concrete architectural consequence (NEW-AD-A), justifying the run.
- **Exp 2, 4, 7** require the remote extractor service to exist first (to obtain real platform media / signed URLs). They run at implementation time.
- **Exp 3** requires porting a scraper and deploying to CF Workers — a substantial effort for an optimization experiment. Deferred.
- **Exp 5, 8** require an iOS device, which this environment does not have. Deferred to when an iOS device is available.
- **Exp 6** requires a paid residential-proxy subscription. Deferred to before IG v1 ship.

**I did not fabricate any result.** The 7 unexecuted experiments have exact specifications and will be run at the appropriate time. The architecture is designed to proceed safely with the uncertainty (via fallbacks, defensive defaults, and state machines that handle both outcomes).

---

## Impact on the Architecture

| Experiment | If it passes | If it fails | Architecture changes? |
|---|---|---|---|
| 1 (DONE) | (n/a — it failed) | jsDelivr is the path | YES — NEW-AD-A |
| 2 | Mediabunny is default | Escape hatch fires more often | NO (fallback exists) |
| 3 | CF Workers viable as alternative | Node.js stays default | NO (optimization) |
| 4 | Resume from byte offset | Restart from zero | NO (state machine handles both) |
| 5 | iOS escape hatch at measured cap | Disable escape hatch on iOS | NO (pure-TS default unaffected) |
| 6 | IG ships in v1 | IG drops from v1 | NO (scope, not architecture) |
| 7 | Proactive re-extraction threshold set | Reactive-only re-extraction | NO (state machine handles both) |
| 8 | iOS large-file share works | Fallback to `<a download>` / chunking | NO (UX, not architecture) |

**Only Experiment 1 changed the architecture** (NEW-AD-A). All others affect implementation details, performance, cost, or scope — not the architectural boundaries defined in Pass 3.

---

## END OF EXPERIMENT STATUS

For the architecture spec, see `planning/PASS-4-ARCHITECTURE.md`. For diagrams and flows, see `planning/PASS-4-SYSTEM-BLUEPRINT.md`.
