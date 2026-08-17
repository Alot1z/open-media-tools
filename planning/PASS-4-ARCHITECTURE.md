# PASS 4 — FINAL SYSTEM ARCHITECTURE

**Project:** open-media-tools
**Repository:** `Alot1z/open-media-tools`
**Pass:** 4 (Final System Architecture)
**Date:** 2026-08-17
**Status:** COMPLETE

**Inputs consumed (read from disk, not summaries):**
- `PASS-1-RESEARCH-REPORT.md` (commit `4a79534`)
- `PASS-2-RED-TEAM-REPORT.md` (commit `2cc2061`)
- `planning/PASS-3-DECISION-LEDGER.md` (commit `aa12a57`, 24 architectural decisions AD-1..AD-24)
- `.research/experiment-1-gh-releases-cors-RESULT.md` (NEW — Exp 1 executed this pass)

**Traceability convention:** Every major architecture decision references its Pass 3 identifier (`AD-x`). Decisions introduced by Pass 4 itself are marked `NEW-AD-x` with justification. The chain is: PASS 1 research → PASS 2 red-team → PASS 3 decision (`AD-x`) → PASS 4 architecture.

---

## 0. Executive Architecture Summary

open-media-tools is a **browser-executed, remote-assisted** media downloader [AD-1]. The system has three runtime surfaces and one build surface:

1. **Browser (GitHub Pages)** — a static React+Vite+TypeScript SPA. Holds all UI, orchestration, stream-to-storage, and local media processing (Mediabunny + mp4box.js + WebCodecs). No WASM in the default flow.
2. **Remote service (Node.js on a VPS/container)** — a stateless extractor + media-byte proxy with three endpoints (`/extract`, `/proxy`, `/health`). Handles only what the browser *cannot*: CORS-bypassed extraction and Referer-gated media fetching for TikTok/Instagram/Facebook.
3. **CDN (jsDelivr npm)** — serves the FFmpeg.wasm escape-hatch asset on demand. *(NEW-AD-A: GitHub Releases was the planned host, but Experiment 1 proved it sends no CORS headers — see §X.)*
4. **CI (GitHub Actions)** — build, test, deploy to Pages, release. Never the download runtime [AD-21].

The browser is the execution surface; the remote service is the operational core for extraction. This is honest framing — Pass 2 §4.2 established that "minimal remote" is misleading because the service carries the majority of operational complexity (extraction, Referer injection, residential-proxy egress, SSRF defense, rate limiting, media proxying).

**The single load-bearing constraint that shapes everything:** GitHub Pages cannot set HTTP response headers [Pass 1 §4.3; Pass 2 §9]. Therefore `SharedArrayBuffer` is unavailable, therefore multi-threaded WASM is impossible, therefore the architecture is **single-threaded WASM at most** — and v1 uses **no WASM at all** by default [AD-6, AD-12].

**The second load-bearing constraint:** iOS Safari evicts all origin storage after 7 days of no use and suspends JS on tab background [Pass 1 §6; Pass 2 §11]. Therefore the architecture treats local storage as **ephemeral on iOS**, hands downloads to the OS Share Sheet immediately, and runs downloads foreground-only with resume-on-revisit.

---

## 1. Frontend

**Decision traceability:** AD-4 (Vite), AD-4 (React), AD-4 (TypeScript); REJECT Next.js [Pass 3 §9].

### 1.1 Stack
- **TypeScript** (strict) — single language across client and remote service [AD-9: no second language].
- **React 18+** — UI components. Chosen for contributor pool and ecosystem over smaller frameworks (Svelte/Solid would save ~30-50KB gzipped but cost contributor friction) [Pass 2 §15].
- **Vite** — build tool and dev server. Content-hashed output defeats the Pages 10-minute `Cache-Control: max-age=600` [Pass 1 §4.3]. ES-module-native. Lazy-loadable chunks via dynamic `import()`.

### 1.2 Application state
- **Zustand** for client UI state (URL input, current job, capability flags, UI flags). Minimal, no provider tree.
- **No global server-state cache** (TanStack Query not needed) — extraction results are ephemeral per-job, not a shared cache. The remote service caches server-side [AD-13].
- **Worker state** lives in the worker (download progress, OPFS handles); the UI subscribes via `postMessage` updates.

### 1.3 Routing
- Single route (`/`) per the Z.ai Code sandbox constraint, but the app supports **view modes** (not routes) toggled by UI state: `idle`, `extracting`, `downloading`, `processing`, `complete`, `error`. If deployed standalone (not in the sandbox), hash-based routing (`#/history`, `#/settings`) is available — no server routing on Pages [Pass 1 §4.5: SPA fallback via 404.html trick].

### 1.4 Initial bundle budget [AD-16 / Pass 3 §16]
- **< 300 KB gzipped** initial bundle: React + Vite runtime + app shell + Zustand + capability detection + the PlatformAdapter registry (static imports for TikTok/IG/FB only; others lazy).
- **0 KB WASM** in initial bundle.
- **Lazy-loaded chunks:** Mediabunny + mp4box.js (~50-100KB) on first download; per-platform adapter code for non-initial platforms; FFmpeg.wasm escape-hatch loader only when triggered.

---

## 2. Browser Runtime (threads and workers)

**Decision traceability:** AD-11 (worker architecture), AD-14 (SW app-shell only).

### 2.1 Threads

| Thread | Responsibility | Lifecycle |
|---|---|---|
| **Main thread** | React rendering, user input, capability detection, orchestration (calls workers + remote), URL validation, PlatformAdapter selection, MediaResult display, save-to-disk/share invocation | Page lifetime |
| **DedicatedWorker** (1 per active job) | Download streaming (fetch → OPFS), media parsing (Mediabunny, mp4box.js), WebCodecs decode/encode, FFmpeg.wasm (escape hatch) | Per-job, terminated on completion/error |
| **ServiceWorker** | App-shell caching (offline UI); update notification | Registered on first visit; evicted on iOS after 7 days [Pass 1 §6.1] |
| **SharedWorker** (OPTIONAL) | Cross-tab download-queue coordination (single job manager across tabs) | Not in v1 default; enable only if multi-tab UX is prioritized [AD-11] |

### 2.2 Message boundaries (UI ↔ DedicatedWorker)

The UI and worker communicate via `postMessage` with a typed message protocol (see §11 Data Contracts). The boundary is **asymmetric**:

- **UI → Worker**: `start`, `pause`, `cancel`, `changeTargetFormat`.
- **Worker → UI**: `progress` (bytesWritten, totalBytes, eta), `stage` (extracting/fetching/processing/saving), `extracted` (MediaResult for display), `error` (typed error), `complete` (final file ref + save-invocation payload).

The worker **never** calls the remote service directly for extraction — extraction happens on the main thread (or a tiny separate worker) *before* the download worker starts, because the UI needs the MediaResult to render the variant picker. The download worker only handles fetch→OPFS→process→save.

**Why this split:** extraction produces a MediaResult the user must see (to pick quality/format) before download. Download is a long streaming operation that belongs in a worker. Mixing them couples UI display to worker lifecycle.

### 2.3 ServiceWorker scope [AD-14]
- Caches the app shell (HTML, JS, CSS, fonts) for offline UI.
- Does **NOT** intercept `fetch()` to media URLs (no gain — CORS still blocks; the remote `/proxy` handles Referer-gated fetch) [Pass 2 §5.1].
- Does **NOT** do Background Sync or Periodic Background Sync (unreliable on iOS; not on Firefox/Safari) [Pass 1 §6.2].
- On update (`onupdatefound`), posts a message to the UI to show "New version available — reload" (avoids stale-asset bugs).

---

## 3. Extraction

**Decision traceability:** AD-3 (capability-based selection), AD-7 (scrapers + yt-dlp), AD-8 (PlatformAdapter), AD-16 (MediaResult).

### 3.1 PlatformAdapter interface

A TypeScript interface + plain-object registry map. **No plugin engine, no registry class** [AD-8, AD-9, Pass 2 §29, §32].

```ts
interface PlatformAdapter {
  readonly platformId: string;          // 'tiktok' | 'instagram' | 'facebook' | 'direct' | ...
  readonly needsRemote: boolean;        // AD-3: declared upfront, not try/fallback
  matchUrl(url: URL): boolean;          // URL pattern match
  extract(request: ExtractionRequest): Promise<MediaResult>;
  // extract() internally calls the remote service when needsRemote=true,
  // or does browser-native extraction when needsRemote=false (e.g., 'direct' adapter).
}
```

Registry:
```ts
const adapters: Record<string, PlatformAdapter> = {
  tiktok: TikTokAdapter,
  instagram: InstagramAdapter,
  facebook: FacebookAdapter,
  direct: DirectUrlAdapter,   // zero-server: user pasted a CDN URL
  // future platforms added here as lazy dynamic imports
};
```

### 3.2 Platform identification flow
1. User submits URL.
2. Main thread iterates `adapters` in declared priority order; first `matchUrl() === true` wins.
3. If no adapter matches: fall back to the `direct` adapter (treat the URL as a direct media URL) OR show "unsupported platform, try the remote breadth backend" (which calls yt-dlp via `/extract` with a generic flag).

### 3.3 Extraction execution (where it runs)

| Adapter | `needsRemote` | Extraction runs in |
|---|---|---|
| `tiktok` | true | Remote (`POST /extract`) — browser cannot fetch tiktok.com HTML (CORS) [Pass 1 §7.2] |
| `instagram` | true | Remote (needs residential proxy egress) |
| `facebook` | true | Remote (needs residential proxy egress) |
| `direct` | false | Browser — the URL IS the media URL; skip extraction, build a synthetic MediaResult |
| future permissive-CDN platform | false | Browser — embedded-JSON scrape (rare) |

### 3.4 Extraction request/response
- **Client → Remote:** `POST /extract` with `{url, options?}` (options: preferred formats, max quality). The client sends its origin in the `Origin` header (browser does this automatically); the remote echoes it in `Access-Control-Allow-Origin`.
- **Remote → Client:** `ExtractionResponse` = `MediaResult` (see §11). Includes per-variant `expiresAt` and `proxyRequired` flags.
- **Caching:** the remote caches extraction results for 30-60s (in-memory or Redis) keyed by URL [AD-13]. The client does NOT cache extraction results (they expire; re-extract on demand).

### 3.5 Extraction failures & retries
- **Network failure / 5xx:** client retries up to 2 times with exponential backoff (1s, 3s). After that: surface "extraction service unavailable" + offer zero-server degraded mode (direct-URL download) [AD-2].
- **4xx (bad URL, unsupported platform):** no retry; surface the specific error.
- **Platform HTML structure changed (scraper returns no media):** remote returns `EXTRACTION_FAILED` with a `scraperStale` flag; client shows "platform may have changed — this is being fixed" (the scraper is a server-side fix, no client update needed).
- **Rate limit (429):** client shows "rate limited, retry in N seconds" with the `Retry-After` value.

### 3.6 Signed URLs, Referer, cookies, rate limiting, proxy usage, SSRF

These are **remote-service concerns** (the browser cannot set Referer [Pass 1 §5.4]; the browser cannot transfer cookies cross-origin). See §6 (Remote Architecture) and §9 (Security).

---

## 4. Remote Architecture

**Decision traceability:** AD-4 (minimum API), AD-5 (proxy when required), AD-7 (scrapers + yt-dlp), AD-13 (cache), AD-19 (security), Pass 2 §28.

### 4.1 API surface (3 endpoints)

| Endpoint | Method | Purpose | Auth |
|---|---|---|---|
| `/extract` | POST | Resolve a post URL to a `MediaResult` | Rate-limited per IP; no API key in v1 |
| `/proxy?u=<url>&sig=<hmac>&exp=<ts>` | GET | Stream media bytes for a Referer-gated CDN URL | Signed URL (HMAC); short TTL (60s) |
| `/health` | GET | `{status, version, uptime}` | Public; no auth |

**The API surface is genuinely small.** The operational complexity behind it is not (see §4.4).

### 4.2 Extractor service (`POST /extract`)

```
Client (browser) 
  → POST /extract {url, options?}
    → Remote validates URL (allowlist per platform) [§9.2]
    → Remote selects scraper: per-platform JS scraper for tiktok/ig/fb;
       yt-dlp subprocess for other platforms [AD-7]
    → If instagram/facebook: route through residential proxy egress [AD-7, §4.5]
    → Scraper fetches platform HTML/JSON, parses, resolves media URLs
    → Remote signs each proxy-candidate URL with HMAC + 60s TTL
    → Remote returns MediaResult {media[], metadata, expiresAt, proxyRecommended}
    → Remote caches result 30-60s keyed by URL [AD-13]
```

**Stateless:** no per-user session, no cookie storage beyond a single request's lifetime, no database. The extraction-result cache is the only state, and it's a performance optimization (safe to drop).

### 4.3 Media proxy (`GET /proxy`)

Used when:
- `proxyRecommended: true` in the MediaResult (the scraper knows the CDN requires Referer/cookies), OR
- the browser's direct `fetch(mediaUrl)` fails (CORS error / 403 / network) — the client falls back to `/proxy`.

```
Client (browser)
  → GET /proxy?u=<encoded-media-url>&sig=<hmac>&exp=<expiry-ts>
    → Remote verifies HMAC signature + not expired (≤60s)
    → Remote re-validates URL against per-platform allowlist [§9.2]
    → Remote fetches the media URL server-side, injecting:
        - Referer: https://<platform>.com/ (per adapter config)
        - User-Agent: a real-browser UA
        - Cookies: none by default (v1); if a platform requires, in-memory for this request only
    → Remote streams the response body to the client with:
        - Access-Control-Allow-Origin: <client origin>
        - Access-Control-Expose-Headers: Content-Range, Content-Length
        - Content-Type: <upstream content-type>
        - Accept-Ranges: bytes (if upstream supports Range)
    → Remote enforces: file-size cap (2GB), timeout (60s), per-IP rate limit
```

**Range support:** if the client requests `Range: bytes=N-` (for resume), the proxy forwards it to the upstream CDN and passes through the `206 Partial Content` + `Content-Range`. If the upstream doesn't support Range, the proxy returns the full 200 and the client must restart from zero [Exp 4, IMPLEMENTATION-TIME].

### 4.4 Operational complexity (honest)

The API is 3 endpoints. The implementation behind it:
- **Per-platform JS scrapers** (TikTok, Instagram, Facebook) — maintained server-side; break when platforms change HTML; monitored via success-rate alerts.
- **yt-dlp subprocess** (breadth backend) — `youtube-dl-exec` or `yt-dlp-exec` Node wrapper spawning the binary; kept version-pinned; updated via scheduled Action.
- **Residential proxy egress** (IG/FB) — BrightData/Oxylabs/Smartproxy integration; configurable per-adapter; costs ~$1-15/GB (see §4.5).
- **SSRF defense** — host allowlist, private-IP rejection, cloud-metadata block, redirect re-validation, DNS-rebinding defense (see §9).
- **Rate limiting** — per-IP token bucket (e.g., 30 req/min extract, 10 req/min proxy).
- **Signed URL minting** — HMAC-SHA256 with a server secret; 60s TTL.
- **Extraction-result cache** — in-memory `Map` (single instance) or Redis (multi-instance), 30-60s TTL.
- **Health monitoring** — `/health` for the client's degraded-mode check and external uptime monitoring.
- **Multi-instance + load balancing** — when traffic grows beyond a single VPS.

This is **not "minimal"** in implementation. It is minimal in *API surface* and *what it does* (extraction + proxy only — no transcoding, no storage, no accounts). Pass 2 §4.2 established this honest framing.

### 4.5 Residential proxy egress

- **Required for:** Instagram, Facebook (datacenter IPs are blocked) [Pass 1 §7.2; Exp 6, NON-BLOCKING but scope-condition for IG/FB].
- **Provider:** BrightData or Oxylabs (configurable; one chosen at deploy time).
- **Routing:** the extractor service selects a residential egress per-adapter (`adapter.proxyEgress = 'residential' | 'datacenter'`).
- **Cost control:** residential proxying is used **only for extraction** (small HTML/JSON fetches, ~50-500KB). Media bytes are **NOT** proxied through residential egress — the `/proxy` endpoint fetches media via the server's datacenter IP if the CDN allows, or via a cheaper datacenter proxy. This avoids paying residential rates (~$1-15/GB) for large media bytes.
- **Fallback if residential unavailable:** IG/FB adapters return `EXTRACTION_FAILED` with `proxyEgressUnavailable`; client shows "Instagram/Facebook temporarily unavailable."

### 4.6 Failure handling (remote)

| Failure | Remote behavior | Client-visible |
|---|---|---|
| Scraper exception | Return `EXTRACTION_FAILED` + `scraperStale` flag | "Platform may have changed — being fixed" |
| yt-dlp timeout (30s) | Return `EXTRACTION_TIMEOUT` | "Extraction timed out, retry" |
| Residential proxy down | Return `EXTRACTION_FAILED` + `proxyEgressUnavailable` | "Instagram/Facebook temporarily unavailable" |
| Rate limit hit | Return 429 + `Retry-After` | "Rate limited, retry in Ns" |
| SSRF attempt (private IP) | Return 400 `URL_NOT_ALLOWED` | "URL not allowed" |
| File-size cap exceeded (proxy) | Abort upstream; return 413 | "File too large" |
| Proxy upstream 403 (expired signed URL) | Return 502 `UPSTREAM_EXPIRED`; client re-extracts | "URL expired, re-extracting…" |

---

## 5. Media Processing

**Decision traceability:** AD-6 (pure-TS default + escape hatch), AD-17 (planning function).

### 5.1 Pipeline (the full media operation lifecycle)

```
USER INPUT (URL)
    ↓ [main thread]
CAPABILITY DETECTION (compute `capabilities` object once) [AD-10]
    ↓ [main thread]
PLATFORM IDENTIFICATION (matchUrl against adapter map) [AD-8]
    ↓ [main thread]
EXTRACTION (adapter.extract → POST /extract, OR browser-native for 'direct') [AD-3, AD-7]
    ↓ [remote OR browser] → returns MediaResult
MEDIA RESULT DISPLAY (UI shows variants: quality/format picker)
    ↓ [main thread] user picks a variant
MEDIA PLANNING (planMediaProcessing(input) → MediaPlan) [AD-17]
    ↓ [main thread] decides: passthrough | remux-via-mediabunny | remux-via-mp4box | escape-hatch-ffmpeg
MEDIA FETCH (DedicatedWorker: fetch(variant.url) or fetch(/proxy?...)) [AD-5, AD-15]
    ↓ [worker] stream → OPFS sync access handle
MEDIA PROCESSING (DedicatedWorker: passthrough | Mediabunny remux | WebCodecs decode/encode | FFmpeg.wasm)
    ↓ [worker] output → OPFS
STORAGE (OPFS file complete; metadata → IndexedDB) [AD-12]
    ↓ [worker → main thread]
DOWNLOAD / SHARE (<a download> Blob on desktop; navigator.share({files}) on iOS) [AD-22]
```

### 5.2 Media planning function (`planMediaProcessing`)

A **function**, not a planner class [AD-9, AD-17, Pass 2 §33]. Decision tree:

```ts
function planMediaProcessing(input: MediaInput, caps: CapabilityState): MediaPlan {
  // 1. PASSTHROUGH: input container/codec matches target, no transform needed
  if (input.container === input.targetContainer && input.codec === input.targetCodec) {
    return { action: 'passthrough' };
  }
  // 2. REMUX (no re-encode): container change only, codec supported by target container
  if (Mediabunny.supports(input.container) && Mediabunny.supports(input.targetContainer)) {
    return { action: 'remux', tool: 'mediabunny' };
  }
  // 3. MP4BOX for MP4-family inspection/fragmentation if Mediabunny insufficient
  if (input.container === 'mp4' || input.container === 'mov') {
    return { action: 'remux', tool: 'mp4box' };
  }
  // 4. WEBCODECS re-encode (if codec change needed AND WebCodecs available)
  if (input.targetCodec && caps.webCodecsVideo && codecSupported(input.codec, input.targetCodec, caps)) {
    return { action: 'transcode', tool: 'webcodecs', muxer: 'mediabunny' };
  }
  // 5. ESCAPE HATCH: FFmpeg.wasm (loaded lazily from jsDelivr)
  return { action: 'escape-hatch', tool: 'ffmpeg-wasm' };
}
```

### 5.3 Tool roles

| Tool | Role | When | Where |
|---|---|---|---|
| **Browser-native codecs** (`<video>`/`<audio>` playback) | Determines "can we just save the source as-is?" | If source container/codec is browser-playable → passthrough | n/a (just a check) |
| **Mediabunny** (pure TS) | Demux/mux/remux MP4/MOV/WebM/MKV/HLS/WAVE/MP3/Ogg/ADTS/FLAC/MPEG-TS | Default remux path | DedicatedWorker |
| **mp4box.js** (BSD-3) | ISOBMFF (MP4) inspection/fragmentation | Fallback for MP4-family edge cases Mediabunny doesn't handle | DedicatedWorker |
| **WebCodecs** | Frame decode/encode (progressive) | When codec change needed; video universal, audio iOS 26+ | DedicatedWorker |
| **FFmpeg.wasm** (escape hatch) | Exotic codec/operation the pure-TS path can't handle | Rare; loaded lazily | DedicatedWorker |

### 5.4 Fallback hierarchy (when a browser can't support an operation)

1. **Passthrough** (no processing) — always preferred if container/codec match.
2. **Mediabunny remux** (pure TS) — default for container changes.
3. **mp4box.js** — fallback for MP4-family edge cases.
4. **WebCodecs re-encode** — when codec change needed AND available (video universal, audio iOS 26+).
5. **FFmpeg.wasm escape hatch** — last resort; loaded lazily from jsDelivr (NEW-AD-A).

**If FFmpeg.wasm fails on iOS (memory cap, Exp 5 IMPLEMENTATION-TIME):** surface "this media can't be processed on iOS; try on desktop" — do not crash. The pure-TS path covers the common cases; the escape hatch is for rare ones.

### 5.5 FFmpeg.wasm escape hatch — loading protocol

- **Not bundled.** Not in the initial bundle, not in any lazy app chunk.
- **Loaded on demand** via dynamic `import()` of a thin loader module, which fetches `@ffmpeg/core` from jsDelivr npm (NEW-AD-A):
  - URL: `https://cdn.jsdelivr.net/npm/@ffmpeg/core@<pinned-version>/dist/umd/ffmpeg-core.wasm`
  - SRI hash verification (Subresource Integrity) on the `.wasm` file [AD-19].
  - Pinned to an exact version (not `@latest`) — supply-chain safety.
- **Single-threaded build only.** Multi-threaded requires SharedArrayBuffer → COOP/COEP → unavailable on Pages [Pass 3 §9 REJECT].
- **Memory cap:** set `MAXIMUM_MEMORY` to ~256-512MB (Exp 5 will refine; default 384MB until then). On iOS, if instantiation fails, fall back to "not supported on this device."
- **Trigger conditions** (when the escape hatch fires) [AD-6]:
  1. Input container not supported by Mediabunny, OR
  2. Codec not decodable by WebCodecs (and not passthrough), OR
  3. Operation not supported by pure-TS path (burn-in subtitles, complex filters), OR
  4. User explicitly requests a format requiring re-encode to a codec pure-TS can't produce.

---

## 6. Storage

**Decision traceability:** AD-12 (three stores), AD-13 (caching).

### 6.1 Store roles (no overlap)

| Store | Role | Data | Persistence |
|---|---|---|---|
| **OPFS** (sync access handle in worker) | Large-file staging | Downloaded media bytes (in-progress + completed before save) | Ephemeral on iOS (7-day eviction); cleared after save-to-OS |
| **IndexedDB** | Metadata | Download history, queue state, user settings, capability snapshot, resume offsets | Ephemeral on iOS (7-day eviction); persistent on desktop (with `persist()`) |
| **Cache API** (via ServiceWorker) | App-shell resources | HTML, JS, CSS, fonts for offline UI | Subject to SW eviction on iOS |
| **Memory** | Transient stream chunks | fetch body chunks in flight (implicit, never whole file) | Process lifetime |

**No SQLite WASM in v1** [AD-12, Pass 2 §19]. IndexedDB suffices for record types (history list, queue, settings KV). Revisit wa-sqlite only if full-text search across history or complex analytics are needed.

### 6.2 OPFS usage pattern

```ts
// In DedicatedWorker:
const root = await navigator.storage.getDirectory();           // OPFS root
const fileHandle = await root.getFileHandle(jobId, { create: true });
const syncHandle = await fileHandle.createSyncAccessHandle();   // worker-only, fast
const writable = syncHandle.writable ?? syncHandle;             // stream or sync writes
// stream fetch body → syncHandle.write(chunk) → track offset in IndexedDB
await syncHandle.flush();
syncHandle.close();
```

- **Resume offset** stored in IndexedDB (`{jobId, bytesWritten, mediaUrl, expiresAt}`) so a re-foregrounded tab can resume [AD-15].
- **Cleanup:** after save-to-OS (or user discard), delete the OPFS file. On iOS, accept eviction anyway.

### 6.3 Quota / persistence behavior

- Call `navigator.storage.persist()` early (after first user gesture) — requested but not guaranteed on iOS [AD-22, Pass 1 §6.1].
- Call `navigator.storage.estimate()` before starting a large download; if `quota - usage < estimatedFileSize + headroom`, warn the user.
- **Quota failure:** OPFS write throws `QuotaExceededError`; worker reports `error{type: 'storage-full'}`; UI shows "Not enough storage — clear some downloads and retry."

### 6.4 Cleanup

- On download complete + save-to-OS: delete OPFS file, keep IndexedDB metadata (history).
- On download cancel/error: delete OPFS file, mark IndexedDB record as cancelled/failed.
- Periodic (on app load): sweep OPFS for orphaned files (no matching active job); delete them.
- iOS: accept that all of this may be wiped after 7 days anyway.

---

## 7. Large-File Handling & Download

**Decision traceability:** AD-15 (stream to OPFS, resume on expiry/failure).

### 7.1 Download state machine (the resume-on-expiry flow Pass 2 §7.1 required)

```
START
  ↓
fetch(mediaUrl or /proxy?sig=...) with Range: bytes=0- if resuming
  ↓
stream response.body → OPFS syncHandle.write(chunk); track bytesWritten in IndexedDB
  ↓
[on chunk] → postMessage progress to UI
  ↓
[on completion] → close handle; proceed to STORAGE → SAVE
  ↓
[on failure] → classify:
    - network error → retry up to 2x with backoff; resume from bytesWritten if CDN supports Range (Exp 4)
    - 403 / CORS error / UPSTREAM_EXPIRED → URL likely expired:
        - if time-since-extraction > (TTL/2) → re-extract (call /extract again, max 3 re-extractions)
        - resume from bytesWritten if new URL + Range supported, else restart from 0
    - 429 rate limit → wait Retry-After; resume
    - storage full → abort; surface error
  ↓
[after 3 re-extractions or 2 network retries] → FAIL: surface "download failed" + keep partial OPFS file for manual retry
```

### 7.2 iOS large-file behavior [AD-22]
- **Foreground-only.** If the tab is backgrounded, the fetch stalls (iOS suspends JS). On re-foreground, the worker detects the stall (no chunk for N seconds) and resumes from the last `bytesWritten` offset.
- **Memory:** never buffer the whole file. Stream chunk-by-chunk (e.g., 1MB chunks) through the worker.
- **Save:** `navigator.share({files: [File]})` — requires a user gesture (the user taps "Save"). For very large files (Exp 8, IMPLEMENTATION-TIME), if share fails, fall back to `<a download>` (less reliable on iOS) or advise "file saved in app storage — use the Share button."

### 7.3 Chrome enhancement [AD-22]
- **Background Fetch API** (Chrome 74+): if available, register the download as a background fetch so it survives tab close with a system notification. Progressive enhancement — not relied upon.

---

## 8. iOS / Safari Architecture

**Decision traceability:** AD-22, Pass 3 §17.

### 8.1 Capability detection (runtime, not assumed)

```ts
const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent) ||
              (navigator.platform === 'MacIntel' && navigator.maxTouchPoints > 1); // iPadOS desktop UA
const caps: CapabilityState = {
  webCodecsVideo: typeof VideoDecoder !== 'undefined',
  webCodecsAudio: typeof AudioDecoder !== 'undefined',    // false on iOS < 26
  opfsSyncHandle: 'createSyncAccessHandle' in FileSystemFile.prototype,
  backgroundFetch: 'BackgroundFetchManager' in window,    // false on iOS
  shareFiles: 'canShare' in navigator && 'files' in (navigator.canShare({files:[new File([''],"t")]}) ? {} : {files:[]}),
  fileSystemAccess: 'showSaveFilePicker' in window,       // false on iOS/Safari/Firefox
  crossOriginIsolated: window.crossOriginIsolated === true, // false on Pages (no COOP/COEP)
  isIOS,
};
```

### 8.2 iOS-specific behavior table

| Operation | iOS behavior | Mitigation |
|---|---|---|
| Download (large) | Foreground-only; stalls on background | Resume-on-revisit via OPFS offset [§7.2] |
| Save to disk | `<a download>` unreliable for media | `navigator.share({files})` [AD-22] |
| Local library | Evicted after 7 days | Hand off to OS immediately; no "library" feature on iOS |
| WebCodecs audio | iOS < 26 has video only | Passthrough or escape hatch (with memory cap) |
| FFmpeg.wasm | Tab crashes if linear memory too high | Cap ~256-512MB; disable if instantiation fails (Exp 5) |
| Multi-threaded WASM | Unavailable (no SAB on Pages) | Single-threaded only [Pass 3 §9 REJECT] |
| File System Access pickers | Not supported | OPFS staging + Share Sheet |

### 8.3 UX when a capability is unavailable
- **WebCodecs audio unavailable (iOS < 26):** if the user requests audio transcoding, show "audio processing needs iOS 26+; saving original audio instead" → passthrough.
- **FFmpeg.wasm can't load (iOS memory):** show "this media can't be processed on iOS; try on desktop."
- **Background Fetch unavailable:** no notification; show "keep this tab open while downloading."
- **Share Sheet fails for large file:** show "file saved in app storage; tap Save again or use a smaller file."

---

## 9. Security Architecture

**Decision traceability:** AD-19, Pass 3 §15.

### 9.1 Trust boundaries (who trusts whom)

```
[User's browser]  ──trusts──>  [GitHub Pages app shell] (CSP-locked to self + jsDelivr + remote origin)
       │
       │ (HTTPS, signed proxy URLs)
       ↓
[Remote service]  ──trusts──>  NOBODY (treats all input as hostile)
       │
       │ (allowlisted egress only)
       ↓
[Platform CDNs + residential proxy]
```

### 9.2 SSRF protection (remote extractor + proxy)

**URL validation pipeline** (every `/extract` and `/proxy` request):
1. **Parse URL** — reject malformed URLs.
2. **Host allowlist** — per-platform adapter declares allowed hosts (e.g., TikTok adapter allows `tiktok.com`, `*.tiktokcdn.com`, `*.byteicdn.com`). Reject anything else.
3. **DNS resolve** — resolve all A/AAAA records for the host.
4. **Private-IP rejection** — reject if any resolved IP is in: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16` (link-local, incl. cloud metadata `169.254.169.254`), `::1`, `fc00::/7`, `fe80::/10`.
5. **Cloud-metadata block** — explicitly reject `169.254.169.254`, `metadata.google.internal`, `metadata.azure.com`.
6. **Pin resolved IP** — connect to the specific resolved IP (defeats DNS rebinding TOCTOU).
7. **Redirect re-validation** — if the upstream returns 3xx, re-run steps 2-6 on the redirect target before following. (Or disable redirects entirely and handle manually.)

### 9.3 Remote proxy abuse prevention
- **Signed proxy URLs:** `/proxy?u=<url>&sig=<hmac-sha256>&exp=<unix-ts>`. The HMAC is computed by the extractor over `url + exp` with a server secret. The proxy verifies HMAC + `exp` not in past + TTL ≤ 60s.
- **Per-IP rate limit:** token bucket per client IP (e.g., 10 proxy requests/min).
- **File-size cap:** abort upstream at 2GB; return 413.
- **Timeout:** 60s per proxy request; abort.
- **Host allowlist:** the proxy re-validates `u` against the per-platform allowlist (the signed URL was minted by our extractor, but defense-in-depth).
- **No credential forwarding:** the proxy does not forward client cookies/headers to the upstream (only the injected Referer/UA per adapter).

### 9.4 CORS
- **Remote service → client:** sends `Access-Control-Allow-Origin: <client origin>` (the Pages domain), `Access-Control-Allow-Methods: GET, POST, OPTIONS`, `Access-Control-Allow-Headers: Content-Type`, `Access-Control-Expose-Headers: Content-Range, Content-Length`. Handles OPTIONS preflight.
- **Client → jsDelivr:** jsDelivr sends `Access-Control-Allow-Origin: *` (verified Exp 1 Probe 3).
- **Client → platform CDNs (direct fetch attempt):** usually fails CORS — fall back to `/proxy` [AD-5].

### 9.5 CSP (Content Security Policy) — via `<meta>` [AD-19, Pass 3 §9]
```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' https://cdn.jsdelivr.net;
               wasm-unsafe-eval;
               connect-src 'self' https://cdn.jsdelivr.net https://<remote-service-origin>;
               media-src 'self' blob: https://<remote-service-origin>;
               img-src 'self' data: https:;
               style-src 'self' 'unsafe-inline';
               worker-src 'self' blob:;
               object-src 'none';
               base-uri 'self';
               form-action 'none'">
```
- `wasm-unsafe-eval` is required for FFmpeg.wasm instantiation (modern CSP directive).
- `'unsafe-inline'` on style-src is a pragmatic concession for styled-components/Vite-injected styles; tighten to `'self' 'nonce-...'` in a future hardening pass.
- **COOP/COEP cannot be set** on Pages → `crossOriginIsolated === false` → no SAB → accepted [Pass 3 §9 REJECT].

### 9.6 Untrusted media / resource exhaustion
- **Worker isolation:** all media parsing runs in DedicatedWorker — a crash doesn't take down the UI.
- **Timeouts:** per-operation (e.g., 30s parse, 60s fetch).
- **File-size cap:** reject inputs > 2GB before parsing.
- **Decompression-bomb defense:** cap decompressed output size; prefer pure-TS parsers (no memory-corruption risk) over C/WASM [Pass 2 §22].
- **Content-type validation:** the proxy passes through the upstream `Content-Type`; the client verifies it matches the MediaResult variant's expected type before processing.

### 9.7 Security control ownership matrix

| Control | Browser | Remote extractor | Media proxy | CI/CD |
|---|---|---|---|---|
| CSP | ✅ (`<meta>`) | — | — | — |
| CORS (response headers) | — | ✅ | ✅ | — |
| SSRF defense | — | ✅ | ✅ | — |
| Signed URLs | verify (transparent) | mint | verify | — |
| Rate limiting | — | ✅ | ✅ | — |
| URL allowlist | — | ✅ | ✅ | — |
| File-size cap | ✅ (pre-download estimate) | ✅ | ✅ | — |
| Worker isolation | ✅ | — | — | — |
| WASM SRI | ✅ | — | — | ✅ (verify at build) |
| Secret hygiene | ✅ (no secrets client-side) | ✅ (env only) | ✅ (env only) | ✅ (GitHub Secrets) |
| Dependency audit | — | — | — | ✅ (`npm audit`, Dependabot) |
| Action injection defense | — | — | — | ✅ (permissions: read, pin SHA, no run: interpolation) |

---

## 10. GitHub Pages, Actions, Releases Architecture

### 10.1 GitHub Pages responsibilities [AD-20]
**Hosts:** the app shell only — `index.html`, JS bundles (content-hashed), CSS, fonts, `404.html` (SPA fallback), `.nojekyll`, the service worker registration script, `manifest.webmanifest` (PWA), the `<meta>` CSP, the `<meta>` theme-color.

**Does NOT host:** FFmpeg.wasm (too large + no CORS — see NEW-AD-A), the remote service, user data, platform-specific scraper code (that's server-side).

**Configuration:**
- `.nojekyll` at publish root (required for correct `.wasm`/`.js` MIME) [Pass 1 §4.3].
- Content-hashed asset URLs (Vite default) to defeat the 10-minute `Cache-Control: max-age=600`.
- SPA routing via `404.html` JS-redirect trick [Pass 1 §4.5].
- HTTPS enforced (Pages default); custom domain via `CNAME` (optional).

### 10.2 GitHub Actions responsibilities [AD-21]

**CI (on push/PR):**
- `install` (bun/npm ci with lockfile)
- `lint` (eslint)
- `typecheck` (tsc --noEmit)
- `test` (unit tests; no e2e in v1 CI)
- `build` (vite build → `dist/`)
- `artifact-validation` (check `dist/` size < 1GB Pages cap; check no secrets in bundle via `gitleaks`/`secret-scan`)
- `security-checks` (`npm audit`, Dependabot status, action-SHA-pinning check)

**Deployment (on push to main, after CI passes):**
- `deploy-pages` (`actions/deploy-pages` — uploads `dist/` artifact, deploys to Pages).

**Release (on tag `v*`):**
- `build` (same as CI)
- `upload-release-artifacts` (source tarball; release notes auto-generated). **Note:** FFmpeg.wasm is NOT uploaded here for browser-fetch (it comes from jsDelivr npm) — Releases holds only source tarballs + metadata now (NEW-AD-A).

**Scheduled maintenance (cron, idempotent + push-triggered [Pass 2 §21]):**
- `dependency-update` (Dependabot handles most; this workflow checks yt-dlp version, Mediabunny/mp4box.js versions, opens PR if outdated).
- `extractor-health-check` (calls `/health` on the remote service; if down, opens an issue).

**Actions NEVER:** serve user downloads, run extraction, proxy media [R8, AD-21].

### 10.3 GitHub Releases — revised role (NEW-AD-A)

**Original Pass 3 plan:** Releases hosts FFmpeg.wasm for browser fetch.
**Experiment 1 result:** Releases sends no CORS headers → browser fetch blocked.
**Revised role:** Releases hosts source tarballs + release metadata only. Browser-fetched WASM comes from jsDelivr npm. See §X (NEW-AD-A) below.

### 10.4 Deployment flow
```
push to main
  → Actions CI runs (lint/typecheck/test/build/validate/security)
  → if green: deploy-pages uploads dist/ → Pages live
  → if tag v*: release workflow creates Release + source tarball
scheduled cron
  → dependency-update + extractor-health-check (idempotent)
```

---

## 11. Data Contracts

**Decision traceability:** AD-16 (MediaResult), AD-8 (PlatformAdapter), Pass 2 §34. Minimal, typed, versioned.

### 11.1 `MediaResult` (normalized extraction output)
```ts
interface MediaResult {
  platform: string;                  // 'tiktok' | 'instagram' | 'facebook' | 'direct' | 'yt-dlp' | ...
  originalUrl: string;
  media: MediaVariant[];
  metadata: {
    title?: string;
    author?: string;
    thumbnailUrl?: string;
    durationMs?: number;
  };
  extractedAt: number;               // unix ms — for TTL tracking
  expiresAt?: number;                // unix ms — earliest variant expiry (for UI warning)
  proxyRecommended: boolean;         // AD-5: hint to skip direct fetch and use /proxy
}
```

### 11.2 `MediaVariant`
```ts
interface MediaVariant {
  url: string;                       // direct CDN URL (may need proxy)
  type: 'video' | 'audio' | 'image';
  container?: string;                // 'mp4' | 'webm' | 'mkv' | ...
  codec?: string;                    // 'h264' | 'h265' | 'vp9' | 'av1' | 'aac' | 'opus' | ...
  quality?: string;                  // '720p' | '1080p' | '128kbps' | ...
  width?: number;
  height?: number;
  durationMs?: number;
  bytes?: number;                    // if known (Content-Length)
  expiresAt?: number;                // unix ms — when this specific URL expires
  proxyRequired?: boolean;           // stronger than MediaResult.proxyRecommended
}
```

### 11.3 `ExtractionRequest` / `ExtractionResponse`
```ts
interface ExtractionRequest { url: string; options?: { preferredFormat?: string; maxQuality?: string; }; }
type ExtractionResponse = MediaResult | ExtractionError;
interface ExtractionError {
  error: true;
  code: 'UNSUPPORTED_PLATFORM' | 'EXTRACTION_FAILED' | 'EXTRACTION_TIMEOUT' | 'RATE_LIMITED' | 'URL_NOT_ALLOWED' | 'PROXY_EGRESS_UNAVAILABLE';
  message: string;
  retryAfter?: number;               // for RATE_LIMITED
  scraperStale?: boolean;            // for EXTRACTION_FAILED (platform HTML changed)
}
```

### 11.4 `MediaPlan`
```ts
type MediaPlan =
  | { action: 'passthrough' }
  | { action: 'remux'; tool: 'mediabunny' | 'mp4box' }
  | { action: 'transcode'; tool: 'webcodecs'; muxer: 'mediabunny' }
  | { action: 'escape-hatch'; tool: 'ffmpeg-wasm' };
```

### 11.5 `DownloadJob`
```ts
interface DownloadJob {
  jobId: string;                     // uuid
  url: string;                       // original input URL
  platform: string;
  variant: MediaVariant;
  plan: MediaPlan;
  status: 'queued' | 'extracting' | 'downloading' | 'processing' | 'saving' | 'complete' | 'failed' | 'cancelled';
  bytesWritten: number;
  totalBytes?: number;
  startedAt: number;
  completedAt?: number;
  error?: { code: string; message: string; };
  resumeOffset?: number;             // for resume-on-revisit
  reextractCount: number;            // capped at 3
}
```

### 11.6 `CapabilityState`
(See §8.1 for the full shape.)

### 11.7 Error model
Errors are typed via `code` strings (see `ExtractionError` and `DownloadJob.error`). The UI maps each code to a user-visible message + action (retry / re-extract / give up / suggest degraded mode). No untyped string errors cross the worker/remote boundary.

### 11.8 Versioning
- `MediaResult` / `ExtractionResponse`: include a `schema: 1` field (optional in v1, becomes mandatory if the contract evolves). The remote service version-pins via the `/health` endpoint's `version` field.
- Client and remote share these types via a `packages/contracts` workspace package (or duplicated TS interface with a comment noting the source of truth is the remote).

### 11.9 Serialization boundaries
- **Client ↔ Worker:** `postMessage` (structured clone; `MediaResult`, `DownloadJob`, `MediaPlan` are plain objects — cloneable).
- **Client ↔ Remote:** JSON over HTTPS.
- **Worker ↔ OPFS:** bytes (ArrayBuffer / Uint8Array chunks; transferable).
- **IndexedDB:** structured-clone of plain objects + ArrayBuffer for any blobs.

---

## 12. Failure Model (complete matrix)

| Scenario | Normal path | Expected failure | Fallback | Fatal failure | User-visible | Logged |
|---|---|---|---|---|---|---|
| Unsupported platform | adapter matches | no adapter matches | `direct` adapter or yt-dlp breadth | yt-dlp also fails | "Unsupported URL — try a direct media link" | client error + remote 400 |
| Extraction failure | /extract 200 | scraper exception | n/a (scraper fix is server-side) | persistent | "Platform may have changed — being fixed" | remote ERROR + scraperStale |
| Expired media URL | fetch 200 | fetch 403/CORS | re-extract (max 3), resume | 3 re-extracts exhausted | "Reconnecting…" then "Download failed — retry" | client warn |
| CORS failure (direct fetch) | direct fetch 200 | CORS error | fall back to /proxy | /proxy also fails | (transparent) then "Download failed" | client warn |
| Proxy failure | /proxy 200 | upstream 5xx | retry once; re-extract if expired | persistent | "Download failed — retry" | remote ERROR |
| Rate limit | 200 | 429 + Retry-After | wait + retry | repeat 429 | "Rate limited, retry in Ns" | remote warn |
| Storage quota | OPFS write ok | QuotaExceededError | n/a | n/a | "Not enough storage — clear downloads" | client error |
| iOS memory pressure | worker ok | tab crash (rare) | resume-on-revisit | repeat crash | "iOS ran out of memory — try a smaller file" | client error (if survived) |
| Unsupported codec | passthrough/remux | WebCodecs lacks codec | escape hatch FFmpeg.wasm | FFmpeg.wasm fails on iOS | "Can't process this on iOS — try desktop" | client warn |
| Malformed media | parse ok | parse throws | try alternate parser | all parsers fail | "File appears corrupt" | client error |
| Oversized media | under 2GB | over 2GB | n/a (hard cap) | n/a | "File too large (max 2GB)" | remote/client error |
| Remote service unavailable | /health 200 | /health 5xx / timeout | zero-server degraded mode (direct-URL) [AD-2] | n/a | "Service unavailable — paste a direct media URL to download anyway" | client warn |
| Signed URL expiry | sig valid | sig expired (60s) | re-extract → new sig | persistent | (transparent) | remote warn |
| GitHub Pages unavailable | app loads | app fails to load | (user retries later) | n/a | "App unavailable — try again later" | (can't log client-side if app didn't load) |

---

## 13. Performance Architecture

**Decision traceability:** AD-16, Pass 3 §16.

| Metric | Budget | Mechanism |
|---|---|---|
| Initial bundle (gzipped) | < 300 KB | React + Vite + app shell + Zustand + adapters (initial 3 only) |
| WASM in initial bundle | 0 KB | No WASM by default; FFmpeg.wasm lazy from jsDelivr |
| Mediabunny lazy-load | ~50-100 KB | Dynamic `import()` on first download |
| FFmpeg.wasm load (escape) | ~25 MB | From jsDelivr (immutable, 1yr cache); rare |
| DedicatedWorker startup | < 50 ms | Native spawn |
| Stream-to-OPFS throughput | network-bound | chunked pipe; 1MB chunks |
| iOS WASM memory cap | 384 MB default (Exp 5 refines) | emscripten `MAXIMUM_MEMORY` |
| Remote extraction latency | 1-5 s typical | + 30-60s server cache hit |
| Remote proxy overhead | +1 RTT + bandwidth | only when proxyRequired |
| Pages cache TTL | 10 min | defeated by content-hashed URLs |
| Pages bandwidth | 100 GB/mo soft | app shell only; media never through Pages |

**Where unnecessary data copies could occur (and how avoided):**
- fetch body → OPFS: use `pipeTo` / streaming `write(chunk)` — no whole-file buffer.
- OPFS → save: read OPFS file → Blob (one copy, unavoidable for `<a download>`/share) → hand off. For very large files, this is the one memory spike; on iOS, cap and warn.
- Worker → UI: `postMessage` with transferable `ArrayBuffer` where possible (zero-copy); plain objects for metadata.
- Remote proxy: stream upstream → client (don't buffer whole file server-side); use `Readable.pipe`.

---

## 14. Security / Privacy Boundary (what leaves the browser)

| Information | Leaves browser? | To whom | Conditions |
|---|---|---|---|
| Public post URL (user input) | YES | Remote `/extract` | Always (for extraction) |
| Extracted metadata (title, author, thumbnail) | YES (inbound) | From remote to browser | Always |
| Media CDN URLs | YES (inbound) | From remote to browser | Always |
| Media bytes | **NO** through Pages; direct browser↔CDN or browser↔remote `/proxy` | CDN (direct) or remote (proxy) | Direct fetch if CORS-permissive; /proxy if Referer-gated |
| Cookies | NO | never sent to remote | v1: stateless, no cookie storage |
| Credentials | NO | never | v1: no user accounts, no platform login |
| Platform auth (session) | NO | never | The remote uses residential proxy + UA, not user session |
| Download history | NO | stays in browser IndexedDB | never transmitted |
| Capability state | NO | stays in browser | used only for client-side branching |
| Error reports | OPTIONAL (opt-in) | Sentry-style endpoint | only if user enables; no media/URL data |

**The browser cannot access credentials it doesn't have.** The remote service does NOT receive user platform credentials (it uses its own egress + UA). If a future version supports "download private content," that requires a separate, explicit auth flow (out of v1 scope).

---

## 15. Repository Architecture

Derived from the architecture (not blindly copied from the prompt's example). Every directory has a clear responsibility.

```
open-media-tools/
├── README.md
├── PASS-1-RESEARCH-REPORT.md          # research (complete)
├── PASS-2-RED-TEAM-REPORT.md          # red-team (complete)
├── planning/
│   ├── PASS-3-DECISION-LEDGER.md      # decisions (complete)
│   ├── PASS-4-ARCHITECTURE.md         # THIS FILE (architecture)
│   ├── PASS-4-SYSTEM-BLUEPRINT.md     # diagrams + flows
│   └── PASS-4-EXPERIMENT-STATUS.md    # experiment status
├── .research/                          # evidence clusters (read-only reference)
│   ├── cluster-{A..F}-*.md
│   ├── raw/                            # raw web-search JSON
│   └── experiment-1-gh-releases-cors-RESULT.md
├── packages/
│   └── contracts/                      # shared TS types (MediaResult, etc.) — client + remote
├── apps/
│   ├── web/                            # the Pages SPA (Vite + React)
│   │   ├── src/
│   │   │   ├── main.tsx                # entry, capability detection, SW registration
│   │   │   ├── App.tsx                 # root, view-mode state
│   │   │   ├── components/             # UI components (URLInput, VariantPicker, DownloadProgress, etc.)
│   │   │   ├── adapters/               # PlatformAdapter implementations (tiktok, instagram, facebook, direct)
│   │   │   ├── workers/                # DedicatedWorker entry (download.ts), media worker
│   │   │   ├── media/                  # planMediaProcessing, Mediabunny/mp4box wrappers, escape-hatch loader
│   │   │   ├── storage/                # OPFS, IndexedDB, Cache API wrappers
│   │   │   ├── remote/                 # /extract, /proxy, /health client
│   │   │   ├── state/                  # Zustand stores
│   │   │   └── caps/                   # capability detection
│   │   ├── public/                     # static assets, favicon, manifest.webmanifest, .nojekyll, 404.html
│   │   ├── index.html                  # with <meta> CSP
│   │   └── vite.config.ts
│   └── api/                            # the remote service (Node.js)
│       ├── src/
│       │   ├── server.ts               # HTTP entry (fastify or native http)
│       │   ├── routes/
│       │   │   ├── extract.ts          # POST /extract
│       │   │   ├── proxy.ts            # GET /proxy
│       │   │   └── health.ts           # GET /health
│       │   ├── extractors/             # per-platform scrapers (tiktok.ts, instagram.ts, facebook.ts) + ytdlp.ts
│       │   ├── security/               # SSRF, allowlist, signed-urls, rate-limit
│       │   ├── proxy/                  # residential-proxy egress, media proxy streaming
│       │   ├── cache/                  # extraction-result cache (in-memory or Redis)
│       │   └── config.ts               # adapter configs (allowed hosts, referer, egress type)
│       └── package.json
├── .github/
│   └── workflows/
│       ├── ci.yml                      # lint/typecheck/test/build/validate/security
│       ├── deploy-pages.yml            # deploy to Pages on main
│       ├── release.yml                 # source tarball on tag v*
│       └── maintenance.yml             # scheduled: dep updates + extractor health
└── package.json                        # workspace root (bun/npm workspaces)
```

**Directory responsibilities (explicit):**
- `packages/contracts/` — the ONLY shared code between client and remote (TS types). Prevents drift.
- `apps/web/src/adapters/` — PlatformAdapter implementations. One file per platform. The registry map lives in `adapters/index.ts`.
- `apps/web/src/workers/` — DedicatedWorker entries. Separate from `components/` (UI) and `media/` (logic).
- `apps/web/src/media/` — media processing logic (planning function, tool wrappers, escape-hatch loader). UI-agnostic.
- `apps/web/src/remote/` — typed client for the remote API. Only place that knows the remote origin.
- `apps/api/src/extractors/` — per-platform scrapers. One file per platform. `ytdlp.ts` is the breadth backend wrapper.
- `apps/api/src/security/` — all security controls (SSRF, allowlist, signed URLs, rate limit). Co-located for auditability.

---

## 16. Architectural Simplification Review (second pass)

After designing the above, I attacked it again per the Pass 4 protocol.

| Subsystem / abstraction / dependency / language / runtime | CAN THIS BE REMOVED? | Verdict |
|---|---|---|
| SharedWorker (cross-tab coordination) | YES — per-tab DedicatedWorker is simpler | REMOVE from v1 (already OPTIONAL; confirm OFF by default) |
| Service Worker | NO — needed for offline app shell | KEEP (app-shell only) |
| Zustand | could use React context | KEEP (Zustand is ~1KB, simpler than context for this state shape) |
| Mediabunny + mp4box.js (two media libs) | could use just Mediabunny | KEEP both — mp4box.js is the MP4 fallback; different strengths |
| FFmpeg.wasm escape hatch | could drop entirely | KEEP — rare real inputs exceed pure-TS coverage; dropping loses completeness |
| The `direct` adapter (zero-server) | could remove | KEEP — enables degraded mode when remote is down [AD-2] |
| `packages/contracts/` workspace | could duplicate types | KEEP — prevents client/remote type drift; one file |
| Extraction-result cache (server) | could remove | KEEP — viral-URL traffic would re-extract otherwise; cheap |
| Residential proxy | could remove (drop IG/FB) | KEEP if IG/FB are v1 targets; DROP IG/FB if proxy fails (Exp 6) |
| CSP `<meta>` | could remove | KEEP — XSS hardening, free |
| A second language (Rust/Python in browser) | NO — TS everywhere | CONFIRM REJECT [AD-9] |
| A second remote runtime (Bun/Deno/CF Workers) | NO — Node.js chosen | CONFIRM OPTIONAL [AD-15 of tech ledger]; CF Workers EXPERIMENTAL |
| `MediaPlan` type | could inline | KEEP — it's a discriminated union, not an engine; aids readability |
| `CapabilityState` object | could inline checks | KEEP as a computed object — avoids re-checking, documents capabilities in one place |
| `DownloadJob` type | could use looser state | KEEP — typed state machine prevents bugs |

**Net result of simplification:** confirmed SharedWorker OFF by default in v1 (was already OPTIONAL). Everything else survives with justification. The architecture is the smallest practical set.

---

## 17. NEW ARCHITECTURAL DECISION (introduced by Pass 4)

### NEW-AD-A: FFmpeg.wasm escape-hatch distribution via jsDelivr npm CDN (USE), not GitHub Releases

- **Why introduced:** Pass 3 AD-6/AD-20/AD-21 specified GitHub Releases as the host for FFmpeg.wasm. **Experiment 1 was executed this pass** (see `.research/experiment-1-gh-releases-cors-RESULT.md`) and empirically proved GitHub Releases sends **no `Access-Control-Allow-Origin` header** on either the `github.com` 302 or the `release-assets.githubusercontent.com` 200 response. A browser `fetch()` from `*.github.io` is therefore CORS-blocked. The CORS preflight (OPTIONS) also fails (404).
- **The new decision:** The browser fetches `@ffmpeg/core` from `https://cdn.jsdelivr.net/npm/@ffmpeg/core@<pinned-version>/dist/umd/ffmpeg-core.wasm`, which sends `access-control-allow-origin: *`, `content-type: application/wasm`, and `cache-control: immutable` (verified Exp 1 Probe 3). SRI hash verification + exact version pin for supply-chain safety.
- **Effect on AD-20 (Pages responsibilities):** unchanged — Pages still hosts only the app shell.
- **Effect on AD-21 (Actions responsibilities):** Releases workflow no longer uploads browser-fetched WASM; it uploads source tarballs + metadata only.
- **Effect on GitHub Releases role:** demoted from "large browser-fetched asset host" to "source tarballs + release metadata." Still USE, but for a smaller purpose.
- **Effect on jsDelivr:** promoted from OPTIONAL (Pass 3) to USE (Pass 4), as the FFmpeg.wasm distribution path.
- **Effect on unpkg:** remains OPTIONAL (alternative to jsDelivr).
- **Future custom FFmpeg.wasm builds (LGPL-only, etc.):** if we ever build a custom WASM, it must be published as an npm package (or hosted on a CORS-friendly static host). GitHub Releases alone won't work. This is a USE-LATER concern.
- **Confidence:** HIGH — empirically measured, not assumed.
- **What would change this:** If GitHub begins sending CORS headers on release-asset responses (unlikely), revert to direct Releases fetch. Re-test periodically.

This is the **only** new architectural decision Pass 4 introduces. Every other architectural element traces directly to a Pass 3 `AD-x`. No silent decisions.

---

## 18. Architecture Decision Traceability (full map)

| Pass 4 architecture element | Pass 3 decision | Pass 2 evidence | Pass 1 evidence |
|---|---|---|---|
| Browser-executed, remote-assisted framing | AD-1 | §4.1 (failed "browser-first") | cluster D (CORS/Referer) |
| Zero-server degraded mode | AD-2 | §27 | cluster D |
| Capability-based extraction selection | AD-3 | §5.2 | cluster D |
| 3-endpoint remote API | AD-4 | §28 | cluster D §3 |
| Proxy when required | AD-5 | §28 | cluster D §7.3 |
| Pure-TS media + escape hatch | AD-6 | §12 | cluster C §8 |
| Scrapers + yt-dlp | AD-7 | §17 | cluster D §1 |
| PlatformAdapter interface | AD-8 | §29 | — |
| Direct functions (no engines) | AD-9 | §6.2, §30-33 | — |
| Capability object | AD-10 | §30 | cluster B |
| Worker architecture | AD-11 | §20 | cluster B, E |
| 3-store storage | AD-12 | §19 | cluster E §3 |
| Caching | AD-13 | §36 | cluster A §1.7 |
| SW app-shell only | AD-14 | §5.1, §35 | cluster B, E |
| Large-file + resume state machine | AD-15 | §7.1 | cluster E §2 |
| MediaResult type | AD-16 | §34 | — |
| planMediaProcessing function | AD-17 | §33 | — |
| No plugin architecture (v1) | AD-18 | §29 | — |
| Security defense-in-depth | AD-19 | §22 | cluster F |
| Pages app-shell only | AD-20 | §9 | cluster A |
| Actions CI/CD only | AD-21 | §21 | cluster A §2 |
| Desktop/mobile boundary | AD-22 | §26 | cluster E |
| No extension (v1) | AD-23 | §10 | cluster D |
| Local/remote boundary | AD-24 | §10, §27, §28 | clusters B, C, D, E |
| **FFmpeg.wasm via jsDelivr (NEW)** | **NEW-AD-A** | Exp 1 result (this pass) | cluster A §3 (Releases limits) |

---

## 19. Remaining Unknowns (carried from Pass 3 §18, updated by Exp 1)

| Unknown | Status after Pass 4 |
|---|---|
| GitHub Releases CORS | **RESOLVED (Exp 1 run)** — fails; jsDelivr is the path (NEW-AD-A) |
| Mediabunny robustness on real platform media | IMPLEMENTATION-TIME (Exp 2; architecture has escape-hatch fallback) |
| Cloudflare Workers viability | DEFERRED (Exp 3; Node.js is the default; optimization experiment) |
| TikTok CDN Range support | IMPLEMENTATION-TIME (Exp 4; AD-15 state machine handles both outcomes) |
| iOS Safari WASM memory cap | IMPLEMENTATION-TIME (Exp 5; needs iOS device; default 384MB cap until measured) |
| Residential-proxy IG extraction | NON-BLOCKING for architecture; scope-condition for IG in v1 (Exp 6; needs paid proxy) |
| Signed-URL TTL | IMPLEMENTATION-TIME (Exp 7; AD-15 handles arbitrary TTL) |
| iOS navigator.share large-file | IMPLEMENTATION-TIME (Exp 8; needs iOS device; fallback is `<a download>`) |

---

## 20. Conditions That Would Change the Architecture (beyond Pass 3 §21)

- **Exp 2 (Mediabunny) < 80% success:** escape hatch fires often → consider FFmpeg.wasm-by-default (accept GPL/cost). Architecture boundary unchanged; performance/cost shifts.
- **Exp 5 (iOS memory) shows FFmpeg.wasm unusable even at 256MB:** disable escape hatch on iOS → pure-TS only on iOS. Architecture unchanged; iOS feature set shrinks.
- **Exp 6 (residential proxy) < 70%:** drop IG from v1 platform list. Architecture unchanged; scope shrinks.
- **jsDelivr becomes unreliable:** switch to unpkg (already verified Exp 1 Probe 4) or self-host on a CORS-friendly static host.
- **Need for multi-threaded WASM emerges (hot path):** migrate hosting from Pages to Cloudflare Pages/Netlify (allow COOP/COEP) → unlock SAB → revisit Rust + multi-threaded FFmpeg.wasm. This is a significant migration, gated on a proven need.
- **A demonstrated third-party plugin use case:** adopt Extism-based plugin system (Pass 3 USE-LATER → USE). Architecture gains a plugin layer.

---

## END OF PASS 4 ARCHITECTURE

This document is the authoritative architecture specification. It is concrete enough that another engineer can implement the system without rediscovering the architectural decisions. Pass 5 (not started) will consume this + the system blueprint + experiment status to produce the implementation roadmap.
