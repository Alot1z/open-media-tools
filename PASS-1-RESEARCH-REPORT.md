# PASS 1 — RESEARCH / DISCOVERY REPORT

**Project:** open-media-tools — open-source browser-native media downloader / tool platform
**Repository:** `Alot1z/open-media-tools`
**Pass:** 1 (Research / Discovery)
**Date:** 2026 (current)
**Status:** COMPLETE
**Evidence base:** `.research/cluster-{A..F}-*.md` (6 cluster reports, ~1,600 lines, ~140KB, each claim cited inline with source URLs). Web searches conducted via the `web-search` skill (`z-ai function -n web_search`).

---

## 1. Executive Summary

This pass establishes the factual technical foundation for an open-source, browser-native media downloader that is hosted on GitHub Pages, executes as much as possible in the browser, and uses a minimal remote service only where browser execution is impossible. The six research clusters (GitHub infra, browser APIs, media+WASM, extraction tools, mobile+storage, security+licensing) converge on a small number of load-bearing findings:

1. **GitHub Pages is viable as the production host, but with hard constraints.** It serves a static site over HTTPS/HTTP2 with a 1GB published-site cap, 100GB/mo soft bandwidth, and a 10-minute `Cache-Control: max-age=600`. Critically, **Pages does not allow custom HTTP headers**, so `Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy` cannot be set — which means **`SharedArrayBuffer` (and therefore multi-threaded WASM) is not available** on a Pages-hosted app. A `.nojekyll` file is required to serve `.wasm` with the correct MIME type. SPA routing requires the `404.html` JS-redirect trick. Large binaries (FFmpeg.wasm ~25MB, multi-thread build ~120MB) **must be distributed via GitHub Releases** (2GB per asset, uncapped bandwidth), not bundled into the Pages site.

2. **Pure browser-native extraction is not viable for the initial target platforms.** TikTok, Instagram, and Facebook all (a) send no CORS headers permitting third-party HTML fetch, (b) use signed/short-lived/Referer-gated CDN URLs, and (c) require cookies or anti-bot bypass. The browser cannot set the `Referer` header on `fetch()` (it is a forbidden header), so even with a direct media URL, fetching the bytes from a cross-origin CDN fails when the CDN enforces Referer. **A remote extractor service is unavoidable** for these platforms. Meta's oEmbed endpoints went tokenless in June 2026 but return only embed HTML/thumbnails, not direct media URLs.

3. **FFmpeg.wasm is not necessary for v1.** The newly mature **Mediabunny** (pure-TS, July 2026) plus **mp4box.js** (BSD-3, GPAC) cover demux/mux/remux for MP4/MOV/WebM/MKV/HLS/WAVE/MP3/Ogg/ADTS/FLAC/MPEG-TS in pure TypeScript with zero WASM dependency. **WebCodecs** (Chrome 94+, Firefox 130+, Safari 16.4+ for video; Safari 26+ for audio) handles frame decode/encode. FFmpeg.wasm (~25MB, GPL with x264, ~20-25× slower than native, 2GB file ceiling, requires SharedArrayBuffer for threads which Pages can't provide) becomes an **escape hatch only**, loaded on demand from GitHub Releases for exotic codecs or operations the pure-TS path can't handle.

4. **WASM is not necessary for v1.** Pure TS + WebCodecs + Mediabunny covers the core download→remux→save flow. WASM (Rust or otherwise) is justified only later, where profiling proves a hot path needs it. This avoids the COOP/COEP problem entirely for v1.

5. **Safari/iOS is the hardest target and constrains the whole design.** iOS Safari evicts all origin storage (SW, IndexedDB, OPFS) after 7 days of no use; suspends JS when a tab is backgrounded (no Background Fetch API); has no File System Access pickers; has unreliable `<a download>` for media; and enforces tight per-tab memory limits that make large-WASM risky. The architecture must stream fetch→OPFS (never buffer whole files), hand off downloads to the OS Share Sheet immediately on iOS, and treat local storage as ephemeral.

6. **The Component Model / WASI 0.2/0.3 is production-ready on the server but is a build-time abstraction in the browser.** `jco transpile` converts components to JS+core WASM for browser use. It is **not a blocker** for v1 and adds complexity that isn't justified yet. Defer.

7. **Licensing is manageable.** Project license: MIT. Dependencies: mp4box.js (BSD-3), muxers (ISC), Mediabunny (MIT) — all permissive. yt-dlp (Unlicense) is referenced, not shipped. Cobalt (AGPL API, CC-BY-NC-SA frontend) is a reference architecture, not copied code — if we fork its API for our remote service, that service component becomes AGPL (acceptable for an open-source project). FFmpeg.wasm (GPL with x264, patent-encumbered for H.264 encode until ~2027-2030) is the one real friction point — mitigate by making it an optional, user-loaded plugin distributed via GitHub Releases, not a default bundle.

**Bottom line:** A browser-first downloader hosted on GitHub Pages is realistic, with a small remote extractor service for the platforms that require it, pure-TS media processing for v1, and FFmpeg.wasm as an optional escape hatch. The architecture must be designed around the Pages header limitation (no SAB/multithreaded WASM), iOS storage eviction, and the Referer/CORS wall on target platform CDNs.

---

## 2. Project Requirements

(From the project brief, restated for traceability.)

| # | Requirement | Source |
|---|---|---|
| R1 | Public GitHub repository | brief |
| R2 | GitHub is canonical source of truth | brief |
| R3 | Production website hosted on GitHub Pages | brief |
| R4 | Real browser application | brief |
| R5 | Browser/local execution wherever technically possible | brief |
| R6 | Minimal remote infrastructure | brief |
| R7 | GitHub Actions for CI/CD/release/maintenance/deployment | brief |
| R8 | GitHub Actions must NOT be the runtime for normal user downloads | brief |
| R9 | Extensible media/platform architecture | brief |
| R10 | Maintainable when platforms change | brief |
| R11 | Desktop + mobile | brief |
| R12 | Strong Safari/iOS support | brief |
| R13 | Chromium support | brief |
| R14 | Firefox support | brief |
| R15 | Lazy loading | brief |
| R16 | Local media processing wherever technically possible | brief |
| R17 | Remote fallback where browser execution cannot realistically work | brief |
| P1 | Initial platforms: TikTok, Instagram, Facebook | brief |
| P2 | Architecture must NOT be hard-coded around those three | brief |

---

## 3. Research Methodology

1. **Repository reconnaissance** (`/home/z/open-media-tools`): the project repo was bootstrapped fresh for this planning effort (commit `1e16823`). No prior planning artifacts existed. The reports are produced fresh.
2. **Six parallel research clusters**, each investigated via the `web-search` skill (`z-ai function -n web_search -a '{...}'`), with results saved to `.research/cluster-{A..F}-*.md`. Each cluster ran 8-16 targeted queries against official docs (MDN, web.dev, GitHub docs, ffmpeg.org, WebKit blog), canonical project repos (yt-dlp, cobalt, ffmpegwasm, GPAC, Mediabunny, wasmtime, bytecode-alliance), and current community discussions (2025-2026 dates).
3. **Source attribution**: every factual claim in the cluster reports is followed by `[source: URL]`. Items with conflicting or weak evidence are marked `[UNCERTAIN]`.
4. **Synthesis**: this report condenses the cluster findings into the 26 required sections and answers the 16 research questions. It does NOT make architecture decisions (that is Pass 3) — it establishes the evidence base.
5. **No fabrication**: where current 2026 evidence was not retrievable, the claim is marked UNCERTAIN rather than guessed.

---

## 4. GitHub Pages Findings

(Detail in `.research/cluster-A-github-infra.md` §1-7; `.research/cluster-F-security-licensing.md` §1.9)

### 4.1 Hard limits
| Limit | Value | Source |
|---|---|---|
| Published site size (hard) | 1 GB | docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits |
| Per-month bandwidth (soft) | 100 GB | same |
| Builds per hour (soft) | 10 | same |
| Build timeout | 10 minutes | same |
| Individual file in repo | 100 MiB hard block | docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github |

### 4.2 What Pages CAN do
- Serve static files over HTTPS (Let's Encrypt on custom domains) with HTTP/2, Fastly-backed edge.
- Serve `.wasm` with correct `application/wasm` MIME type **if `.nojekyll` is present** (otherwise Jekyll mangles it).
- Serve ES modules (`type="module"`) with correct `text/javascript` MIME.
- Custom domains via `CNAME` file; HTTPS enforced.
- SPA routing via the `404.html` JS-redirect trick (no native rewrite).
- 750+ MIME types derived from `mime-db`.

### 4.3 What Pages CANNOT do
- **Set custom HTTP response headers** — no `Cache-Control` override, no `Content-Security-Policy` via headers, no `Strict-Transport-Security`, no `Cross-Origin-Opener-Policy`, no `Cross-Origin-Embedder-Policy`, no `Cross-Origin-Resource-Policy`. [source: community/discussions/13309]
- This means **no `SharedArrayBuffer`** (requires cross-origin isolation via COOP+COEP), so **no multi-threaded WASM** on a Pages-hosted app. [source: wasmer docs; blog.tomayac.com]
- Workarounds exist but are unreliable: the `coi-serviceworker` shim registers a SW that re-injects COOP/COEP — flaky on iOS, breaks some embeds. Not recommended for production.
- A CSP **can** be set via `<meta http-equiv="Content-Security-Policy" content="...">` in the HTML (document-level CSP). This is viable for XSS hardening.
- No `_headers` / `_redirects` (Netlify-style) support.
- Default `Cache-Control: max-age=600` (10 min) on assets — all JS/CSS/WASM must use content-hashed URLs (Vite default) to defeat the short TTL.

### 4.4 Implications for this project
- The app shell (HTML/JS/CSS, small WASM) lives on Pages. ✓
- Large binaries (FFmpeg.wasm ~25MB single-thread / ~120MB multi-thread) MUST ship via **GitHub Releases**, fetched at runtime by the client. Multi-thread build is unusable on Pages anyway (needs SAB).
- All assets content-hashed (Vite default) to bypass 10-min cache TTL.
- `.nojekyll` file required at publish root.
- SPA routing via `404.html` redirect.

---

## 5. Browser Capability Findings

(Detail in `.research/cluster-B-browser-apis.md`)

### 5.1 Web Workers / Shared Workers / Service Workers
- **DedicatedWorker**: universally supported. Primary unit for off-main-thread work (media parsing, extraction logic, WASM).
- **SharedWorker**: supported on all targets including iOS Safari 16+ (re-added Apr 2022 after being dropped in Safari 6.1). [source: MDN SharedWorker; itnext.io/safari-now-fully-supports-sharedworkers]
- **ServiceWorker**: universally supported, but **iOS Safari evicts SW + all origin storage after 7 days of no use**. [source: webkit.org/blog/14403/updates-to-storage-policy]

### 5.2 Cross-origin isolation (COOP/COEP/CORP)
- `SharedArrayBuffer` requires `crossOriginIsolated` true, which requires `Cross-Origin-Opener-Policy: same-origin` AND `Cross-Origin-Embedder-Policy: require-corp` HTTP headers on the document.
- **GitHub Pages cannot set these headers** (see §4.3) → SAB unavailable → multi-threaded WASM unavailable on Pages-hosted app.
- This is the single most important platform constraint for media processing in-browser.

### 5.3 Streams API
- `ReadableStream`, `WritableStream`, `TransformStream`, `pipeTo`, `pipeThrough`, byte streams, async iteration — universally supported (Chrome, Firefox, Safari 15.2+).
- Canonical large-file pattern: `fetch(url).body.pipeTo(opfsWritableStream)`.

### 5.4 Fetch / CORS
- `fetch()` streaming supported everywhere.
- `mode: 'no-cors'` returns an **opaque** response — cannot read the body. Useless for extraction (can only consume into a media element or cache).
- Cross-origin media fetch requires the target to send `Access-Control-Allow-Origin`. TikTok/IG/FB do not.
- **`Referer` and `Origin` are forbidden headers** — JS cannot set them on `fetch()`. Platforms whose CDNs require a specific Referer are unfetchable from the browser. [source: MDN forbidden headers]

### 5.5 OPFS (Origin Private File System)
- Supported: Chrome 102+, Firefox 111+, Safari 15.2+.
- `createSyncAccessHandle()` — worker-only, synchronous, fast. Best for streaming media writes.
- `createWritable()` — main thread, async, slower.
- Subject to iOS 7-day eviction.
- NOT visible to user's file system — must export via `<a download>` or `navigator.share`.

### 5.6 IndexedDB
- Universally supported. Quota: Chrome ~60% disk, Firefox ~50%, Safari higher than the old 1GB cap but evictable on iOS.
- Good for metadata/history/queue (small records). For large blobs, prefer OPFS.

### 5.7 Cache API
- Stores `Response` objects, works in SW and window. Good for app-shell caching and short-lived extractor-result cache. Quota shared with IDB/OPFS.

### 5.8 File System Access API
- `showSaveFilePicker` / `showOpenFilePicker` / `showDirectoryPicker` — **Chrome/Edge only**. Firefox refuses. Safari ships only OPFS.
- Fallback: `<a download>` (desktop) or `navigator.share({files})` (iOS).

### 5.9 Browser downloads
- `<a download>` cross-origin is ignored (becomes navigation) — must fetch into a Blob first, then `URL.createObjectURL` + `<a download>`.
- iOS Safari `<a download>` for media types often opens inline instead of downloading — use `navigator.share({files: [File]})`.
- No native resumable download in Safari; Chrome's download manager resumes.

### 5.10 MediaSource Extensions (MSE)
- For streaming playback, not file output. iPhone supports only `ManagedMediaSource` (Safari 17.1+). Limited relevance to a downloader.

### 5.11 Web Audio
- `AudioContext.decodeAudioData` decodes audio for analysis/playback. Not for muxing. Useful for audio-only extraction preview.

---

## 6. Safari/iOS Findings

(Detail in `.research/cluster-E-mobile-storage.md` §1; `.research/cluster-B-browser-apis.md`)

### 6.1 The 7-day storage eviction (critical)
- iOS Safari evicts **all script-writable storage** (SW, IndexedDB, Cache API, OPFS) for an origin after 7 days of no Safari use. [source: webkit.org/blog/14403/updates-to-storage-policy]
- `navigator.storage.persist()` is requested but **not reliably honored** on iOS — the 7-day cap still applies.
- Implication: A local media library stored in the browser is **not viable on iOS**. Downloads must be handed off to the OS (Files app / camera roll via Share Sheet) immediately, not stored as a "library."

### 6.2 No background download
- iOS Safari suspends JS when a tab is backgrounded; the SW can be killed within seconds.
- **No Background Sync, no Periodic Background Sync, no Background Fetch API** on iOS. [source: MDN Background Fetch API; web.dev/articles/background-fetch-ai]
- A large download stalls when the user switches tabs or locks the phone. Workaround: stream to OPFS with resumable Range requests so a re-foregrounded tab can resume.

### 6.3 Memory limits
- Mobile Safari tab memory is severely constrained — crashes occur well below the ~2GB theoretical limit, often at a few hundred MB of active JS heap. [source: lapcatsoftware.com/articles/2026/1/7; emscripten issue #19374; babylonjs forum]
- FFmpeg.wasm (allocates 256MB-2GB linear memory) is **high-risk on iOS**. Must cap WASM memory and stream files (never buffer a full media file in JS heap).

### 6.4 WebCodecs on iOS
- `VideoDecoder` + `VideoEncoder`: iOS 16.4+ (Safari 16.4, Mar 2023). [source: MDN VideoDecoder]
- `AudioDecoder` + `AudioEncoder`: iOS 26+ (Safari 26, Sep 15, 2025). [source: testmuai.com/webcodecs-browser-support]
- Implication: video decode path is usable on iOS 16.4+; audio decode via WebCodecs requires iOS 26+ (fallback: Web Audio `decodeAudioData`).

### 6.5 SharedWorker on iOS
- Supported since Safari 16 (Apr 2022), including iOS. [source: MDN; itnext.io]

### 6.6 File System Access pickers
- Not supported on Safari/Firefox. Only OPFS (origin-private) is available. [cluster B]

### 6.7 `<a download>` on iOS
- Inconsistent — often opens media inline. Use `navigator.share({files})` for reliable save-to-Files/camera-roll. [cluster E §1.7]

### 6.8 iPadOS
- Reports desktop-class UA; API support largely matches macOS Safari (WebCodecs, OPFS). Desktop-class mode does not unlock extra APIs.

---

## 7. Browser Extraction Findings

(Detail in `.research/cluster-D-extraction.md`)

### 7.1 What a normal website (no extension) can do
- Fetch same-origin resources freely.
- Fetch cross-origin resources **only if** the target sends `Access-Control-Allow-Origin` matching the origin (or `*`).
- Use `mode: 'no-cors'` — but the response is **opaque** (cannot read body, only consume into `<video>`/`<img>`/cache). Useless for extraction (we need to read the HTML/JSON).
- Load cross-origin media via `<video src=...>` / `<img src=...>` (the browser loads it for playback/display) — but **cannot read the bytes** from JS.
- `Referer` and `Origin` are forbidden headers — cannot be set on `fetch()`.

### 7.2 Per-platform feasibility (initial targets)

| Platform | HTML fetch (CORS) | oEmbed | Embedded JSON scrape | Direct CDN URL fetch | Referer required | Cookies required |
|---|---|---|---|---|---|---|
| TikTok | ❌ blocked | ✅ public (embed HTML only) | ✅ `__UNIVERSAL_DATA_FOR_REHYDRATION__` (but page is cross-origin) | ❌ signed + Referer-gated | ✅ tiktok.com | ⚠️ sometimes |
| Instagram | ❌ blocked | ✅ tokenless since Jun 2026 (embed only) | ❌ removed `_sharedData`; signed GraphQL + datacenter-IP block | ❌ scontent CDN requires cookies | ✅ instagram.com | ✅ |
| Facebook | ❌ blocked | ✅ tokenless since Jun 2026 (embed only) | ❌ signed GraphQL | ❌ signed + expiring | ✅ facebook.com | ✅ |

- **oEmbed** (Meta tokenless since Jun 15, 2026; TikTok always public) returns **embed HTML/thumbnail only**, NOT direct media URLs. Does not solve extraction. [source: cluster D]
- **TikTok CDN URLs** are signed and short-lived (minutes to hours) and require a `Referer: https://www.tiktok.com/` header. Browser cannot set Referer → direct fetch fails.
- **Instagram** removed `window._sharedData` years ago; now uses signed GraphQL with datacenter-IP blocking (residential egress needed for remote). [source: cluster D]
- **Facebook** CDN URLs are signed and expiring.

### 7.3 Viable extraction architectures
1. **Browser extension** (manifest v3) with `host_permissions` for the target platforms + `declarativeNetRequest` to spoof Referer headers. Bypasses CORS and Referer restrictions. Most viable for TikTok. But: extension is a separate distribution channel (Chrome Web Store, Firefox Add-ons, Safari App Extension), not the Pages site.
2. **Remote extractor service**: `POST /extract {url} → {media[], metadata, expiresAt, proxyRecommended}`. Wraps yt-dlp for breadth, or thin per-platform JS scrapers (like Cobalt does). Residential-proxy egress for IG/FB.
3. **Hybrid**: the Pages app calls the remote extractor; the remote returns direct URLs where usable, or a signed proxy URL where the CDN requires Referer/cookies.

### 7.4 Conclusion
- **Pure browser-native extraction is NOT viable** for TikTok/Instagram/Facebook as a no-extension web app.
- A **remote extractor service is unavoidable** for these platforms.
- The remote service may return direct URLs (rare) or must proxy media bytes (common, when CDN requires Referer/cookies).

---

## 8. Media Processing Findings

(Detail in `.research/cluster-C-media-wasm.md` Part 1)

### 8.1 FFmpeg.wasm
- Current: `@ffmpeg/ffmpeg` 0.12.x, `@ffmpeg/core` 0.12.x.
- Single-threaded build: ~25MB WASM. Multi-threaded build: ~120MB, requires SharedArrayBuffer (→ COOP/COEP → unavailable on GitHub Pages).
- Performance: ~20-25× slower than native FFmpeg.
- File ceiling: ~2GB practical (linear memory limit).
- License: GPL (ships with libx264). Patent risk: H.264/H.265 encode until ~2027-2030.
- Supports `-c copy` remux (no re-encode).
- Startup: 1-3s to instantiate.
- **Verdict**: escape hatch only, not default. Load on demand from GitHub Releases for exotic codecs/operations the pure-TS path can't handle.

### 8.2 Pure-TS media tooling (the v1 path)
- **Mediabunny** (Vanilagy, July 2026): pure-TS, zero-dep, read/write MP4/MOV/WebM/MKV/HLS/WAVE/MP3/Ogg/ADTS/FLAC/MPEG-TS. Supersedes `mp4-muxer` and `webm-muxer`. Major enabler — covers most remux/mux/demux needs with no WASM.
- **mp4box.js 1.0.0** (June 2025, GPAC, BSD-3): gold standard for ISOBMFF (MP4) inspection/fragmentation. TypeScript support.
- **mp4-muxer / webm-muxer** (ISC): focused muxers, usable with WebCodecs.
- **WebCodecs**: frame decode/encode. Pair with a muxer for remux-with-reencode.

### 8.3 WebCodecs
- `VideoDecoder`/`VideoEncoder`: Chrome 94+, Firefox 130+, Safari 16.4+ (iOS 16.4+).
- `AudioDecoder`/`AudioEncoder`: Chrome 94+, Firefox 130+, Safari 26+ (iOS 26+).
- Codec coverage (2026): ~85% AV1 encode / 90% decode; H.264 universal; H.265 Safari-only; VP9 Chrome/Firefox; AV1 Chrome/Safari 17+/Firefox 129+.
- WebCodecs decodes/encodes **frames** — it does NOT demux/mux containers. Always pair with a muxer (Mediabunny, mp4box.js, mp4-muxer).

### 8.4 MediaSource / Web Audio
- MSE: streaming playback only, not file output. iPhone needs `ManagedMediaSource` (Safari 17.1+). Limited relevance.
- Web Audio `decodeAudioData`: audio decode for analysis/preview, not muxing.

### 8.5 Browser-native codecs
- `<video>`/`<audio>` playback: H.264/AAC universal; VP9/Opus Chrome/Firefox; HEVC Safari; AV1 Chrome/Safari 17+/Firefox 129+.
- Relevant for: "can we just save the source stream as-is?" — yes, if the source container/codec is browser-playable, remux is unnecessary; just stream bytes to OPFS and save.

---

## 9. WASM Findings

(Detail in `.research/cluster-C-media-wasm.md` Part 2)

### 9.1 Languages
- **Rust + wasm-bindgen**: best for new WASM, mature ecosystem, small binaries. Note: `wasm-pack` was **archived July 2025** — use `cargo` directly or `cargo-component`.
- **Zig**: small binaries, thin ecosystem. Viable but niche.
- **AssemblyScript**: TS-like, easier, limited ecosystem.
- **C/C++ (emscripten)**: only path for existing C libs (FFmpeg, GPAC). Heavy toolchain.
- **Go → wasm**: 10-25MB binaries, GC overhead. Poor fit for our use.

### 9.2 Component Model / WASI
- **WASI 0.3 ratified June 11, 2026** with native async (Wasmtime 43+, jco). Production-ready on **server**.
- In **browser**, the Component Model is a **build-time abstraction** — `jco transpile` converts components to JS+core WASM. No browser-native Component Model runtime.
- **Verdict**: not a blocker for v1, adds unjustified complexity. Defer. If we later need cross-language WASM plugins, revisit.

### 9.3 JS-in-WASM runtimes (for sandboxed extractors)
- **Javy** (Bytecode Alliance/Shopify): compiles JS to WASM, 1-16KB modules. Right tool for sandboxed third-party extractor scripts.
- **QuickJS + emscripten**: alternative JS-in-WASM. Larger, mature.
- **Pyodide** (CPython in WASM): 6.4MB + 4-5s init. Could run yt-dlp's pure-Python deps (mutagen) but needs FFmpeg.wasm sidecar. Heavier than a Javy JS port. Not viable for browser; possible for remote service.

### 9.4 Extism
- Universal WASM plugin system with a working browser JS SDK. Plausible foundation for a plugin architecture (if we adopt one). Not required for v1.

### 9.5 WebContainers
- Node.js runtime in browser (StackBlitz). Designed for dev environments. Overkill for a downloader. Requires COOP/COEP (Pages can't provide).

### 9.6 Is WASM actually necessary?
**No for v1.** Pure TS + WebCodecs + Mediabunny covers the core flow. WASM (Rust or otherwise) is justified only later where profiling proves a hot path needs it. Avoiding WASM in v1 sidesteps the COOP/COEP/multithreading problem entirely.

---

## 10. Language/Runtime Findings

### 10.1 Frontend
- **TypeScript**: required (strict typing, ecosystem). Universal for browser tooling.
- **React**: mature component model, huge ecosystem. Viable. Alternatives: Svelte/Solid (smaller bundles, fewer contributors). React is the safe default for an open-source project seeking contributors.
- **Vite**: fast dev server, ES-module-native, content-hashed output (defeats Pages 10-min cache). Ideal for a Pages-hosted SPA.
- **Next.js**: full-stack framework with SSR/SSG. **Overkill** for a Pages-hosted static app — we don't need server rendering, API routes, or Next's server features. Next's static export is possible but heavier than Vite. Vite is the better fit.

### 10.2 Runtimes (for the remote service + CI)
- **Node.js**: universal, mature, huge ecosystem. yt-dlp wrappers (`youtube-dl-exec`) are Node. Safe default.
- **Bun**: fast, Node-compatible, native TypeScript. Viable for the remote service. Smaller ecosystem for edge cases.
- **Deno**: native TypeScript, secure-by-default permissions. Viable but smaller ecosystem.
- **Cloudflare Workers**: edge runtime, V8-isolate, no Node APIs, 128MB memory, sub-ms cold start. Excellent for a stateless extractor proxy if we can fit the logic (no yt-dlp binary — would need pure-JS extractors). Free tier generous.
- **Serverless (Lambda/Functions)**: cold-start penalty, but scales to zero. Viable for low-traffic.
- **Container/VPS**: full control, can run yt-dlp binary. Higher ops burden.

### 10.3 Verdict (preliminary, Pass 3 decides)
- Remote service: Node.js or Bun (for yt-dlp wrapper compatibility) OR Cloudflare Workers (for edge + free tier, pure-JS extractors only).
- CI: GitHub Actions (runs on Node by default).

---

## 11. yt-dlp Findings

(Detail in `.research/cluster-D-extraction.md` §1)

- **Current version**: latest release Dec 7, 2025 (active maintenance). [source: cluster D]
- **License**: Unlicense (public domain). We can reference/wrap freely.
- **Extractors**: 1,000+ sites.
- **Python dependency**: requires Python 3.9+ (3.9 just dropped from support). Not browser-runnable.
- **JS wrappers**: `youtube-dl-exec`, `yt-dlp-exec` (npm) — wrap the binary, spawn as child process. Not pure-JS ports. **No pure-JS port of yt-dlp exists.**
- **Maintenance burden**: per-site Python extractor classes; sites change frequently; yt-dlp releases often to keep up.
- **Legal**: RIAA DMCA takedown (Oct 2020) was reversed (EFF, Nov 2020). Repo remains up. Precedent: general-purpose download tool is not inherently a DMCA §1201 circumvention device. [source: eff.org/deeplinks/2020/11/github-reinstates-youtube-dl-after-riaas-abuse-dmca]
- **Role in our architecture**: the remote extractor service may invoke yt-dlp as a backend (for breadth of platform support), with thin per-platform JS scrapers (Cobalt-style) for the initial targets. yt-dlp is NOT shipped to the browser.

---

## 12. Cobalt Findings

(Detail in `.research/cluster-D-extraction.md` §2; `.research/cluster-F-security-licensing.md` §2.3)

- **Project**: `imputnet/cobalt`. Reference architecture for a media downloader API.
- **License**: API = **AGPL-3.0**; web frontend = **CC-BY-NC-SA-4.0** (NonCommercial). [source: github.com/imputnet/cobalt/blob/main/api/LICENSE; /web/README.md]
- **API status**: public v7 API was **shut down Nov 11, 2024**. Self-host via Docker + `API_AUTH_REQUIRED=1` + `API_KEY_URL`. v11.2 added local processing (Jun 2025).
- **Extraction mechanism**: own JS scrapers (does NOT shell out to yt-dlp — inferred, unconfirmed). Returns either direct CDN URL (redirect) or proxied stream.
- **Response modes**: direct redirect (302 to CDN URL) OR proxied stream (service fetches and re-streams, adding CORS headers).
- **Implications**:
  - AGPL on the API means if we fork Cobalt's API for our remote service, that service is AGPL. Acceptable for an open-source project (we're public).
  - CC-BY-NC-SA on the frontend means we **cannot copy Cobalt's UI**. We write our own.
  - Cobalt is the closest existing reference architecture. Worth studying but not copying wholesale.

---

## 13. Remote Architecture Findings

### 13.1 What is the smallest viable remote service?
A stateless HTTP service exposing:
- `POST /extract` — body `{url}`; response `{media: [{url, type, quality, container, codec, expiresAt}], metadata: {...}, proxyRecommended: bool}`.
- `GET /proxy?u=<url>&sig=<hmac>` — proxies media bytes for a specific resolved URL when the browser can't fetch directly (Referer/cookies required). Adds CORS headers.

No cookie storage, no user accounts, no database (extraction results cached in memory or short-TTL Redis for 30-60s).

### 13.2 When must it proxy media bytes?
- CDN requires a `Referer` header the browser can't set (TikTok, IG, FB — common).
- CDN URL is signed and the browser fetch fails CORS.
- Cookies/session required (IG/FB private content).
- Rate-limited per-IP and the browser's shared IP would be blocked.

### 13.3 Can it just return direct URLs?
- For platforms with permissive CDNs (rare) — yes.
- For TikTok/IG/FB — frequently no. The proxy path is the common case for these.

### 13.4 SSRF / open-proxy mitigations
(Detail in `.research/cluster-F-security-licensing.md` §1.1-1.2)
- Per-platform host allowlist.
- Private-IP rejection (10/8, 172.16/12, 192.168/16, 127/8, 169.254/16 incl. cloud metadata, ::1, fc00::/7).
- Cloud-metadata block (169.254.169.254, metadata.google.internal).
- Redirect re-validation or disable redirects.
- DNS-rebinding defense (pin resolved IP).
- Signed proxy URLs (HMAC, ~60s TTL).
- Per-IP rate limit, file-size cap, timeout.
- No cookie/credential storage by default.

---

## 14. Storage Findings

(Detail in `.research/cluster-E-mobile-storage.md` Part 3)

### 14.1 IndexedDB
- Universally supported. Quota: Chrome ~60% disk, Firefox ~50%, Safari higher than old 1GB cap but evictable on iOS (7-day).
- Good for: metadata, history, queue (small records).
- Less good for: large media blobs (overhead, structured clone).

### 14.2 OPFS
- Chrome 102+, Firefox 111+, Safari 15.2+.
- `createSyncAccessHandle()` (worker-only) — fast synchronous writes. **Primary large-file staging area.**
- Subject to iOS 7-day eviction.
- Not user-visible — must export via `<a download>` / `navigator.share`.

### 14.3 Cache API
- Stores `Response` objects. Good for: app-shell caching (SW offline), short-lived extractor-result cache.
- Quota shared with IDB/OPFS via Storage API.

### 14.4 Service Worker cache
- Subset of Cache API, used by the SW for offline app shell + (optionally) media caching.
- iOS 7-day eviction applies.

### 14.5 SQLite WASM
- **wa-sqlite** (rhashimoto): SQLite → WASM with pluggable VFS. `OPFSCoopSyncVFS` is the recommended persistent VFS (2025), works on all browsers, excellent performance. [source: powersync.com/blog/sqlite-persistence-on-the-web]
- **sql.js**: in-memory only, load/save whole DB manually. Simpler, not for large persistent DBs.
- **sqlite.org official wasm** ships an OPFS VFS since 3.40+ (2023).
- Adds ~1-2MB WASM. Use case: history with full-text search, complex queue state. **For v1, IndexedDB may suffice** — SQLite WASM is a USE-LATER candidate.

---

## 15. CI/CD Findings

(Detail in `.research/cluster-A-github-infra.md` §2-4)

### 15.1 GitHub Actions
- Job timeout: 6h. Concurrent jobs: 20 (Free) / 40 (Pro). Minutes: 2,000 (Free) / 3,000 (Pro) — **unlimited & free for public repos** (key for OSS).
- Cache: 10GB/repo (LRU). Artifact retention: 90 days default (configurable 1-400 days).
- Scheduled cron: 5-min minimum, **best-effort and unreliable** (skips, multi-hour delays). Maintenance workflows must be idempotent and self-healing.
- OIDC supported; environments with up to 6 required reviewers; reusable workflows (4-level nesting); Dependabot + OIDC since Feb 2026.
- Security risks: `pull_request_target` (pwn-request — mitigated as of Dec 8, 2025 but still dangerous with `run:` interpolation), expression injection via issue/PR body.
- **Must NOT be the runtime for normal user downloads** (R8) — Actions are for build/test/release/maintenance/deploy only.

### 15.2 GitHub Releases
- 2 GiB per asset (hard), no total-size or bandwidth cap. Auto SHA-256 digests since Jun 2025.
- Served from `objects.githubusercontent.com` / `release-assets.githubusercontent.com` (Fastly, 302 redirect).
- Direct download URLs NOT rate-limited; REST API 60/hr/IP unauth, 5,000/hr auth.
- Use for distributing: FFmpeg.wasm builds, large WASM modules, optional plugins. NOT for the app shell (Pages).

### 15.3 git-LFS
- 10GB storage + 10GB bandwidth/month free (raised from 1/1 in Jul 2023). $5/50GB data packs.
- **Broken on public forks** — don't use for contributor-modifiable assets.
- Does NOT count against Pages.

### 15.4 Deployment flow
1. Push to `main` → Actions workflow builds the Vite app → uploads `dist/` as artifact → deploys to Pages (via `actions/deploy-pages`).
2. Tag a release (`v1.2.3`) → Actions workflow builds large assets (FFmpeg.wasm) → uploads to GitHub Release.
3. Scheduled workflow (daily/weekly) → checks for upstream dependency updates (yt-dlp, FFmpeg.wasm, Mediabunny) → opens PR or auto-updates.

---

## 16. Security Findings

(Detail in `.research/cluster-F-security-licensing.md` Part 1)

### 16.1 SSRF (remote extractor)
- Allowlist + private-IP rejection + cloud-metadata block + redirect re-validation + DNS-rebinding defense. See §13.4.

### 16.2 Open proxy abuse
- Signed proxy URLs (HMAC, short TTL), per-IP rate limit, host allowlist, file-size cap. The `/proxy` endpoint is opt-in, not default.

### 16.3 Malicious media
- Run parsers in DedicatedWorker with timeouts + file-size caps. Prefer pure-TS parsers (no memory-corruption risk) over C/WASM where possible. For WASM FFmpeg: pinned build + SRI.

### 16.4 Resource exhaustion
- Worker timeouts, WASM fuel metering, input size caps, parse-depth limits.

### 16.5 WASM security
- Sandboxed linear memory. Supply-chain risk mitigated by SRI on `.wasm` files + pinned builds (commit SHA). [source: MDN SRI; andrewlock.net]

### 16.6 Workers
- Same-origin, structured-clone postMessage, no exfiltration beyond origin. Trusted Types for DOM.

### 16.7 Dependency supply chain
- Lockfile committed, `npm audit` / socket.dev in CI, pinned exact versions, Dependabot, minimize dep count.

### 16.8 GitHub Actions security
- `permissions: contents: read` default; per-job escalation. Pin third-party actions to SHA. OIDC for cloud deploys. Avoid `pull_request_target` with `run:` interpolation. [source: securitylab.github.com; gitguardian.com; wiz.io]

### 16.9 CSP on Pages
- `<meta http-equiv="Content-Security-Policy" content="...">` works for document CSP.
- COOP/COEP/HSTS cannot be set (no header control). SAB unavailable.

### 16.10 CORS for remote service
- `Access-Control-Allow-Origin: https://<pages-domain>` (or `*` for public unauth). Handle OPTIONS preflight.

### 16.11 Secrets
- Zero secrets on Pages site (client is public). Remote service secrets in its env only.

### 16.12 Content legality
- youtube-dl/RIAA precedent (reversed). General-purpose download tool not inherently illegal. Mitigations: no search/discovery, no hosting, user-initiated only, "personal use" framing. [source: eff.org; copyrightlately.com]

---

## 17. Licensing Findings

(Detail in `.research/cluster-F-security-licensing.md` Part 2)

### 17.1 Dependency license matrix
| Dependency | License | MIT-compatible? | Notes |
|---|---|---|---|
| FFmpeg.wasm (with x264) | GPL | ⚠️ viral if linked | H.264 encode patent until ~2027-2030 |
| FFmpeg.wasm (LGPL-only build) | LGPL-2.1 | ✅ dynamic link | No x264; lower codec support |
| yt-dlp | Unlicense | ✅ | Referenced, not shipped |
| Cobalt API | AGPL-3.0 | ✅ if svc is AGPL | Network copyleft on modified service |
| Cobalt web | CC-BY-NC-SA-4.0 | ❌ NonCommercial | Do not copy frontend |
| gallery-dl | GPL-2.0 | ⚠️ if shipped | Python, not browser-relevant |
| mp4box.js | BSD-3 | ✅ | Permissive |
| mp4-muxer / webm-muxer | ISC | ✅ | Permissive |
| Mediabunny | MIT | ✅ | Permissive |
| Rust + wasm-bindgen | MIT/Apache | ✅ | Permissive |
| wasmtime / jco / Extism | Apache/MIT/BSD | ✅ | Permissive |

### 17.2 Codec patent landscape
| Codec | Patent status | Royalty-free? |
|---|---|---|
| H.264/AVC | Mostly expired; some until ~2027-2030 | No (until expiry) |
| H.265/HEVC | Complex multiple pools | No |
| VP9 | Royalty-free (Google) | Yes |
| AV1 | Royalty-free (AOMedia) | Yes |
| AAC | Patent-encumbered (Via Licensing) | No |
| Opus | Royalty-free (IETF) | Yes |
| MP3 | Patents expired (2017+) | Yes |

### 17.3 Recommended project license
**MIT** (or Apache-2.0). Permissive, maximizes adoption. Compatible with all permissive deps. Caveats: FFmpeg.wasm (GPL) must be an optional, user-loaded plugin (not bundled) to avoid GPL obligations on the core app; if we fork Cobalt's API, that service component is AGPL (acceptable for open source).

---

## 18. Performance Findings

### 18.1 Initial bundle
- App shell (HTML/JS/CSS): target <300KB gzipped (React + Vite + app code). Lazy-load everything else.
- No WASM in initial bundle. Mediabunny (pure TS, ~50-100KB) lazy-loaded when media processing is needed.
- WebCodecs is a browser API — zero bundle cost.

### 18.2 Lazy bundles
- Per-platform extractor UI (route-level code splitting).
- Media processing module (Mediabunny + mp4box.js) — loaded on first download.
- FFmpeg.wasm — loaded only if the pure-TS path can't handle the input (escape hatch), from GitHub Releases.

### 18.3 WASM size
- FFmpeg.wasm single-thread: ~25MB. Multi-thread: ~120MB (unusable on Pages anyway).
- wa-sqlite: ~1-2MB (if adopted).
- v1 target: zero WASM in default flow.

### 18.4 Worker startup
- DedicatedWorker spawn: <10ms. SharedWorker: <50ms (first load).
- WASM instantiation (FFmpeg): 1-3s — only when escape hatch is triggered.

### 18.5 Media processing
- Remux via Mediabunny: near-realtime for typical files (TS, no decode).
- WebCodecs decode: realtime or faster (hardware-accelerated).
- FFmpeg.wasm: 20-25× slower than native — acceptable for occasional use, not for batch.

### 18.6 Memory
- Stream fetch→OPFS; never hold a full media file in JS heap.
- iOS tab memory: a few hundred MB JS heap before crash risk. Cap WASM linear memory at ~256MB on iOS.

### 18.7 Mobile
- iOS: foreground-only downloads, 7-day storage eviction, Share Sheet for saves.
- Android Chrome: Background Fetch API available (progressive enhancement).

---

## 19. Large-File Findings

(Detail in `.research/cluster-E-mobile-storage.md` §2)

### 19.1 Strategy
1. `fetch(url)` → stream `response.body` (ReadableStream).
2. DedicatedWorker opens OPFS `createSyncAccessHandle()` for target file.
3. `response.body.pipeTo(opfsWritable)` or chunk-by-chunk `postMessage` → `handle.write(chunk)`.
4. Track bytes written for resume (store offset in IndexedDB).
5. On completion: read OPFS file → Blob → `<a download>` (desktop) or `navigator.share({files})` (iOS).
6. Resume on failure: re-`fetch` with `Range: bytes=N-` (if CDN supports).

### 19.2 Limits
- No whole-file buffering in JS heap (memory).
- iOS: tab must stay foregrounded; download stalls on background.
- Chrome: Background Fetch API for survivability (progressive enhancement).
- CDN must support Range for resume (most do; signed URLs may not).

---

## 20. Mobile Findings

(Detail in `.research/cluster-E-mobile-storage.md`)

### 20.1 iOS Safari
- 7-day storage eviction (no workaround).
- No background download (no web API).
- No File System Access pickers.
- Unreliable `<a download>` for media — use `navigator.share({files})`.
- WebCodecs: video iOS 16.4+, audio iOS 26+.
- SharedWorker: iOS 16+.
- Memory: tight (few hundred MB JS heap).
- `navigator.storage.persist()`: requested, not guaranteed.

### 20.2 Android Chrome
- Background Fetch API (Chrome 74+): downloads survive tab close.
- OPFS, WebCodecs, File System Access pickers all supported.
- Quota: ~60% disk.

### 20.3 PWA
- Installable (manifest + SW). iOS PWA storage eviction still applies.
- Standalone mode: no browser chrome, but same API limits.

---

## 21. Emerging Technologies

### 21.1 Mediabunny (July 2026)
- Pure-TS media container read/write. Supersedes mp4-muxer/webm-muxer. **Major enabler** — removes WASM dependency for v1 media processing. [source: cluster C]

### 21.2 WASI 0.3 (June 11, 2026)
- Ratified with native async. Production-ready on server. Browser use via `jco transpile`. Not a v1 blocker. [source: cluster C]

### 21.3 WebCodecs AudioDecoder/Encoder on iOS (Safari 26, Sep 15, 2025)
- Completes the WebCodecs story on iOS. Older iOS (16.4-25) has video only. [source: cluster E]

### 21.4 Meta tokenless oEmbed (June 15, 2026)
- Instagram/Facebook/Threads oEmbed no longer require tokens. But returns embed HTML/thumbnail only, not direct media URLs. Does not solve extraction. [source: cluster D]

### 21.5 GitHub Actions `pull_request_target` hardening (Dec 8, 2025)
- Now always uses default-branch workflow file. Mitigates stale-workflow exploitation. Still dangerous with `run:` interpolation. [source: cluster F]

### 21.6 coi-serviceworker (COOP/COEP shim)
- Registers a SW that re-injects COOP/COEP headers on GitHub Pages to enable SAB. Flaky on iOS, breaks some embeds. **Not recommended for production.** [source: cluster F]

---

## 22. Candidate Technologies

(For Pass 3 to assign USE / USE-LATER / OPTIONAL / EXPERIMENTAL / REJECT.)

| Category | Candidate |
|---|---|
| Frontend | TypeScript, React, Vite |
| Browser APIs | DedicatedWorker, SharedWorker, ServiceWorker, Streams, Fetch, OPFS, IndexedDB, Cache API, WebCodecs (progressive), `navigator.share`, `navigator.storage.persist` |
| Media (TS) | Mediabunny, mp4box.js, mp4-muxer/webm-muxer |
| Media (WASM, escape hatch) | FFmpeg.wasm (single-thread, GitHub Releases) |
| Extraction | Remote extractor service (Node/Bun + yt-dlp OR Cloudflare Workers + pure-JS); browser-native oEmbed for metadata |
| Storage | OPFS (large files), IndexedDB (metadata), Cache API (app shell) |
| Remote | Node.js or Bun (yt-dlp wrapper) OR Cloudflare Workers (pure-JS extractors) |
| CI/CD | GitHub Actions, GitHub Pages (app shell), GitHub Releases (large assets) |
| Optional/deferred | SQLite WASM (wa-sqlite), Rust+WASM, Component Model/WIT/jco, Extism, Javy, Background Fetch API (Chrome-only enhancement) |

---

## 23. Technologies That Appear Unnecessary

| Technology | Why unnecessary |
|---|---|
| Next.js | Overkill for static Pages site; Vite is lighter and fits. |
| Multi-threaded FFmpeg.wasm | Requires SAB → COOP/COEP → Pages can't provide. |
| SharedArrayBuffer | Same — needs COOP/COEP, unavailable on Pages. |
| Pyodide | Huge, slow startup; yt-dlp's native deps don't all work. Use yt-dlp server-side instead. |
| WebContainers | Dev-environment tool, overkill for a downloader. Needs COOP/COEP. |
| Go → WASM | 10-25MB binaries, GC overhead, poor fit. |
| gallery-dl | Python, image-galleries focus, not browser-relevant. |
| Cobalt web frontend | CC-BY-NC-SA (NonCommercial). Write our own UI. |
| `coi-serviceworker` COOP/COEP shim | Flaky, breaks embeds. Not production-viable. |
| Component Model / WIT / jco (for v1) | Server-ready, browser is build-time abstraction. Adds complexity without v1 value. |
| Extism (for v1) | Plugin system premature without a proven plugin requirement. |
| SQLite WASM (for v1) | IndexedDB suffices for v1 metadata. Defer. |

---

## 24. Major Risks

1. **Pages header limitation** → no SAB → no multi-threaded WASM. Constrains all media-processing design. (CONFIRMED)
2. **iOS 7-day storage eviction** → no reliable local library on iOS. Downloads must hand off to OS immediately. (CONFIRMED)
3. **iOS no background download** → large downloads stall when tab backgrounded. (CONFIRMED)
4. **CORS + Referer wall on target CDNs** → remote extractor + proxy service unavoidable. (CONFIRMED)
5. **Platform anti-bot / datacenter-IP blocking** (IG/FB) → remote service needs residential-proxy egress or risks IP bans. (LIKELY)
6. **Platform ToS / legal exposure** → DMCA takedown risk (youtube-dl precedent reversed, but ongoing). (CONFIRMED risk)
7. **FFmpeg.wasm GPL + H.264 patent** → must be optional plugin, not bundled, to keep core MIT. (CONFIRMED)
8. **CDN signed-URL expiry** → extracted URLs expire in minutes-hours; download must start promptly, and resume may need re-extraction. (CONFIRMED)
9. **Scheduled Actions unreliability** → maintenance workflows must be idempotent + self-healing + triggered on push. (CONFIRMED)
10. **Release-asset CORS** (post-2025 domain change to `release-assets.githubusercontent.com`) → needs empirical browser `fetch()` test before committing to runtime binary fetching. (UNCERTAIN)
11. **WASM supply chain** → compromised FFmpeg.wasm build. Mitigate with SRI + pinned builds. (CONFIRMED risk)
12. **Remote service abuse** (open proxy / SSRF) → strict allowlist + signed URLs + rate limits. (CONFIRMED risk)

---

## 25. Unknowns

(Marked for Pass 2 red-team investigation.)

1. Whether Cobalt's API uses yt-dlp internally or pure-JS scrapers (inferred pure-JS, unconfirmed).
2. Exact TikTok CDN URL TTL ("minutes to hours" — no precise citation).
3. Whether a browser extension's `declarativeNetRequest` Referer spoofing alone suffices for TikTok CDN, or whether cookies are also required.
4. Whether GitHub Releases' `release-assets.githubusercontent.com` serves CORS headers allowing browser `fetch()` from a `*.github.io` origin (post-2025 domain change). **Needs empirical test.**
5. Exact iOS Safari tab memory limit (anecdotal "few hundred MB"; no published hard number).
6. Whether `navigator.storage.persist()` ever survives the iOS 7-day cap in practice.
7. Whether an LGPL-only FFmpeg.wasm build (without x264) is practically usable for our remux needs.
8. Whether AGPL on a Cobalt-derived remote service "infects" the client app (legal consensus: no, but untested in court for this exact pattern).
9. Whether the `coi-serviceworker` shim is reliable enough to ship (anecdotal: flaky on iOS).
10. Sunfishcobalt / inv.tux.pizza — not surfaced by search; need direct checks if relevant.
11. Whether Cloudflare Workers can run our extractor logic within the 128MB memory / CPU limits (yt-dlp can't run there; pure-JS scrapers likely can).
12. Whether Mediabunny handles all container variants we'll encounter (MKV, fragmented MP4, HLS) robustly — needs empirical test.

---

## 26. Questions Requiring Red-Team Investigation (→ Pass 2)

1. **Is "browser-first" actually achievable, or is it a misleading framing given that extraction is unavoidably remote for the initial targets?** (Challenge the project's core premise.)
2. **Is the remote service actually "minimal," or will it grow into a full Cobalt clone?** (Challenge the scope boundary.)
3. **Is GitHub Pages the right host, or does the COOP/COEP limitation force a migration to Cloudflare Pages/Netlify later?** (Challenge the hosting decision.)
4. **Is React justified, or would a smaller framework (Svelte/Solid/Qwik) meaningfully reduce bundle and improve mobile perf?** (Challenge the frontend choice.)
5. **Is FFmpeg.wasm ever actually needed if Mediabunny + WebCodecs cover the cases?** (Challenge the escape-hatch inclusion.)
6. **Is the plugin architecture solving a real requirement, or is it premature?** (Challenge the extensibility goal.)
7. **Does the Service Worker earn its complexity, or is it just for offline app shell?** (Challenge the SW inclusion.)
8. **Is IndexedDB sufficient, or do we need SQLite WASM for the history/queue?** (Challenge the storage choice.)
9. **Can the remote service be stateless, or do auth/cookies/rate-limits force state?** (Challenge the statelessness assumption.)
10. **Does the iOS 7-day eviction make a "local library" feature impossible on iOS, and should we just not offer it?** (Challenge the feature set.)
11. **Is the Component Model / WASI worth tracking, or is it noise?** (Challenge the deferral.)
12. **Are we underestimating the maintenance burden of per-platform extractors?** (Challenge the maintainability claim.)
13. **What happens when a platform changes its page structure?** (Failure mode analysis.)
14. **Is the signed-URL expiry window long enough for large-file download + resume?** (Failure mode.)
15. **Does the remote proxy path introduce legal liability (we're re-streaming copyrighted bytes)?** (Legal red-team.)
16. **Are we over-engineering with a "capability engine" / "runtime registry" / "media planner" before we have a working v1?** (Challenge premature abstractions.)

---

## END OF PASS 1 REPORT

**Next:** Pass 2 (Red-team / Feasibility / Counter-research) will challenge every conclusion above, test failure scenarios, and produce a provisional decision matrix. Pass 3 will then convert the surviving evidence into the formal decision ledger.
