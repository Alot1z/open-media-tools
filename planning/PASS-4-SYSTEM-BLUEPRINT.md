# PASS 4 — SYSTEM BLUEPRINT

**Project:** open-media-tools
**Repository:** `Alot1z/open-media-tools`
**Pass:** 4 (System Blueprint — diagrams + flows)
**Date:** 2026-08-17
**Status:** COMPLETE
**Companion document:** `planning/PASS-4-ARCHITECTURE.md` (authoritative spec)

This blueprint contains the visual/flow view of the architecture. For prose specifications, see the architecture document. Diagrams are ASCII so they render in any markdown viewer and diff cleanly.

---

## 1. System Overview

```
                         ┌─────────────────────────────────────────────┐
                         │              USER (browser)                 │
                         │   desktop Chromium/Firefox · iOS Safari ·   │
                         │              Android Chrome                 │
                         └──────────────────┬──────────────────────────┘
                                            │ HTTPS
                                            │
                  ┌─────────────────────────┼─────────────────────────┐
                  │                         │                         │
                  ▼                         ▼                         ▼
        ┌──────────────────┐    ┌────────────────────┐    ┌────────────────────┐
        │  GitHub Pages    │    │  Remote Service    │    │  jsDelivr npm CDN  │
        │  (app shell)     │    │  (Node.js, VPS)    │    │  (FFmpeg.wasm)     │
        │                  │    │                    │    │                    │
        │ • HTML/JS/CSS    │    │ POST /extract      │    │ @ffmpeg/core       │
        │ • ServiceWorker  │    │ GET  /proxy?sig=   │    │ application/wasm   │
        │ • .nojekyll      │    │ GET  /health       │    │ CORS: *            │
        │ • CSP <meta>     │    │                    │    │ immutable cache    │
        │ • 404.html SPA   │    │ Stateless,        │    │                    │
        │                  │    │ SSRF-defended,     │    │ (loaded on demand  │
        │ NO headers       │    │ rate-limited       │    │  only — escape     │
        │ (no COOP/COEP)   │    │                    │    │  hatch)            │
        └──────────────────┘    └─────────┬──────────┘    └────────────────────┘
                                          │ egress (allowlisted)
                          ┌───────────────┼───────────────┐
                          │               │               │
                          ▼               ▼               ▼
                ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
                │ per-platform │  │ yt-dlp       │  │ residential      │
                │ JS scrapers  │  │ (breadth)    │  │ proxy (IG/FB)    │
                │ (tiktok/ig/  │  │ subprocess   │  │ BrightData/      │
                │  facebook)   │  │              │  │ Oxylabs          │
                └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘
                       │                 │                   │
                       └────────┬────────┴───────────────────┘
                                ▼
                   ┌────────────────────────────┐
                   │  Platform CDNs             │
                   │  (TikTok/IG/FB video CDNs) │
                   │  Referer-gated, signed URLs│
                   └────────────────────────────┘

        ┌──────────────────────────────────────────────────────────────┐
        │  GitHub Actions (CI/CD — NOT runtime)                       │
        │  build → test → deploy Pages · tag → Release (tarball) ·    │
        │  scheduled: dep updates + extractor health                  │
        └──────────────────────────────────────────────────────────────┘
```

**Three runtime surfaces:** GitHub Pages (app shell), Remote Service (extraction + proxy), jsDelivr (escape-hatch WASM).
**One build surface:** GitHub Actions.
**Hard constraint:** Pages cannot set HTTP headers → no SharedArrayBuffer → no multi-threaded WASM. v1 uses no WASM by default.

---

## 2. Component Diagram (browser internals)

```
┌──────────────────────────── BROWSER (origin: *.github.io) ────────────────────────────┐
│                                                                                        │
│  ┌──────────────────────────── MAIN THREAD ────────────────────────────┐              │
│  │                                                                      │              │
│  │  React UI (Vite-built SPA)                                           │              │
│  │   ├─ URLInput                                                        │              │
│  │   ├─ VariantPicker (shows MediaResult)                               │              │
│  │   ├─ DownloadProgress (subscribes to worker postMessage)             │              │
│  │   ├─ History (reads IndexedDB)                                       │              │
│  │   └─ ErrorBoundary                                                   │              │
│  │                                                                      │              │
│  │  Zustand store (view-mode, current job, caps)                        │              │
│  │  CapabilityState (computed once: webCodecs, opfs, shareFiles, ...)   │              │
│  │  PlatformAdapter registry (plain map: tiktok/ig/fb/direct)           │              │
│  │  RemoteClient (/extract, /proxy, /health)                            │              │
│  │  planMediaProcessing(input, caps) → MediaPlan                        │              │
│  │  SaveHandler (<a download> | navigator.share)                        │              │
│  │                                                                      │              │
│  └──────┬───────────────────────────────────────────────┬───────────────┘              │
│         │ postMessage (typed protocol)                  │ postMessage                  │
│         ▼                                               ▼                              │
│  ┌──────────────────────┐                  ┌─────────────────────────────┐            │
│  │  ServiceWorker       │                  │  DedicatedWorker (per job)  │            │
│  │  (app-shell cache)   │                  │                             │            │
│  │   • caches HTML/JS/  │                  │  Download streamer          │            │
│  │     CSS/fonts        │                  │   fetch(url|/proxy) →       │            │
│  │   • update notify    │                  │     OPFS syncHandle.write   │            │
│  │   • NO media intercept│                 │   • track bytesWritten      │            │
│  │   • NO background sync│                 │   • resume on expiry/fail   │            │
│  └──────────────────────┘                  │                             │            │
│                                            │  Media processor            │            │
│                                            │   • Mediabunny (remux)      │            │
│                                            │   • mp4box.js (MP4 fallback)│            │
│                                            │   • WebCodecs (decode/enc)  │            │
│                                            │   • FFmpeg.wasm (escape,    │            │
│                                            │      lazy from jsDelivr)    │            │
│                                            └──────────┬──────────────────┘            │
│                                                       │                              │
│  ┌──────────────────── STORAGE ────────────────────────┼──────────────────┐           │
│  │  OPFS (sync access handle, worker)  ◄───────────────┘  large files    │           │
│  │  IndexedDB  ◄─── metadata, history, queue, resume offsets             │           │
│  │  Cache API (via SW) ◄── app shell                                    │           │
│  └──────────────────────────────────────────────────────────────────────┘           │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Data-Flow Diagram (the entities and what flows between them)

```
                    ┌──────────┐
                    │   User   │
                    └────┬─────┘
                         │ 1. paste URL
                         ▼
                ┌────────────────┐
                │  Browser UI    │
                │  (main thread) │
                └────┬───────┬───┘
         2. matchUrl │       │ 3. POST /extract {url}
            (adapter)│       ▼
                     │  ┌──────────────┐         4. fetch HTML/JSON
                     │  │   Remote     │ ───────────────────────────► Platform
                     │  │   Service    │ ◄─────────────────────────── (TikTok/IG/FB)
                     │  │  (Node.js)   │         5. parsed MediaResult
                     │  └──────┬───────┘
                     │         │ 6. MediaResult (JSON, CORS headers)
                     │  ◄──────┘
                     │
                     │ 7. user picks variant
                     │ 8. planMediaProcessing → MediaPlan
                     │
            ┌────────┴────────┐
            │ 9. postMessage  │  {variant, plan, job}
            ▼                 │
    ┌─────────────────┐       │
    │ DedicatedWorker │       │
    └────────┬────────┘       │
             │ 10. fetch(variant.url)  ──direct──►  CDN (if CORS-permissive)
             │    │                                │
             │    │ 11. CORS fail / proxyRequired  │
             │    ▼                                │
             │  fetch(/proxy?u=...&sig=...)  ──►  Remote /proxy ──► CDN (Referer injected)
             │    │                                │
             │    ▼ 12. streamed bytes             │
             │  OPFS.write(chunk)  ◄───────────────┘
             │    │
             │ 13. (if plan ≠ passthrough) Mediabunny/WebCodecs/FFmpeg
             │    │
             │ 14. OPFS (final file)
             │    │
             │ 15. postMessage {complete, fileRef}
             ▼
    ┌─────────────────┐
    │  Browser UI     │ 16. <a download> Blob  OR  navigator.share({files})
    │  (save-to-OS)   │
    └─────────────────┘
```

---

## 4. Runtime-Flow Diagram (where each step executes)

```
STEP                          EXECUTES IN          NOTES
─────────────────────────────────────────────────────────────────────────
capability detection          browser main         once, at startup
platform identification       browser main         matchUrl on adapter map
extraction (TikTok/IG/FB)     REMOTE               browser POSTs /extract
extraction (direct URL)       browser main         no remote call
MediaResult display           browser main         React render
media planning                browser main         planMediaProcessing()
media fetch                   browser WORKER       stream → OPFS
  (direct CDN)                browser WORKER       if CORS-permissive
  (proxied)                   REMOTE + WORKER      /proxy streams to worker
media processing              browser WORKER       Mediabunny/mp4box/WebCodecs/FFmpeg
  passthrough                 (none)               bytes already in OPFS
  remux (Mediabunny)          browser WORKER       pure TS
  transcode (WebCodecs)       browser WORKER       progressive (audio iOS 26+)
  escape hatch (FFmpeg.wasm)  browser WORKER       lazy-loaded from jsDelivr
storage                       browser WORKER       OPFS + IndexedDB
save to disk                  browser main         <a download> / navigator.share
```

---

## 5. Extraction Flow (detailed)

```
                          ┌─────────────────────┐
                          │ user pastes URL     │
                          └──────────┬──────────┘
                                     ▼
                          ┌─────────────────────┐
                          │ URL.parse + validate│
                          └──────────┬──────────┘
                                     ▼
                          ┌─────────────────────┐
                          │ matchUrl on adapters│  (tiktok → ig → fb → direct)
                          └──────────┬──────────┘
                                     │
                       ┌─────────────┴─────────────┐
                       │                           │
                  needsRemote?                needsRemote?
                  = false (direct)            = true (tiktok/ig/fb)
                       │                           │
                       ▼                           ▼
            ┌────────────────────┐    ┌─────────────────────────┐
            │ build synthetic    │    │ POST /extract {url}      │
            │ MediaResult from   │    │  → remote validates URL  │
            │ the URL itself     │    │  → selects scraper       │
            │ (variant.url =     │    │  → if ig/fb: residential │
            │  user's URL)       │    │    proxy egress          │
            └─────────┬──────────┘    │  → fetch platform HTML   │
                      │               │  → parse, resolve URLs   │
                      │               │  → sign proxy-candidate  │
                      │               │    URLs (HMAC, 60s TTL)  │
                      │               │  → cache 30-60s          │
                      │               │  → return MediaResult    │
                      │               └────────────┬─────────────┘
                      │                            │
                      └─────────────┬──────────────┘
                                    ▼
                          ┌─────────────────────┐
                          │ MediaResult         │
                          │  {media[], metadata,│
                          │   expiresAt,        │
                          │   proxyRecommended} │
                          └──────────┬──────────┘
                                     ▼
                          ┌─────────────────────┐
                          │ UI: variant picker  │
                          │ (quality/format)    │
                          └──────────┬──────────┘
                                     ▼
                              user picks variant
                                     │
                                     ▼  (proceed to Media Flow §6)
```

**Extraction failure branches:**
- no adapter matches → try yt-dlp breadth (remote, generic flag) OR fall back to `direct` adapter
- 5xx/network → retry 2× backoff → then "service unavailable, try direct URL" (zero-server degraded mode [AD-2])
- 4xx → surface specific error (UNSUPPORTED_PLATFORM / URL_NOT_ALLOWED)
- 429 → "rate limited, retry in Retry-After seconds"
- scraperStale → "platform may have changed — being fixed" (server-side fix)

---

## 6. Media Flow (detailed, the download+process pipeline)

```
                  ┌──────────────────────────────────┐
                  │ variant picked + MediaPlan chosen│
                  └───────────────┬──────────────────┘
                                  ▼
              ┌───────────────────────────────────────────┐
              │ spawn DedicatedWorker(jobId)              │
              └───────────────┬───────────────────────────┘
                              ▼
              ┌───────────────────────────────────────────┐
              │ open OPFS syncHandle for jobId            │
              │ load resumeOffset from IndexedDB (if any) │
              └───────────────┬───────────────────────────┘
                              ▼
              ┌───────────────────────────────────────────┐
              │ plan.action?                              │
              └─────┬──────────────┬──────────────┬───────┘
                    │              │              │
             passthrough      remux/         escape-hatch
                    │         transcode            │
                    │              │              │
                    ▼              ▼              ▼
         fetch(url|/proxy,  fetch → OPFS    (lazy-load FFmpeg.wasm
           stream→OPFS)     → Mediabunny/    from jsDelivr first)
                    │       mp4box/WebCodecs       │
                    │              │              │
                    │              ▼              ▼
                    │        processed → OPFS ◄───┘
                    │              │
                    └──────┬───────┘
                           ▼
              ┌───────────────────────────────────────────┐
              │ OPFS file complete                         │
              │ write DownloadJob metadata to IndexedDB   │
              └───────────────┬───────────────────────────┘
                              ▼
              ┌───────────────────────────────────────────┐
              │ postMessage {complete, fileRef} to main   │
              └───────────────┬───────────────────────────┘
                              ▼
              ┌───────────────────────────────────────────┐
              │ SAVE (main thread, needs user gesture)    │
              │  desktop: <a download> Blob               │
              │  iOS: navigator.share({files:[File]})     │
              │  Chrome desktop: showSaveFilePicker (opt) │
              └───────────────────────────────────────────┘
```

**Fetch sub-flow (with resume-on-expiry state machine):**

```
fetch(variant.url or /proxy?sig=..., Range: bytes=offset- if resuming)
  │
  ├─ 200/206 OK → stream body → OPFS.write(chunk) → track bytesWritten
  │      │
  │      ├─ complete → close handle → proceed
  │      │
  │      └─ chunk failure → retry from bytesWritten (if Range supported)
  │
  ├─ 403 / CORS error / UPSTREAM_EXPIRED → URL expired
  │      │
  │      ├─ re-extract (call /extract again, max 3)
  │      ├─ resume with new URL + Range (if supported) OR restart from 0
  │      └─ after 3 re-extracts → FAIL
  │
  ├─ 429 → wait Retry-After → resume
  │
  └─ network error → retry 2× backoff → resume from bytesWritten (if Range)
                       │
                       └─ after 2 retries → FAIL (keep partial OPFS for manual retry)
```

---

## 7. Failure Flow (end-to-end failure handling)

```
                          ┌─────────────────────────┐
                          │  subsystem reports      │
                          │  typed error            │
                          └────────────┬────────────┘
                                       ▼
                          ┌─────────────────────────┐
                          │  classify error code    │
                          └────────────┬────────────┘
                                       │
            ┌──────────────┬───────────┼───────────┬──────────────┐
            │              │           │           │              │
         recoverable    retryable   degraded    fatal        user-action
            │              │           │           │              │
            ▼              ▼           ▼           ▼              ▼
      auto-retry      retry with   zero-server   surface       show action
      (backoff)       backoff      degraded mode "failed"      ("clear
                      (max N)      (direct URL)               storage")
            │              │           │           │              │
            └──────────────┴───────────┴───────────┴──────────────┘
                                       ▼
                          ┌─────────────────────────┐
                          │  log (client warn/err, │
                          │  remote ERROR)         │
                          │  update DownloadJob    │
                          │  status in IndexedDB   │
                          └─────────────────────────┘
```

(Full failure matrix in `PASS-4-ARCHITECTURE.md` §12.)

---

## 8. Deployment Flow

```
┌─────────────── DEVELOPER ───────────────┐
│  git push origin main                   │
└──────────────────┬──────────────────────┘
                   ▼
        ┌──────────────────────┐
        │  GitHub Actions: CI  │  (on push/PR)
        │  • install           │
        │  • lint + typecheck  │
        │  • test              │
        │  • build (vite)      │
        │  • artifact validate │  (size < 1GB, no secrets)
        │  • security (audit)  │
        └──────────┬───────────┘
                   │ green
                   ▼
        ┌──────────────────────┐
        │  deploy-pages        │  (actions/deploy-pages)
        │  uploads dist/       │
        └──────────┬───────────┘
                   ▼
        ┌──────────────────────┐
        │  GitHub Pages LIVE   │  (app shell, .nojekyll, CSP meta)
        └──────────────────────┘


┌─────────────── TAG v* ──────────────────┐
│  git tag v1.2.3 && git push --tags      │
└──────────────────┬──────────────────────┘
                   ▼
        ┌──────────────────────┐
        │  release workflow    │
        │  • build             │
        │  • source tarball    │
        │  • GitHub Release    │  (NO browser-fetched WASM — NEW-AD-A)
        └──────────────────────┘


┌─────────────── SCHEDULED (cron) ────────┐
│  • dependency-update (yt-dlp, libs)     │  idempotent + push-triggered
│  • extractor-health-check (/health)     │
└─────────────────────────────────────────┘


┌─────────────── REMOTE SERVICE ──────────┐
│  Deployed independently (VPS/container) │  NOT via Actions
│  • Fly.io / Railway / Hetzner VPS       │
│  • env: RESIDENTIAL_PROXY_*, HMAC_SECRET│
└─────────────────────────────────────────┘
```

---

## 9. Trust Boundaries

```
╔══════════════════════════════════════════════════════════════════════╗
║  TRUST ZONE 1: USER DEVICE                                             ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  Browser (attacker-controlled environment: devtools, user can  │  ║
║  │  modify anything client-side)                                   │  ║
║  │  ┌──────────────────────────────────────────────────────────┐  │  ║
║  │  │  App shell (from GitHub Pages — trusted origin)          │  │  ║
║  │  │  CSP-locked to self + jsDelivr + remote origin           │  │  ║
║  │  └──────────────────────────────────────────────────────────┘  │  ║
║  │  Storage (OPFS/IDB/Cache) — user can clear; evictable on iOS   │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
                                    │
                          HTTPS (CORS, signed URLs)
                                    │
╔══════════════════════════════════════════════════════════════════════╗
║  TRUST ZONE 2: REMOTE SERVICE (operated by us)                        ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  Treats ALL input as hostile (URLs are user-supplied)           │  ║
║  │  SSRF defense: allowlist + private-IP reject + cloud-metadata   │  ║
║  │  Signed proxy URLs (HMAC, 60s TTL)                              │  ║
║  │  Rate-limited per IP                                             │  ║
║  │  Secrets in env only (HMAC_SECRET, proxy creds) — never client  │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
                                    │
                          allowlisted egress only
                                    │
╔══════════════════════════════════════════════════════════════════════╗
║  TRUST ZONE 3: EXTERNAL (platforms + CDNs + proxies)                  ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  TikTok/Instagram/Facebook — UNTRUSTED (anti-bot, ToS)          │  ║
║  │  Platform CDNs — UNTRUSTED (signed URLs, Referer-gated)         │  ║
║  │  Residential proxy — SEMI-TRUSTED (paid provider)               │  ║
║  │  jsDelivr CDN — TRUSTED for delivery (SRI verifies integrity)   │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════╗
║  TRUST ZONE 4: BUILD/CI (GitHub Actions)                              ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │  permissions: contents: read by default; escalate per-job       │  ║
║  │  Actions pinned to SHA; no pull_request_target run: injection   │  ║
║  │  Secrets in GitHub Secrets — never in code/logs/commits         │  ║
║  │  OIDC for any cloud deploys                                     │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
```

**What crosses each boundary:**
- Zone 1 → Zone 2: post URL (public), OPTIONS preflight, signed-proxy GET requests. **Never** credentials/cookies/history.
- Zone 2 → Zone 3: HTTP fetches to allowlisted platform hosts + CDNs; residential-proxy traffic for IG/FB.
- Zone 1 → jsDelivr (Zone 3): `fetch()` of `@ffmpeg/core` WASM (SRI-verified).
- Zone 4 → Zone 1: built `dist/` deployed to Pages. **No secrets** in the bundle (verified by `gitleaks` in CI).
- Zone 4 → Zone 2: remote service deployed independently (not via Actions); secrets provisioned in the service env directly.

---

## END OF SYSTEM BLUEPRINT

For the authoritative prose specification, see `planning/PASS-4-ARCHITECTURE.md`. For experiment status, see `planning/PASS-4-EXPERIMENT-STATUS.md`.
