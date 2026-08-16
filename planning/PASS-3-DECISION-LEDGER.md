# PASS 3 — FORMAL TECHNOLOGY + ARCHITECTURE DECISION LEDGER

**Project:** open-media-tools
**Repository:** `Alot1z/open-media-tools`
**Pass:** 3 (Formal Technology + Architecture Decisions)
**Date:** 2026 (current)
**Status:** COMPLETE

---

## 1. Executive Decision Summary

This ledger converts the evidence from Pass 1 (Research) and Pass 2 (Red-team) into a formal, traceable decision system. Every major technology and architectural decision receives exactly one state: **USE**, **USE-LATER**, **OPTIONAL**, **EXPERIMENTAL**, or **REJECT** — with evidence, reasoning, alternatives, and the conditions that would change it.

**The architecture that emerges from the evidence** is a **browser-executed, remote-assisted** media downloader:

- A **static SPA** (React + Vite + TypeScript) hosted on **GitHub Pages**, with large binary assets (FFmpeg.wasm escape hatch) distributed via **GitHub Releases**.
- The browser handles **all UI, orchestration, stream-to-storage, and local media processing** (remux/demux/mux via pure-TS Mediabunny + mp4box.js; frame decode/encode via WebCodecs as a progressive enhancement).
- A **minimal-surface remote service** (Node.js on a VPS/container) handles only what the browser *cannot*: **extraction** (CORS bypass + Referer/cookie injection for TikTok/IG/FB) and **media-byte proxying** (when CDNs require Referer the browser can't set). The service is stateless, allowlisted, SSRF-defended, rate-limited, and uses residential-proxy egress for IG/FB.
- **No WASM in the default v1 flow.** FFmpeg.wasm (GPL, ~25MB, single-threaded) is an optional escape hatch loaded on demand from Releases, triggered only when the pure-TS path can't handle an input.
- **No multi-threaded WASM** ever on GitHub Pages (no COOP/COEP headers → no SharedArrayBuffer). This is a hard, accepted platform constraint.
- **iOS shapes the design**: 7-day storage eviction (no local library — hand off to OS Share Sheet immediately), no background download (foreground-only with resume-on-revisit), tight memory (cap WASM, stream files, never buffer whole files in JS heap).
- **No premature abstractions**: a `PlatformAdapter` interface replaces a plugin engine; direct functions replace capability/runtime/extraction registries and a media planner. A normalized `MediaResult` data type is kept (it's a type, not an engine).
- **MIT license** for the core; FFmpeg.wasm (GPL) isolated as an optional plugin; Cobalt (AGPL) studied but not copied (frontend is CC-BY-NC-SA, NonCommercial).

**The minimum technology set** is 15 items (see §10). Everything else is deferred, optional, experimental, or rejected with explicit reasons.

**What would change major decisions**: a GitHub Releases CORS failure (→ jsDelivr mirror), Mediabunny unreliability on real platform media (→ more FFmpeg.wasm reliance), IG residential-proxy failure (→ drop IG from v1), or a migration to a host that allows COOP/COEP (→ unlock multi-threaded WASM, revisit Rust).

---

## 2. Inputs Used

| Input | Path | Commit | Status |
|---|---|---|---|
| Pass 1 Research Report | `PASS-1-RESEARCH-REPORT.md` | `4a79534` | COMPLETE (729 lines, 26 sections, 16 research questions answered) |
| Pass 2 Red-Team Report | `PASS-2-RED-TEAM-REPORT.md` | `2cc2061` | COMPLETE (921 lines, 41 sections, provisional decision matrix, 8-experiment plan) |
| Evidence cluster A (GitHub infra) | `.research/cluster-A-github-infra.md` | `4a79534` | 324 lines, cited |
| Evidence cluster B (Browser APIs) | `.research/cluster-B-browser-apis.md` | `4a79534` | 333 lines, cited |
| Evidence cluster C (Media + WASM) | `.research/cluster-C-media-wasm.md` | `4a79534` | 250 lines, cited |
| Evidence cluster D (Extraction tools) | `.research/cluster-D-extraction.md` | `4a79534` | 238 lines, cited |
| Evidence cluster E (Mobile + Storage) | `.research/cluster-E-mobile-storage.md` | `4a79534` | 207 lines, cited |
| Evidence cluster F (Security + Licensing) | `.research/cluster-F-security-licensing.md` | `4a79534` | 251 lines, cited |
| Repository state | `Alot1z/open-media-tools` | `main` @ `2cc2061` | Public, 3 commits, planning-only (no implementation) |

**External sources consulted** (via web-search skill): MDN, web.dev, GitHub docs, ffmpeg.org, WebKit blog, caniuse, GPAC/mp4box.js repo, Mediabunny docs, yt-dlp repo, Cobalt repo, Bytecode Alliance, EFF, Via Licensing, powersync.com, testmuai.com, and others — all cited inline in the cluster files.

**Discrepancies found**: None between Pass 1 and the repository (repo was bootstrapped fresh for this planning effort; no prior state to conflict). Pass 2 revised two Pass 1 framings ("browser-first" → "browser-executed, remote-assisted"; "minimal remote" → "remote service is the operational core") — both reflected in this ledger.

---

## 3. Complete Technology Decision Ledger

| Technology / Decision | Purpose | Pass 1 | Pass 2 | Pass 3 | Evidence | Reason | Alternatives | Confidence |
|---|---|---|---|---|---|---|---|---|
| **GitHub Pages** | Host the app shell (static SPA) | candidate | USE | **USE** | cluster A §1 | Right host: free, HTTPS, HTTP/2, Fastly edge; 1GB site / 100GB BW suffices for app shell | Cloudflare Pages, Netlify (allow COOP/COEP but add complexity) | HIGH |
| **GitHub Releases** | Distribute large assets (FFmpeg.wasm) | candidate | USE (CORS test) | **USE** | cluster A §3; cluster F | 2GB/asset, uncapped bandwidth, SHA-256 digests | jsDelivr CDN mirror (if CORS fails), git-LFS (broken on forks) | MEDIUM (CORS test pending) |
| **GitHub Actions** | CI/CD: build/test/release/deploy/maintenance | candidate | USE | **USE** | cluster A §2; cluster F §1.8 | Free for public repos; not runtime (R8) | None (project requirement) | HIGH |
| **.nojekyll** | Serve .wasm with correct MIME on Pages | candidate | USE | **USE** | cluster A §1.3 | Required for application/wasm MIME | Jekyll processing (breaks .wasm MIME) | HIGH |
| **Content-hashed asset URLs** | Defeat Pages 10-min cache TTL | identified | USE | **USE** | cluster A §1.7 | Vite default; ensures new deploys reach users | Manual cache-busting query strings | HIGH |
| **Vite** | Frontend build tool / dev server | candidate | USE | **USE** | cluster C | Fast, ES-module-native, content-hashed output | Next.js (overkill), webpack (slower), esbuild (lower-level) | HIGH |
| **React** | UI component framework | candidate | USE (challenged) | **USE** | cluster C | Contributor pool, ecosystem, acceptable bundle | Svelte/Solid (smaller, fewer contributors), Qwik (novel) | MEDIUM |
| **TypeScript** | Type-safe language (client + remote) | candidate | USE | **USE** | cluster C | Strict typing, universal across stack | JavaScript (no safety), Flow (dead) | HIGH |
| **Next.js** | Full-stack React framework | candidate | REJECT | **REJECT** | cluster C; Pass 2 §15 | Overkill for static Pages (no SSR/SSG/API routes needed); heavier than Vite | Vite (chosen) | HIGH |
| **DedicatedWorker** | Off-main-thread work (download, parse, WASM) | candidate | USE | **USE** | cluster B, E | Primary worker unit; universal support | Main thread (blocks UI) | HIGH |
| **SharedWorker** | Cross-tab coordination | candidate | OPTIONAL | **OPTIONAL** | cluster B, E; Pass 2 §20 | iOS 16+; useful for single download-manager across tabs | Per-tab DedicatedWorker (simpler, no sharing) | MEDIUM |
| **ServiceWorker** | App-shell caching (offline UI) | candidate | USE (reduced) | **USE** (app shell only) | cluster B, E; Pass 2 §5.1, §35 | Offline app shell; NOT media interception (no gain, CORS still blocks) | No SW (online-only) | HIGH |
| **Streams API** | Stream fetch → OPFS (avoid buffering) | candidate | USE | **USE** | cluster B | Universal support; canonical large-file pattern | Buffer whole file in memory (crashes iOS) | HIGH |
| **Fetch / CORS** | Core download mechanism | candidate | USE | **USE** | cluster B, D | Core API; no-cors is opaque (useless for extraction) | XMLHttpRequest (legacy) | HIGH |
| **OPFS** | Large-file staging (download → process → save) | candidate | USE | **USE** | cluster B, E | Sync access handle in worker; fast writes; all browsers | IndexedDB blobs (slower), File System Access (Chrome-only) | HIGH |
| **IndexedDB** | Metadata, history, queue, settings | candidate | USE | **USE** | cluster B, E | Universal; sufficient for v1 record types | SQLite WASM (deferred) | HIGH |
| **Cache API** | App-shell caching (with SW) | candidate | USE | **USE** | cluster B, E | Stores Response objects; works in SW | IndexedDB for responses (overkill) | HIGH |
| **WebCodecs (VideoDecoder/Encoder)** | Frame decode/encode (progressive) | candidate | USE | **USE** (progressive) | cluster B, C, E | Chrome 94+, FF 130+, Safari 16.4+ (iOS 16.4+) | FFmpeg.wasm (escape hatch) | HIGH |
| **WebCodecs (AudioDecoder/Encoder)** | Audio frame decode/encode | candidate | USE (caveat) | **USE** (progressive, iOS 26+) | cluster E §1.4 | Safari 26+ (Sep 2025); older iOS needs fallback | Web Audio decodeAudioData (decode only), FFmpeg.wasm | HIGH |
| **navigator.share({files})** | iOS save-to-Files / camera roll | candidate | USE | **USE** | cluster E §1.7 | Reliable iOS save; requires user gesture | `<a download>` (unreliable on iOS for media) | HIGH |
| **navigator.storage.persist()** | Request persistent storage | candidate | OPTIONAL | **OPTIONAL** | cluster E §3.5 | Requested, not guaranteed on iOS 7-day cap | Nothing (accept eviction) | MEDIUM |
| **Background Fetch API** | Survive tab close on Chrome | candidate | OPTIONAL | **OPTIONAL** (enhancement) | cluster E §2.3 | Chrome 74+ only; not on iOS/Firefox | Foreground-only (accepted on iOS) | HIGH |
| **File System Access API** | Desktop save-to-chosen-folder | candidate | OPTIONAL | **OPTIONAL** | cluster B | Chrome/Edge only; Safari/Firefox fall back to `<a download>` | `<a download>` (universal fallback) | MEDIUM |
| **Mediabunny** | Pure-TS demux/mux/remux | candidate | USE (test) | **USE** (needs Exp 2) | cluster C §8.2 | Pure-TS, zero-dep, covers MP4/MOV/WebM/MKV/HLS/etc.; supersedes mp4-muxer/webm-muxer | mp4box.js (MP4 only), FFmpeg.wasm (escape hatch) | MEDIUM (robustness test pending) |
| **mp4box.js** | ISOBMFF inspection/fragmentation | candidate | USE | **USE** | cluster C §8.2 | BSD-3, GPAC-maintained, gold standard for MP4 | Mediabunny (may suffice), FFmpeg.wasm | HIGH |
| **mp4-muxer / webm-muxer** | Focused muxers (with WebCodecs) | candidate | OPTIONAL | **OPTIONAL** | cluster C | If Mediabunny insufficient for a specific case | Mediabunny (preferred) | MEDIUM |
| **FFmpeg.wasm (single-thread)** | Escape hatch: exotic codec/operation | candidate | USE-LATER | **USE-LATER** (escape hatch) | cluster C §8.1; cluster F §2.1 | Load on demand from Releases; GPL isolated as optional plugin; ~25MB; 20-25× slower than native | Pure-TS path (default), native FFmpeg server-side (not browser) | MEDIUM (trigger conditions defined) |
| **Multi-threaded FFmpeg.wasm** | Faster escape hatch | candidate | REJECT | **REJECT** | cluster A, C, F; Pass 2 §12 | Needs SharedArrayBuffer → COOP/COEP → unavailable on Pages | Single-threaded FFmpeg.wasm | HIGH |
| **SharedArrayBuffer** | Multi-threaded WASM memory | candidate | REJECT | **REJECT** | cluster A, B, F | Needs COOP/COEP headers; Pages can't set them; coi-serviceworker shim is flaky | Single-threaded WASM, pure TS | HIGH |
| **coi-serviceworker (COOP/COEP shim)** | Enable SAB on Pages | identified | REJECT | **REJECT** | cluster F §1.9 | Flaky on iOS, breaks embeds; not production-viable | Migrate host (Cloudflare Pages) if SAB ever needed | HIGH |
| **WASM (general, v1)** | Custom high-perf modules | candidate | REJECT (v1) | **REJECT** (v1) | cluster C; Pass 2 §13 | Pure TS suffices for v1; avoids COOP/COEP problem entirely | Rust/Zig/AS → WASM (USE-LATER if needed) | HIGH |
| **Rust + wasm-bindgen** | Custom WASM (future) | candidate | USE-LATER | **USE-LATER** | cluster C §9.1 | Best for new WASM if a hot path needs it; wasm-pack archived Jul 2025 (use cargo) | Zig, AssemblyScript | HIGH |
| **Zig** | Custom WASM (future) | candidate | OPTIONAL | **OPTIONAL** | cluster C | Small binaries, thin ecosystem | Rust (preferred) | LOW |
| **AssemblyScript** | Custom WASM (future) | candidate | OPTIONAL | **OPTIONAL** | cluster C | TS-like, limited ecosystem | Rust (preferred) | LOW |
| **C/C++ (emscripten)** | Compile existing C libs (FFmpeg, GPAC) | identified | USE (via FFmpeg.wasm) | **USE** (only via FFmpeg.wasm) | cluster C | Only path for existing C libs; used for the escape hatch | Rust rewrite (enormous effort) | HIGH |
| **Go → WASM** | Custom WASM | candidate | REJECT | **REJECT** | cluster C | 10-25MB binaries, GC overhead, poor fit | Rust, TS | HIGH |
| **Component Model / WIT / jco** | Cross-language WASM composability | candidate | USE-LATER | **USE-LATER** | cluster C §9.2; Pass 2 §14 | Server-ready; browser is build-time abstraction; no v1 value | Direct WASM imports | HIGH |
| **WASI 0.2 / 0.3** | WASM system interface | candidate | USE-LATER | **USE-LATER** | cluster C §9.2 | Ratified Jun 2026; server use; browser via jco; defer | None | HIGH |
| **Javy** | JS-in-WASM sandbox | candidate | USE-LATER | **USE-LATER** | cluster C §9.3 | If sandboxed third-party extractors needed (plugin system) | QuickJS | MEDIUM |
| **QuickJS (emscripten)** | JS-in-WASM sandbox | candidate | USE-LATER | **USE-LATER** | cluster C §9.3 | Alternative JS-in-WASM | Javy | MEDIUM |
| **Extism** | Universal WASM plugin system | candidate | USE-LATER | **USE-LATER** | cluster C §9.4 | If a plugin system is justified | Custom adapter interface (v1) | MEDIUM |
| **Pyodide** | CPython in WASM (run yt-dlp in browser) | candidate | REJECT | **REJECT** | cluster C §9.3 | 6.4MB + 4-5s init; yt-dlp native deps don't work; too heavy | yt-dlp server-side (chosen) | HIGH |
| **WebContainers** | Node.js runtime in browser | candidate | REJECT | **REJECT** | cluster C §9.5 | Overkill; needs COOP/COEP; dev-environment tool | None | HIGH |
| **SQLite WASM (wa-sqlite)** | Structured queries on history | candidate | USE-LATER | **USE-LATER** | cluster E §3.4 | IndexedDB suffices for v1; revisit if FTS/complex queries needed | IndexedDB (v1) | HIGH |
| **sql.js** | In-memory SQLite | identified | REJECT | **REJECT** | cluster E §3.4 | In-memory only; not for persistent DBs | wa-sqlite (if needed later) | HIGH |
| **yt-dlp (remote, server-side)** | Breadth backend (1000+ sites) | candidate | USE | **USE** | cluster D §1; cluster F §2.2 | Unlicense; server-side; invoked by remote service for breadth | Per-platform JS scrapers (for initial targets) | HIGH |
| **Cobalt (reference architecture)** | Study extraction API shape | candidate | OPTIONAL | **OPTIONAL** (study, don't copy) | cluster D §2; cluster F §2.3 | AGPL API (if forked, service is AGPL); CC-BY-NC-SA frontend (don't copy) | Build our own (preferred) | HIGH |
| **gallery-dl** | Image gallery extraction | candidate | REJECT | **REJECT** | cluster D | Python, not browser-relevant, GPL | yt-dlp (broader) | HIGH |
| **Remote extractor service** | Extraction for TikTok/IG/FB | candidate | USE | **USE** | cluster D §3; Pass 2 §10, §28 | Unavoidable: CORS + Referer wall | Browser extension (separate distribution) | HIGH |
| **Remote media proxy** | Fetch Referer-gated media bytes | candidate | USE | **USE** | cluster D §3; Pass 2 §28 | When CDN requires Referer/cookies browser can't set | Direct browser fetch (rare permissive CDNs) | HIGH |
| **Node.js (remote service)** | Runtime for remote service | candidate | USE | **USE** | cluster C, D | yt-dlp wrapper compatibility; universal | Bun, Deno (optional) | HIGH |
| **Bun (remote service)** | Alternative runtime | candidate | OPTIONAL | **OPTIONAL** | cluster C | Faster, native TS; smaller ecosystem | Node.js (chosen) | MEDIUM |
| **Deno (remote service)** | Alternative runtime | candidate | OPTIONAL | **OPTIONAL** | cluster C | Secure-by-default; smaller ecosystem | Node.js (chosen) | MEDIUM |
| **Cloudflare Workers (remote)** | Edge runtime for extractors | candidate | EXPERIMENTAL | **EXPERIMENTAL** (Exp 3) | cluster C; Pass 2 §18 | Pure-JS only (no yt-dlp); 128MB; free tier; needs test | Node.js on VPS (chosen default) | MEDIUM |
| **Serverless (Lambda/Functions)** | Scales-to-zero remote | candidate | OPTIONAL | **OPTIONAL** | cluster C | Cold start; viable for low-traffic | VPS/container (chosen) | MEDIUM |
| **Container (Fly.io/Railway/Render)** | Remote service hosting | candidate | USE | **USE** | cluster C | Full control, easy deploy, yt-dlp | VPS (cheaper, more ops) | HIGH |
| **VPS (Hetzner/DigitalOcean)** | Cheapest remote hosting | candidate | USE | **USE** | cluster C | Cheap, full control | Container platform (more managed) | HIGH |
| **Residential proxy (BrightData/Oxylabs)** | IG/FB egress (datacenter blocked) | identified | USE (Exp 6) | **USE** (Exp 6) | cluster D; Pass 2 §7.2 | IG/FB block datacenter IPs | Drop IG from v1 (if proxy fails) | MEDIUM (cost + experiment) |
| **Plugin architecture** | Third-party extensibility | candidate | REJECT (v1) | **REJECT** (v1) | Pass 2 §6.1, §29 | Premature; no demonstrated use case; use PlatformAdapter interface | PlatformAdapter map (chosen) | HIGH |
| **Capability Engine** | Detect browser features | candidate | REJECT (v1) | **REJECT** (v1) | Pass 2 §6.2, §30 | Over-abstracted; use direct feature detection object | Direct `capabilities` object (chosen) | HIGH |
| **Runtime Registry** | Select browser/remote | candidate | REJECT (v1) | **REJECT** (v1) | Pass 2 §6.2, §31 | Over-abstracted; use a function | `selectExtractionStrategy()` (chosen) | HIGH |
| **Extraction Registry** | Register platform adapters | candidate | REJECT (v1) | **REJECT** (v1) | Pass 2 §6.2, §32 | Over-abstracted; use a map | Plain object map (chosen) | HIGH |
| **Media Planner** | Decide processing pipeline | candidate | REJECT (v1) | **REJECT** (v1) | Pass 2 §6.2, §33 | Over-abstracted; use a function | `planMediaProcessing()` (chosen) | HIGH |
| **Normalized Media Model (MediaResult)** | Decouple adapters from UI | candidate | USE | **USE** | Pass 2 §6.2, §34 | A data type (not an engine); genuinely useful | Per-adapter bespoke shapes (coupling) | HIGH |
| **MIT project license** | Project license | candidate | USE | **USE** | cluster F §2.8 | Permissive, maximizes adoption, compatible with deps | Apache-2.0 (equivalent), GPL (viral) | HIGH |
| **Extraction-result cache (server)** | Avoid re-extraction of viral URLs | identified | USE | **USE** | Pass 2 §36 | 30-60s TTL; in-memory or Redis | None (re-extract every time) | HIGH |
| **Health-check endpoint** | Monitoring + client degraded-mode | identified | USE | **USE** | Pass 2 §36 | `GET /health` for uptime + client check | None | HIGH |
| **jsDelivr CDN mirror** | CORS-friendly asset mirror | identified | OPTIONAL | **OPTIONAL** (if Exp 1 fails) | Pass 2 §36 | Mirror of GH Releases with CORS headers | GitHub Releases (chosen default) | MEDIUM |
| **Client-side error reporting** | Production debugging | identified | OPTIONAL | **OPTIONAL** | Pass 2 §36 | Sentry-style; privacy-conscious, opt-in | None | MEDIUM |
| **CSP via `<meta>`** | XSS hardening | identified | USE | **USE** | cluster F §1.9 | Document-level CSP; works on Pages | None (COOP/COEP impossible on Pages) | HIGH |

---

## 4. Architectural Decision Ledger

### AD-1: Browser-first execution → reframed as "browser-executed, remote-assisted"
- **Decision:** The browser executes all UI, orchestration, stream-to-storage, and local media processing. The remote service executes only what the browser *cannot* (extraction, Referer-gated fetch).
- **Why this:** Pass 2 §4.1 proved "browser-first" is misleading — extraction is unavoidably remote for TikTok/IG/FB. The honest framing matches reality.
- **Why not pure browser-first:** CORS + Referer wall (cluster D). No browser API works around it.
- **Why not pure remote:** Project requirement R5 (browser execution wherever possible); local processing (Mediabunny, WebCodecs) is genuinely browser-capable and avoids server load/cost.
- **Evidence:** cluster D (CORS/Referer); cluster C (Mediabunny/WebCodecs browser-capable).
- **What would change it:** A browser extension distribution (bypasses CORS) — but that's a separate channel, not the Pages site.
- **Confidence:** HIGH.

### AD-2: Zero-server capability boundary
- **Decision:** The app supports a **zero-server degraded mode**: if the remote service is down, the app still works for (a) direct-media-URL downloads (user pastes a CDN URL) and (b) permissive-CDN platforms. Extraction-dependent platforms (TikTok/IG/FB) degrade to "service unavailable."
- **Why this:** Pass 2 §40 identified the remote service as a SPOF. A degraded mode preserves partial utility during outages.
- **Why not full zero-server:** Impossible for the initial targets (cluster D).
- **Evidence:** Pass 2 §27, §40.
- **What would change it:** Nothing — this is a robustness feature, not a constraint.
- **Confidence:** HIGH.

### AD-3: Remote fallback → capability-based selection (not "try browser, fall back")
- **Decision:** Each `PlatformAdapter` declares upfront whether it needs remote extraction. There's no "try browser, fall back to remote" — the system uses the right strategy for the platform's known constraints.
- **Why this:** Pass 2 §5.2 showed "try-fallback" is wrong for platforms where the browser provably can't extract.
- **Why not try-fallback:** Wastes time on a doomed browser attempt; misleads the user.
- **Evidence:** Pass 2 §5.2; cluster D.
- **Confidence:** HIGH.

### AD-4: Minimum backend (remote service API surface)
- **Decision:** The remote service exposes exactly three endpoints:
  - `POST /extract {url}` → `{media[], metadata, expiresAt, proxyRecommended}`
  - `GET /proxy?u=<url>&sig=<hmac>` → proxies media bytes (Referer/cookies + CORS headers)
  - `GET /health` → `{status, version}`
- No transcoding, no storage, no user accounts, no database (extraction cache optional, in-memory/Redis 30-60s TTL).
- **Why this:** Pass 2 §28 defined the minimum; cluster D §3 confirmed.
- **Why not more:** Every added endpoint/feature increases attack surface and ops cost.
- **Evidence:** Pass 2 §28; cluster D.
- **Confidence:** HIGH.

### AD-5: Remote media proxying — when required
- **Decision:** The `/proxy` endpoint is invoked only when (a) `proxyRecommended: true` from `/extract`, or (b) the browser's direct fetch of the media URL fails (CORS/Referer/403). The proxy adds CORS headers and injects the required Referer/cookies.
- **Why this:** Pass 2 §28; cluster D §7.3.
- **Why not always proxy:** Wastes server bandwidth; direct browser fetch is faster and cheaper when the CDN permits it.
- **Why not never proxy:** TikTok/IG/FB CDNs require Referer the browser can't set.
- **Evidence:** cluster D; Pass 2 §28.
- **Confidence:** HIGH.

### AD-6: Media engine — pure-TS default, FFmpeg.wasm escape hatch
- **Decision:** v1 media processing uses **Mediabunny + mp4box.js** (pure TS) for demux/mux/remux, and **WebCodecs** (progressive) for frame decode/encode. **FFmpeg.wasm** (single-threaded, ~25MB, from GitHub Releases) is loaded on demand only when:
  - The input container is not supported by Mediabunny, OR
  - The codec is not decodable by WebCodecs, OR
  - The operation is not supported by the pure-TS path (e.g., burn-in subtitles, complex filters), OR
  - The user explicitly requests a format requiring re-encode to a codec pure-TS can't produce.
- **Why this:** cluster C §8; Pass 2 §12. Pure-TS avoids WASM size/init cost and the COOP/COEP problem.
- **Why not FFmpeg.wasm by default:** 25MB download, 1-3s init, 20-25× slower than native, GPL/patent friction, iOS memory risk.
- **Why not drop FFmpeg.wasm entirely:** Some real inputs will exceed Mediabunny/WebCodecs coverage; the escape hatch preserves completeness.
- **Evidence:** cluster C; Pass 2 §12; Exp 2 (Mediabunny robustness), Exp 5 (iOS memory).
- **What would change it:** If Mediabunny proves >95% reliable on real platform media (Exp 2), the escape hatch fires rarely; if <80%, it fires often and the design shifts toward FFmpeg.wasm-by-default.
- **Confidence:** MEDIUM (pending experiments).

### AD-7: Extraction engine — remote, per-platform scrapers + yt-dlp fallback
- **Decision:** The remote service uses **thin per-platform JS scrapers** for the initial targets (TikTok, Instagram, Facebook) — faster and more maintainable than yt-dlp's generic extractors for these specific sites. **yt-dlp** (server-side, via `youtube-dl-exec` or equivalent) is the breadth backend for other platforms the user may paste.
- **Why this:** cluster D §1; Pass 2 §17. yt-dlp is Python (not browser-runnable); per-platform scrapers give control and speed for the common cases.
- **Why not pure yt-dlp:** Slower per-extraction; generic extractors break more often for specific sites.
- **Why not pure per-platform scrapers:** Loses breadth (1000+ sites); maintenance burden of writing scrapers for every site.
- **Evidence:** cluster D; Pass 2 §17.
- **Confidence:** HIGH.

### AD-8: Platform adapter model — simple interface, no plugin engine
- **Decision:** A `PlatformAdapter` interface (TS) with a plain-object registry map. Adapters are static or lazy dynamic imports. No runtime plugin loading, no WASM plugins, no registry class.
- **Why this:** Pass 2 §6.1, §29, §32. A plugin engine is YAGNI without a demonstrated third-party use case.
- **Why not a full plugin system:** Premature complexity; the adapter interface suffices.
- **Evidence:** Pass 2 §29.
- **What would change it:** A demonstrated third-party-contributor use case that the adapter interface can't handle.
- **Confidence:** HIGH.

### AD-9: Runtime abstraction — none (direct functions)
- **Decision:** No runtime registry/engine. Extraction strategy selection is a function (`selectExtractionStrategy(adapter)`); capability detection is a computed object; media planning is a function (`planMediaProcessing(input)`).
- **Why this:** Pass 2 §6.2, §30-33. Abstractions should prevent real duplication; these don't.
- **Why not engines/registries:** Architecture-astronaut overhead; functions are clearer.
- **Evidence:** Pass 2 §30-33.
- **Confidence:** HIGH.

### AD-10: Capability detection — direct feature detection
- **Decision:** A single `capabilities` object computed at startup:
  ```ts
  const capabilities = {
    webCodecsVideo: typeof VideoDecoder !== 'undefined',
    webCodecsAudio: typeof AudioDecoder !== 'undefined',
    opfsSyncHandle: ..., backgroundFetch: ..., shareFiles: ..., ...
  };
  ```
- **Why this:** Pass 2 §30. One-liner per feature; no engine.
- **Evidence:** Pass 2 §30.
- **Confidence:** HIGH.

### AD-11: Worker architecture — DedicatedWorker primary, SharedWorker optional, SW app-shell only
- **Decision:**
  - **DedicatedWorker**: primary unit for download streaming, media parsing, WASM. One per tab.
  - **SharedWorker**: OPTIONAL — if multi-tab coordination matters (single download queue across tabs). iOS 16+.
  - **ServiceWorker**: app-shell caching ONLY. No media interception, no background sync.
- **Why this:** Pass 2 §5.1, §20, §35.
- **Why not SW for media:** CORS still blocks; proxy handles it; SW is unreliable on iOS.
- **Evidence:** cluster B, E; Pass 2 §35.
- **Confidence:** HIGH.

### AD-12: Storage architecture — three stores, clear roles
- **Decision:**
  - **OPFS** (sync access handle in worker): large-file staging (download → process → save). Ephemeral on iOS (7-day eviction).
  - **IndexedDB**: metadata, history, queue, settings (small records).
  - **Cache API** (via SW): app-shell caching for offline UI.
  - **Memory**: transient stream chunks (implicit).
- No SQLite WASM in v1.
- **Why this:** Pass 2 §19. Each store has a distinct role; no overlap.
- **Why not SQLite WASM:** IndexedDB suffices for v1 record types; SQLite adds ~1-2MB WASM unjustifiably.
- **Why not IndexedDB for large blobs:** OPFS sync handles are faster and lower-overhead.
- **Evidence:** cluster E §3; Pass 2 §19.
- **Confidence:** HIGH.

### AD-13: Caching — extraction-result cache (server) + asset cache (client)
- **Decision:**
  - **Server**: extraction-result cache (in-memory or Redis, 30-60s TTL) keyed by URL — avoids re-extraction of viral URLs.
  - **Client**: app-shell cache (Cache API via SW) + content-hashed asset URLs (Vite default) defeating Pages 10-min TTL.
- **Why this:** Pass 2 §36; cluster A §1.7.
- **Evidence:** Pass 2 §36.
- **Confidence:** HIGH.

### AD-14: Service Worker — app-shell caching only
- **Decision:** SW registers, caches the app shell (HTML/JS/CSS) for offline UI. Does NOT intercept media fetches, does NOT do background sync, does NOT do periodic background sync. Update notification on new SW version.
- **Why this:** Pass 2 §5.1, §35. SW for media is no gain (CORS blocks); iOS evicts SW in 7 days.
- **Evidence:** cluster B, E; Pass 2 §35.
- **Confidence:** HIGH.

### AD-15: Large-file strategy — stream to OPFS, resume on expiry/failure
- **Decision:**
  1. `fetch(url)` → stream `response.body`.
  2. DedicatedWorker opens OPFS sync access handle.
  3. `pipeTo` or chunked `postMessage` → `handle.write(chunk)`.
  4. Track bytes written in IndexedDB (for resume).
  5. On completion: read OPFS → Blob → `<a download>` (desktop) or `navigator.share({files})` (iOS).
  6. On failure/expiry: re-extract (if URL expired), resume with `Range: bytes=N-` (if CDN supports, Exp 4) or restart from zero. Cap re-extraction attempts at 3.
- **Why this:** cluster E §2; Pass 2 §7.1.
- **Why not buffer whole file:** iOS memory crash.
- **Why not native download manager:** Unreliable on iOS; no resume on URL expiry.
- **Evidence:** cluster E; Pass 2 §7.1; Exp 4, Exp 7.
- **Confidence:** HIGH (resume-on-expiry state machine needs implementation care).

### AD-16: Normalized media model — MediaResult data type
- **Decision:** A `MediaResult` interface normalizes extraction output across adapters:
  ```ts
  interface MediaResult {
    platform: string; originalUrl: string;
    media: Array<{url, type, container?, codec?, quality?, width?, height?, durationMs?, expiresAt?, proxyRequired?}>;
    metadata: {title?, author?, thumbnailUrl?, durationMs?};
    extractedAt: number;
  }
  ```
- **Why this:** Pass 2 §34. A data type (not an engine); genuinely decouples adapters from UI.
- **Evidence:** Pass 2 §34.
- **Confidence:** HIGH.

### AD-17: Media planning — direct function
- **Decision:** `planMediaProcessing(input: MediaInput): ProcessingPlan` — a function (switch on container/codec/target), not a planner class.
- **Why this:** Pass 2 §33.
- **Evidence:** Pass 2 §33.
- **Confidence:** HIGH.

### AD-18: Plugin architecture — REJECTED for v1
- **Decision:** No plugin system in v1. Use `PlatformAdapter` interface + plain-object map. Revisit only when a demonstrated third-party-contributor use case appears AND the adapter interface proves insufficient.
- **Why this:** Pass 2 §6.1, §29.
- **Evidence:** Pass 2 §29.
- **What would change it:** A real plugin requirement (e.g., user-supplied extractors for niche platforms) that the adapter interface can't handle.
- **Confidence:** HIGH.

### AD-19: Security boundaries — defense-in-depth on the remote service
- **Decision:** (see §15 for detail) Per-platform host allowlist, private-IP rejection, cloud-metadata block, redirect re-validation, DNS-rebinding defense, signed proxy URLs (HMAC, 60s TTL), per-IP rate limit, file-size cap, timeout, no cookie storage by default, worker isolation for media parsing, SRI on WASM, pinned builds, CSP via `<meta>`, zero secrets on client.
- **Why this:** cluster F Part 1; Pass 2 §22.
- **Evidence:** cluster F; Pass 2 §22.
- **Confidence:** HIGH.

### AD-20: GitHub Pages responsibilities — app shell only
- **Decision:** Pages hosts the app shell (HTML/JS/CSS, small). NOT large assets (Releases), NOT the remote service (VPS/container), NOT user data (browser storage).
- **Why this:** cluster A; Pass 2 §9.
- **Evidence:** cluster A.
- **Confidence:** HIGH.

### AD-21: GitHub Actions responsibilities — CI/CD only, not runtime
- **Decision:** Actions does: build (Vite), test (lint/unit), deploy to Pages, build+upload large assets to Releases, scheduled maintenance (dependency updates, extractor-health checks). Actions does NOT: serve user downloads, run extraction, proxy media.
- **Why this:** Project requirement R8; cluster A §2.
- **Evidence:** cluster A; Pass 2 §21.
- **Confidence:** HIGH.

### AD-22: Desktop/mobile boundary
- **Decision:** Same SPA, responsive. Desktop: File System Access API for save-to-folder (optional), Background Fetch API (Chrome). Mobile (iOS): navigator.share for saves, foreground-only downloads, ephemeral storage. Mobile (Android): Background Fetch API, OPFS, WebCodecs.
- **Why this:** cluster E.
- **Evidence:** cluster E.
- **Confidence:** HIGH.

### AD-23: Browser/extension boundary
- **Decision:** v1 ships as a Pages web app (no extension). A browser extension (manifest v3, host_permissions, declarativeNetRequest for Referer spoofing) is a **future distribution channel** that could enable zero-server extraction for TikTok — but it's out of scope for v1.
- **Why this:** Pass 2 §10. Extension is a separate distribution with its own review processes.
- **Evidence:** cluster D; Pass 2 §10.
- **What would change it:** If the remote service's cost/legal risk grows, an extension distribution becomes attractive.
- **Confidence:** HIGH.

### AD-24: Local/remote boundary
- **Decision:**
  - **MUST run in browser**: UI, orchestration, stream-to-storage, local media processing (Mediabunny, WebCodecs), save-to-disk/share.
  - **MAY run in browser**: WebCodecs audio (iOS 26+), Background Fetch (Chrome), File System Access (Chrome).
  - **MUST run remotely**: extraction (TikTok/IG/FB), Referer-gated media fetch, residential-proxy egress (IG/FB).
  - **MAY run remotely**: extraction-result cache, health monitoring.
  - **CANNOT realistically run in browser**: yt-dlp (Python), multi-threaded WASM (no SAB on Pages), background download (iOS).
- **Why this:** cluster D; Pass 2 §10, §27, §28.
- **Evidence:** clusters B, C, D, E.
- **Confidence:** HIGH.

---

## 5. Technologies Becoming Core

(USE — required for the v1 architecture.)

1. GitHub Pages (app shell)
2. GitHub Releases (large assets)
3. GitHub Actions (CI/CD)
4. Vite (build)
5. React (UI)
6. TypeScript (language)
7. DedicatedWorker (off-main-thread)
8. ServiceWorker (app-shell cache only)
9. Streams API (large-file streaming)
10. Fetch / CORS (download)
11. OPFS (large-file staging)
12. IndexedDB (metadata)
13. Cache API (app shell)
14. WebCodecs (progressive — video universal, audio iOS 26+)
15. navigator.share (iOS saves)
16. Mediabunny (pure-TS media processing)
17. mp4box.js (ISOBMFF inspection)
18. Remote extractor service (Node.js)
19. Remote media proxy (with signed URLs)
20. yt-dlp (server-side, breadth backend)
21. Residential proxy (IG/FB egress)
22. Extraction-result cache (server, 30-60s TTL)
23. Health-check endpoint
24. Normalized MediaResult type
25. PlatformAdapter interface
26. CSP via `<meta>`
27. MIT license
28. .nojekyll + content-hashed asset URLs

---

## 6. Technologies Remaining Optional

(OPTIONAL — useful under particular conditions, not required for v1.)

1. SharedWorker (cross-tab coordination — if multi-tab matters)
2. Background Fetch API (Chrome enhancement — not on iOS)
3. File System Access API (Chrome desktop save-to-folder)
4. navigator.storage.persist() (requested, not guaranteed on iOS)
5. mp4-muxer / webm-muxer (if Mediabunny insufficient for a case)
6. Bun (alternative remote runtime)
7. Deno (alternative remote runtime)
8. Serverless (Lambda/Functions — low-traffic alternative)
9. Cobalt (reference architecture — study, don't copy)
10. jsDelivr CDN mirror (if GitHub Releases CORS fails — Exp 1)
11. Client-side error reporting (Sentry-style, opt-in)
12. Zig, AssemblyScript (if custom WASM needed later — Rust preferred)

---

## 7. Technologies Deferred

(USE-LATER — useful and justified, but deliberately deferred.)

1. FFmpeg.wasm escape hatch (loaded on demand, not in default flow) — **USE-LATER** (trigger conditions defined in AD-6)
2. Rust + wasm-bindgen (if a hot path needs custom WASM)
3. Component Model / WIT / jco (if cross-language WASM plugins needed)
4. WASI 0.2 / 0.3 (server-ready; browser via jco; defer)
5. Javy / QuickJS-in-WASM (if sandboxed third-party extractors needed)
6. Extism (if a plugin system is justified)
7. SQLite WASM / wa-sqlite (if history needs full-text search / complex queries)
8. Cloudflare Workers remote runtime (EXPERIMENTAL — Exp 3)
9. Multi-threaded WASM (deferred until host migration allows COOP/COEP)

---

## 8. Technologies Requiring Experiments

(EXPERIMENTAL — needs a concrete experiment before promotion.)

1. **Cloudflare Workers as remote extractor** (Exp 3) — Can a pure-JS TikTok scraper run within 128MB / 50ms CPU? If yes, edge runtime with no yt-dlp dependency.
2. **GitHub Releases CORS** (Exp 1) — Does `release-assets.githubusercontent.com` send CORS headers? If no, shift to jsDelivr.
3. **Mediabunny robustness** (Exp 2) — >90% success on real platform media? If no, escape hatch fires more often.
4. **iOS Safari FFmpeg.wasm memory cap** (Exp 5) — Practical WASM memory limit before tab crash. Sets iOS escape-hatch viability.
5. **Residential-proxy IG extraction** (Exp 6) — Residential >70% vs datacenter <20%? If residential fails, IG may drop from v1.
6. **TikTok CDN Range support** (Exp 4) — Does Range work on signed URLs? Affects resume strategy.
7. **Signed-URL TTL** (Exp 7) — Per-platform expiry measurement. Sets re-extraction threshold.
8. **navigator.share large-file on iOS** (Exp 8) — Can iOS share a 500MB video to Files? Affects iOS large-file UX.

---

## 9. Technologies Rejected

(REJECT — not part of the architecture, with replacement/reason.)

| Technology | Replacement | Reason |
|---|---|---|
| Next.js | Vite | Overkill for static Pages (no SSR/API routes needed) |
| Multi-threaded FFmpeg.wasm | Single-threaded FFmpeg.wasm (escape hatch) | Needs SAB → COOP/COEP → unavailable on Pages |
| SharedArrayBuffer | Single-threaded WASM / pure TS | Needs COOP/COEP, unavailable on Pages |
| coi-serviceworker (COOP/COEP shim) | Migrate host if SAB ever needed | Flaky, breaks embeds, not production-viable |
| Pyodide | yt-dlp server-side | Too heavy (6.4MB + 5s init); yt-dlp native deps don't work in browser |
| WebContainers | None | Overkill (dev-environment tool); needs COOP/COEP |
| Go → WASM | Rust (if WASM needed) / TypeScript | 10-25MB binaries, GC overhead, poor fit |
| gallery-dl | yt-dlp (server-side) | Python, not browser-relevant |
| Cobalt web frontend | Our own UI | CC-BY-NC-SA (NonCommercial) — cannot copy |
| Component Model / WIT / jco (for v1) | Direct WASM imports (if ever needed) | Build-time abstraction; no v1 value |
| Extism (for v1) | PlatformAdapter interface | Premature plugin system |
| SQLite WASM (for v1) | IndexedDB | IndexedDB suffices for v1 record types |
| sql.js | wa-sqlite (if needed later) | In-memory only; not for persistent DBs |
| Plugin architecture (v1) | PlatformAdapter interface + map | Premature; no demonstrated use case |
| Capability Engine | Direct `capabilities` object | Over-abstracted; one-liner per feature |
| Runtime Registry | `selectExtractionStrategy()` function | Over-abstracted |
| Extraction Registry | Plain object map | Over-abstracted |
| Media Planner | `planMediaProcessing()` function | Over-abstracted |
| Service Worker media interception | DedicatedWorker + remote proxy | No gain (CORS blocks); proxy handles it |

---

## 10. Final Minimum Technology Set

(ONLY technologies actually required by the v1 system. The smallest practical set.)

### Frontend (browser)
1. **TypeScript** — language
2. **React** — UI framework
3. **Vite** — build tool / dev server
4. **DedicatedWorker** — off-main-thread work
5. **ServiceWorker** — app-shell caching only
6. **Streams API** — large-file streaming
7. **OPFS** — large-file staging (sync access handle in worker)
8. **IndexedDB** — metadata/history/queue/settings
9. **Cache API** — app-shell cache
10. **WebCodecs** — frame decode/encode (progressive)
11. **navigator.share** — iOS saves
12. **Mediabunny** — pure-TS demux/mux/remux
13. **mp4box.js** — ISOBMFF inspection

### Hosting / distribution
14. **GitHub Pages** — app shell (with `.nojekyll`, content-hashed URLs, CSP via `<meta>`)
15. **GitHub Releases** — large assets (FFmpeg.wasm escape hatch)

### CI/CD
16. **GitHub Actions** — build/test/release/deploy/maintenance

### Remote service
17. **Node.js** — runtime
18. **yt-dlp** (via wrapper) — breadth extraction backend
19. **Per-platform JS scrapers** — TikTok/IG/FB extraction
20. **Residential proxy** — IG/FB egress
21. **Extraction-result cache** (in-memory or Redis) — 30-60s TTL

### Data types / interfaces
22. **MediaResult** — normalized extraction output
23. **PlatformAdapter** — interface + plain-object map

### License
24. **MIT** — project license

**Total: 24 items** (down from Pass 1's ~30 candidates). Everything else is optional, deferred, experimental, or rejected.

---

## 11. Zero-Server Boundary

**What works entirely from GitHub Pages + browser (zero server):**

1. **App shell loads** — HTML/JS/CSS served from Pages; works offline after first load (SW app-shell cache).
2. **Direct-media-URL download** — user pastes a direct CDN URL (not a post URL). If the CDN is CORS-permissive, the browser fetches, streams to OPFS, saves. No remote needed.
3. **Local media processing** — Mediabunny remux, WebCodecs decode/encode, mp4box.js inspection — all in-browser on files the user already has (e.g., from OPFS or a file picker).
4. **Permissive-CDN platform extraction** — for platforms that send CORS headers and don't require Referer (rare; not TikTok/IG/FB), the browser can extract via embedded-JSON scrape.
5. **Degraded mode** — if the remote service is down, the app still loads and handles the above; extraction-dependent platforms show "service unavailable."

**What does NOT work zero-server:**
- Extraction for TikTok/Instagram/Facebook (CORS + Referer wall).
- Fetching media bytes from Referer-gated CDNs (TikTok/IG/FB).
- yt-dlp (Python; not browser-runnable).

---

## 12. Minimum Remote Boundary

**What requires remote infrastructure:**

1. **Extraction** (`POST /extract`): resolve a post URL to direct media URLs + metadata. Must bypass CORS (server fetches, adds CORS headers to response) and inject Referer/cookies for the target platform.
2. **Media-byte proxying** (`GET /proxy?u=...&sig=...`): fetch media bytes from a Referer-gated CDN and re-serve to the browser with CORS headers. Used when the browser's direct fetch fails.
3. **Residential-proxy egress**: for Instagram/Facebook (datacenter IPs blocked). The remote service routes extraction requests through a residential proxy.
4. **Health monitoring** (`GET /health`): for the client's degraded-mode check and uptime monitoring.

**What the remote does NOT do:**
- Transcoding / media processing (browser does this).
- Storage of media (browser does this; remote is stateless).
- User accounts / authentication (none in v1).
- Search / discovery (user supplies URLs only).
- Hosting of the app shell (Pages does this).

---

## 13. Runtime Boundary

**What runs where:**

| Runtime | Runs |
|---|---|
| **Browser main thread** | UI, React rendering, user input, capability detection, orchestration (calls workers/remote) |
| **DedicatedWorker** | Download streaming (fetch → OPFS), media parsing (Mediabunny, mp4box.js), WebCodecs decode/encode, FFmpeg.wasm (escape hatch) |
| **SharedWorker** (optional) | Cross-tab download queue coordination (if adopted) |
| **ServiceWorker** | App-shell caching (offline UI); update notifications |
| **WASM** | FFmpeg.wasm (escape hatch only, single-threaded, in DedicatedWorker) |
| **Remote service (Node.js)** | Extraction (per-platform scrapers + yt-dlp), media proxying, SSRF defense, rate limiting, residential-proxy egress, extraction-result cache |
| **GitHub Actions (CI)** | Build, test, deploy to Pages, upload Releases, scheduled maintenance |

---

## 14. Dependency Boundaries

**Client (browser) dependencies:**
- `react`, `react-dom` — UI
- `vite` — build (dev dependency)
- `mediabunny` — media processing (or vendored)
- `mp4box.js` — ISOBMFF inspection
- (optional, lazy) `@ffmpeg/ffmpeg`, `@ffmpeg/core` — escape hatch, loaded from Releases at runtime, not bundled

**Remote service dependencies:**
- `node` runtime
- `youtube-dl-exec` or `yt-dlp-exec` (wraps yt-dlp binary) — breadth extraction
- (optional) `ioredis` — extraction-result cache (or in-memory Map)
- HTTP framework (e.g., `fastify` or native `http`) — server
- (optional) residential-proxy SDK (provider-specific)

**CI dependencies:**
- `actions/checkout`, `actions/setup-node`, `actions/deploy-pages` — standard actions (pinned to SHA)

**Boundary rules:**
- Client and remote share **only** the `MediaResult` type (via a shared types package or duplicated TS interface).
- No client dependency on the remote's runtime (the client calls the remote via HTTP).
- No remote dependency on client libraries (the remote doesn't import React etc.).

---

## 15. Security Boundaries

### Remote service (primary attack surface)
- **SSRF**: per-platform host allowlist (only known platform + CDN domains); private-IP rejection (10/8, 172.16/12, 192.168/16, 127/8, 169.254/16 incl. cloud metadata, ::1, fc00::/7); cloud-metadata block; redirect re-validation (or disable redirects); DNS-rebinding defense (pin resolved IP).
- **Open proxy**: signed proxy URLs (HMAC-SHA256, 60s TTL); per-IP rate limit (e.g., 30 req/min); host allowlist on `/proxy`; file-size cap (e.g., 2GB); timeout (e.g., 60s); the `/proxy` is opt-in (only when `proxyRecommended` or direct fetch fails).
- **DoS**: rate limit, size cap, timeout, queue (reject if overloaded).
- **No cookie storage by default**: stateless; if cookies needed for a platform, hold in-memory for the request duration only.

### Browser (client)
- **CSP via `<meta>`**: restrict script/style/img/connect sources to self + GitHub Releases (or jsDelivr) + remote service origin.
- **Worker isolation**: media parsing in DedicatedWorker; timeouts; file-size caps; prefer pure-TS parsers (no memory-corruption risk).
- **WASM supply chain**: SRI (Subresource Integrity) hash on `.wasm` files; pinned builds (commit SHA); verify against reproducible build if possible.
- **Untrusted URLs**: user-supplied URLs go to the remote (which has SSRF defense); never `eval` or inject into DOM.
- **Secrets**: zero secrets on client (everything client-side is public).

### CI/CD
- `permissions: contents: read` default; per-job escalation only.
- Pin third-party actions to commit SHA.
- Avoid `pull_request_target` with `run:` interpolation of untrusted input.
- OIDC for any cloud deploys (no static cloud credentials).
- Secrets in GitHub Secrets only; never in code/logs/commits.

### Content legality
- No search/discovery (user supplies URLs).
- No hosting of media (bytes flow through proxy or direct; not stored).
- User-initiated downloads only.
- Clear "personal use" framing in UI/README.
- MIT license; reference youtube-dl/RIAA precedent (reversed) if challenged.

---

## 16. Performance Constraints

| Metric | Budget | Basis |
|---|---|---|
| Initial bundle (gzipped) | < 300 KB | React + Vite + app code; lazy-load the rest |
| WASM in initial bundle | 0 KB | Pure-TS default; FFmpeg.wasm lazy from Releases |
| Mediabunny lazy-load | ~50-100 KB | Loaded on first download |
| FFmpeg.wasm load (escape hatch) | ~25 MB | From Releases; rare; 1-3s init |
| DedicatedWorker startup | < 50 ms | Native spawn |
| Stream-to-OPFS throughput | Network-bound | Not browser-bound |
| iOS WASM memory cap | ~256-512 MB | Exp 5; below the ~2GB emscripten default that crashes iOS |
| Remote extraction latency | 1-5 s typical | HTML fetch + parse + return |
| Remote proxy overhead | +1 round-trip + bandwidth | Only when needed |
| Pages cache TTL | 10 min (max-age=600) | Defeated by content-hashed URLs |
| Pages bandwidth | 100 GB/mo (soft) | App shell only; media doesn't flow through Pages |

**Items marked EXPERIMENTAL** (budgets not yet empirically validated):
- iOS WASM memory cap (Exp 5).
- Mediabunny throughput on real platform media (Exp 2).
- Remote extraction latency under load.

---

## 17. Safari/iOS Constraints

(These shape the entire design.)

1. **7-day storage eviction**: all origin storage (SW, IDB, OPFS, Cache) evicted after 7 days of no Safari use. `navigator.storage.persist()` is requested but not reliably honored. → **No local media library on iOS**; downloads hand off to OS Share Sheet immediately.
2. **No background download**: iOS suspends JS when tab hidden; SW killed within seconds. No Background Sync / Periodic Background Sync / Background Fetch API. → **Foreground-only downloads**; resume-on-revisit via OPFS byte tracking + re-extraction.
3. **Memory limits**: tab crashes well below ~2GB (often few hundred MB JS heap). → **Cap WASM at ~256-512MB**; stream files (never buffer whole file); avoid large WASM in default flow.
4. **WebCodecs audio needs iOS 26+** (Safari 26, Sep 2025). Older iOS (16.4-25) has video only. → **Progressive enhancement**; fallback to passthrough or FFmpeg.wasm escape hatch for audio on older iOS.
5. **`<a download>` unreliable for media** (opens inline). → **navigator.share({files})** for reliable save-to-Files/camera-roll; requires user gesture.
6. **No File System Access pickers** (showSaveFilePicker etc.). → **OPFS for staging**; Share Sheet for export.
7. **SharedWorker supported** (iOS 16+). → Available for cross-tab coordination if adopted.
8. **OPFS supported** (iOS 15.2+), sync access handle in worker. → Primary large-file staging.
9. **No SharedArrayBuffer** (needs COOP/COEP; Pages can't set). → **No multi-threaded WASM** on Pages-hosted app.
10. **iPadOS**: same API support as iOS Safari (desktop UA); no special handling needed.

---

## 18. Remaining Unknowns

(To be resolved by experiments — §19 — or accepted as platform limits.)

1. GitHub Releases CORS for browser fetch (Exp 1).
2. Mediabunny robustness on real TikTok/IG/FB media (Exp 2).
3. Cloudflare Workers viability for pure-JS extraction (Exp 3).
4. TikTok CDN Range-request support on signed URLs (Exp 4).
5. iOS Safari practical WASM memory cap (Exp 5).
6. Residential-proxy IG extraction success rate (Exp 6).
7. Per-platform signed-URL TTL (Exp 7).
8. navigator.share large-file success on iOS (Exp 8).
9. Whether Cobalt uses yt-dlp internally (inferred pure-JS, unconfirmed — not blocking).
10. Whether an LGPL-only FFmpeg.wasm build (no x264) is practically usable for our remux needs (not blocking — GPL build as optional plugin is acceptable).
11. Whether AGPL on a Cobalt-derived remote service "infects" the client (legal consensus: no, but untested — not blocking if we build our own service).
12. Exact iOS Safari tab memory limit (no published number — Exp 5 will bound it).

---

## 19. Required Experiments

(Each must be run before the corresponding decision is finalized in implementation.)

### Exp 1: GitHub Releases CORS
- **Question:** Does `release-assets.githubusercontent.com` send `Access-Control-Allow-Origin`?
- **Setup:** Upload a test asset to a Release; `fetch()` from a `*.github.io` page; inspect headers.
- **Success:** CORS header present; fetch succeeds.
- **Failure:** Use jsDelivr mirror (`cdn.jsdelivr.net/gh/...`).
- **Consequence:** Determines FFmpeg.wasm distribution URL.
- **Decision dependency:** AD-6 (escape hatch), GitHub Releases USE.

### Exp 2: Mediabunny robustness
- **Question:** >90% parse/mux success on real TikTok/IG/FB media?
- **Setup:** 20 sample files; run Mediabunny parse + remux.
- **Success:** ≥90%.
- **Failure:** Investigate; may need mp4box.js fallback or more FFmpeg.wasm reliance.
- **Consequence:** Escape-hatch trigger frequency; may shift AD-6 toward FFmpeg.wasm-by-default.
- **Decision dependency:** AD-6.

### Exp 3: Cloudflare Workers extractor
- **Question:** Can pure-JS TikTok scraper run within 128MB / 50ms CPU?
- **Setup:** Port scraper to JS; deploy to CF Workers; test 50 URLs.
- **Success:** ≥80%; <50ms CPU; <128MB.
- **Failure:** Node.js + yt-dlp on VPS remains the path.
- **Consequence:** May enable edge runtime (cheaper, faster, no yt-dlp dependency).
- **Decision dependency:** Cloudflare Workers EXPERIMENTAL → may promote to USE.

### Exp 4: TikTok CDN Range support
- **Question:** Does TikTok CDN honor `Range:` on signed URLs?
- **Setup:** Abort download at 50%; resume with `Range: bytes=N-`.
- **Success:** 206 Partial Content.
- **Failure:** Resume restarts from zero.
- **Consequence:** AD-15 resume strategy (resume vs restart).
- **Decision dependency:** AD-15.

### Exp 5: iOS Safari FFmpeg.wasm memory cap
- **Question:** Practical WASM linear-memory cap before tab crash?
- **Setup:** Load FFmpeg.wasm with MAXIMUM_MEMORY 256M/512M/1G/2G; process 500MB file.
- **Success:** Identify safe cap.
- **Failure:** Even 256M crashes → disable escape hatch on iOS.
- **Consequence:** iOS escape-hatch viability; performance budget.
- **Decision dependency:** AD-6 (escape hatch on iOS).

### Exp 6: Residential-proxy IG extraction
- **Question:** Residential >70% vs datacenter <20%?
- **Setup:** Run IG extractor via BrightData vs datacenter; test 30 URLs.
- **Success:** Residential ≥70%.
- **Failure:** Reconsider IG as v1 target.
- **Consequence:** IG viability in v1; proxy cost.
- **Decision dependency:** AD-7 (extraction engine), Residential proxy USE.

### Exp 7: Signed-URL TTL
- **Question:** How long are TikTok/IG/FB CDN URLs valid?
- **Setup:** Extract; fetch at 0/5/10/30/60/120 min.
- **Success:** Measure per-platform TTL.
- **Failure:** N/A (measurement).
- **Consequence:** AD-15 re-extraction threshold per platform.
- **Decision dependency:** AD-15.

### Exp 8: navigator.share large-file on iOS
- **Question:** Can iOS share a 500MB video to Files?
- **Setup:** Share a 500MB MP4 from OPFS on iOS Safari.
- **Success:** Save succeeds within ~30s.
- **Failure:** Chunk or advise smaller files.
- **Consequence:** iOS large-file UX.
- **Decision dependency:** AD-15, AD-22.

---

## 20. Decision Dependencies

(Graph of which decisions depend on which.)

```
AD-1 (browser-executed, remote-assisted) 
  ├─ depends on: cluster D (CORS/Referer wall), cluster C (browser-capable processing)
  └─ enables: AD-2, AD-3, AD-24

AD-2 (zero-server degraded mode)
  └─ depends on: AD-1 (knowing what's browser-capable)

AD-3 (capability-based selection)
  └─ depends on: AD-1

AD-4 (minimum backend API)
  ├─ depends on: cluster D, AD-3
  └─ enables: AD-5, AD-19

AD-5 (remote proxying when required)
  └─ depends on: AD-4

AD-6 (media engine: pure-TS + escape hatch)
  ├─ depends on: cluster C, Exp 2, Exp 5
  └─ affects: AD-15 (large-file strategy)

AD-7 (extraction engine: scrapers + yt-dlp)
  ├─ depends on: cluster D, Exp 6
  └─ enables: AD-4

AD-8 (platform adapter model)
  └─ depends on: AD-3

AD-11 (worker architecture)
  └─ depends on: cluster B, E

AD-12 (storage architecture)
  └─ depends on: cluster E

AD-13 (caching)
  └─ depends on: AD-4

AD-14 (service worker app-shell only)
  └─ depends on: cluster B, E, AD-11

AD-15 (large-file strategy + resume-on-expiry)
  ├─ depends on: AD-6, AD-12, Exp 4, Exp 7
  └─ affects: AD-22 (mobile)

AD-19 (security boundaries)
  └─ depends on: AD-4, AD-5

AD-20 (Pages responsibilities)
  └─ depends on: cluster A

AD-21 (Actions responsibilities)
  └─ depends on: cluster A

AD-22 (desktop/mobile boundary)
  └─ depends on: AD-15, cluster E

AD-24 (local/remote boundary)
  └─ depends on: all above (synthesis)

Exp 1 (GH Releases CORS) → affects: AD-6 escape-hatch distribution
Exp 2 (Mediabunny) → affects: AD-6 escape-hatch frequency
Exp 3 (CF Workers) → affects: AD-7 (may promote CF Workers to USE)
Exp 4 (TikTok Range) → affects: AD-15
Exp 5 (iOS memory) → affects: AD-6 (iOS escape-hatch viability)
Exp 6 (Residential proxy) → affects: AD-7 (IG viability)
Exp 7 (URL TTL) → affects: AD-15
Exp 8 (iOS share large file) → affects: AD-15, AD-22
```

---

## 21. Conditions That Would Change Decisions

| Decision | Condition that would change it | New decision |
|---|---|---|
| GitHub Pages as host | If we need SAB/multi-threaded WASM for a proven hot path | Migrate to Cloudflare Pages/Netlify (allows COOP/COEP) |
| GitHub Releases for assets | Exp 1 fails (no CORS) | Use jsDelivr CDN mirror |
| Mediabunny as default media engine | Exp 2 < 80% success | Shift to FFmpeg.wasm-by-default (accept GPL/cost) |
| FFmpeg.wasm as escape hatch | Exp 5 shows iOS can't run it even at 256MB | Disable escape hatch on iOS; pure-TS only |
| Node.js remote runtime | Exp 3 succeeds (CF Workers viable) | Migrate remote to Cloudflare Workers (edge, free tier) |
| IG as v1 target | Exp 6 < 70% residential success | Drop IG from v1; add later |
| yt-dlp as breadth backend | A platform's per-platform scraper is unmaintainable | Rely more on yt-dlp generic extractors |
| REJECT plugin architecture | A demonstrated third-party-contributor use case appears | Adopt Extism-based plugin system |
| REJECT SQLite WASM | History needs full-text search / complex analytics | Adopt wa-sqlite + OPFSCoopSyncVFS |
| REJECT Component Model | We adopt cross-language WASM plugins | Integrate WIT/jco for plugin interfaces |
| MIT project license | We decide to fork Cobalt's API wholesale | Project or service component becomes AGPL |
| Remote service on VPS | Traffic grows beyond VPS capacity | Migrate to container platform (Fly.io/Railway) or serverless |
| No browser extension | Remote service cost/legal risk grows | Build a manifest-v3 extension for zero-server extraction |

---

## END OF PASS 3 DECISION LEDGER

**This ledger is the formal decision system. Pass 4 (not in scope for this pass) will consume it to produce the final architecture and implementation roadmap.**

**Validation checklist (per protocol):**
- [x] Every major technology has exactly one decision state (USE / USE-LATER / OPTIONAL / EXPERIMENTAL / REJECT)
- [x] No major decision lacks evidence (each cites Pass 1 cluster + Pass 2 section)
- [x] Pass 1 and Pass 2 are explicitly represented (§2 Inputs; every decision row)
- [x] Rejected technologies have replacements/reasons (§9)
- [x] Experimental technologies have concrete experiments (§8, §19)
- [x] Minimum technology set is actually minimal (§10: 24 items, down from ~30)
- [x] No implementation code changed (planning artifacts only)
- [x] No secrets present (verified: no token in any file)
