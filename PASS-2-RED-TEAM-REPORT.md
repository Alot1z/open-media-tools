# PASS 2 — RED-TEAM / FEASIBILITY / COUNTER-RESEARCH REPORT

**Project:** open-media-tools
**Repository:** `Alot1z/open-media-tools`
**Pass:** 2 (Red-team / Feasibility / Counter-research)
**Date:** 2026 (current)
**Status:** COMPLETE
**Input consumed:** `PASS-1-RESEARCH-REPORT.md` (commit `4a79534`) + `.research/cluster-{A..F}-*.md`

---

## Methodology

Every major conclusion from Pass 1 is treated as a **hypothesis**. For each, this pass:
1. States the Pass 1 claim.
2. Challenges it independently (searches for counterexamples, hidden constraints, better alternatives).
3. Determines whether it **survives**, **fails**, or is **overengineered/underengineered**.
4. Assigns a provisional decision state: USE / USE-LATER / OPTIONAL / EXPERIMENTAL / REJECT.

Failure scenarios are tested against the proposed architecture. A provisional decision matrix and experiment plan close the pass. **No final architecture is produced** — that is Pass 3.

---

## 1. Executive Summary

Pass 1's research is largely sound, but red-team scrutiny exposes **five conclusions that must be revised** and **two that fail outright**:

**Failed assumptions:**
- **"Browser-first execution" as the framing is misleading.** For the initial targets (TikTok/IG/FB), extraction is *unavoidably* remote. The honest framing is "browser-executes-everything-except-extraction-and-Referer-gated-fetch." Calling it "browser-first" oversells what the browser does and hides the remote service's centrality. **REVISE the framing.**
- **"Minimal remote infrastructure" understates the remote service.** The remote service must do extraction (CORS bypass), Referer injection, cookie handling, residential-proxy egress, media proxying, SSRF defense, rate limiting, and signed-URL minting. This is not "minimal" — it's the majority of the system's operational complexity. **REVISE the scope honesty.**

**Overengineered decisions:**
- **Plugin architecture (premature).** No concrete plugin requirement has been demonstrated for v1. The three initial platforms can be handled by a simple platform-adapter interface. A full plugin system (WASM plugins, Extism, registry) is YAGNI until a real third-party-contributor use case exists. **DEFER.**
- **Service Worker for anything beyond app-shell caching.** Pass 1 included SW for offline + media interception. On iOS, SW is evicted in 7 days and killed on background — investing in SW-mediated download interception is low-ROI. **REDUCE to app-shell caching only.**
- **Capability Engine / Runtime Registry / Media Planner abstractions.** These are architecture-for-architecture's-sake until v1 proves they're needed. A direct platform-adapter map + a download function suffices. **REJECT for v1.**

**Underengineered decisions:**
- **Signed-URL expiry handling.** Pass 1 noted expiry but didn't specify the re-extraction-on-expiry flow. A large download that stalls near completion, then re-extracts, then resumes — the state machine is non-trivial and must be designed.
- **Residential-proxy egress for IG/FB.** Pass 1 flagged it as "LIKELY needed" but didn't cost or source it. This is a real ongoing operational cost (~$50-500/mo depending on volume) that affects the project's viability as a free service.

**Surviving assumptions (the load-bearing ones):**
- GitHub Pages hosts the app shell; large assets via GitHub Releases. ✓
- No COOP/COEP on Pages → no SAB → no multi-threaded WASM. ✓ (CONFIRMED, not a Pass 1 error)
- Pure-TS media processing (Mediabunny + mp4box.js + WebCodecs) for v1; FFmpeg.wasm as escape hatch. ✓
- Remote extractor service is unavoidable for TikTok/IG/FB. ✓
- iOS 7-day eviction + no background download constrain the design. ✓
- MIT license with FFmpeg.wasm as optional plugin. ✓

**Net effect on the technology set:** the minimum viable set shrinks. Pass 1's candidate list had ~30 technologies; the red-team-surviving core is ~15.

---

## 2. PASS 1 Assumptions Tested

| # | Pass 1 Assumption | Verdict | Confidence |
|---|---|---|---|
| A1 | GitHub Pages is the right production host | SURVIVES (with caveat: COOP/COEP limitation is binding) | HIGH |
| A2 | GitHub Releases distribute large WASM/binary assets | SURVIVES (needs CORS empirical test — see Unknowns) | MEDIUM |
| A3 | No COOP/COEP on Pages → no SAB → no multi-threaded WASM | SURVIVES (CONFIRMED, hard platform limit) | HIGH |
| A4 | React + Vite + TypeScript for frontend | SURVIVES (React challenged but defensible; Vite clearly right) | MEDIUM |
| A5 | Next.js is overkill | SURVIVES | HIGH |
| A6 | Web Workers / SharedWorker / ServiceWorker available | SURVIVES (but SW role reduced) | HIGH |
| A7 | OPFS for large-file staging | SURVIVES | HIGH |
| A8 | IndexedDB for metadata | SURVIVES (SQLite WASM deferred) | HIGH |
| A9 | WebCodecs progressive enhancement | SURVIVES (audio iOS 26+ caveat) | HIGH |
| A10 | Mediabunny + mp4box.js replace FFmpeg.wasm for v1 | SURVIVES (needs empirical robustness test) | MEDIUM |
| A11 | FFmpeg.wasm as escape hatch, loaded on demand | SURVIVES (but escape-hatch trigger conditions must be defined) | MEDIUM |
| A12 | WASM not necessary for v1 | SURVIVES | HIGH |
| A13 | Rust not necessary for v1 | SURVIVES | HIGH |
| A14 | Component Model / WASI 0.2/0.3 deferred | SURVIVES | HIGH |
| A15 | Remote extractor service unavoidable for TikTok/IG/FB | SURVIVES (CONFIRMED) | HIGH |
| A16 | Remote service returns direct URLs OR proxies media | SURVIVES (proxy is the common case) | HIGH |
| A17 | Remote service can be stateless | PARTIALLY FAILS (cookies/rate-limit may force state) | MEDIUM |
| A18 | "Browser-first" framing | FAILS (misleading; extraction is remote) | HIGH |
| A19 | "Minimal remote infrastructure" | FAILS (remote service is the operational core) | HIGH |
| A20 | Plugin architecture | OVERENGINEERED (defer) | HIGH |
| A21 | Capability Engine / Runtime Registry / Media Planner | OVERENGINEERED (reject for v1) | HIGH |
| A22 | Service Worker for offline + media interception | OVERENGINEERED (reduce to app-shell cache) | HIGH |
| A23 | SQLite WASM for v1 | OVERENGINEERED (IndexedDB suffices) | HIGH |
| A24 | Extism for plugins | OVERENGINEERED (defer) | HIGH |
| A25 | MIT project license | SURVIVES | HIGH |
| A26 | FFmpeg.wasm as optional GitHub Releases plugin (GPL isolation) | SURVIVES | HIGH |
| A27 | Scheduled Actions for maintenance | SURVIVES (must be idempotent + self-healing) | HIGH |
| A28 | navigator.share for iOS saves | SURVIVES | HIGH |
| A29 | Background Fetch API as Chrome enhancement | SURVIVES | HIGH |
| A30 | "coi-serviceworker" COOP/COEP shim | REJECT (flaky, not production-viable) | HIGH |

---

## 3. Surviving Assumptions

(Carried forward to Pass 3 as load-bearing evidence.)

1. **GitHub Pages hosts the app shell.** 1GB site / 100GB bandwidth / 10-min cache. Content-hashed assets + `.nojekyll`. (A1, CONFIRMED)
2. **GitHub Releases distributes large assets** (FFmpeg.wasm). 2GB/asset, uncapped bandwidth. (A2, needs CORS test)
3. **No COOP/COEP on Pages → no SAB → single-threaded WASM only.** (A3, CONFIRMED hard limit)
4. **React + Vite + TypeScript.** Vite is clearly correct. React is defensible (contributor pool, ecosystem); a smaller framework would save ~30-50KB gzipped but cost contributor friction. (A4)
5. **DedicatedWorker + SharedWorker for off-main-thread work.** SharedWorker on iOS 16+. (A6)
6. **OPFS sync access handle (worker) for large-file streaming writes.** (A7)
7. **IndexedDB for metadata/history/queue.** (A8)
8. **Cache API + Service Worker for app-shell offline caching only.** (A9, reduced scope)
9. **WebCodecs as progressive enhancement** (video iOS 16.4+, audio iOS 26+; fallback to passthrough). (A9)
10. **Mediabunny + mp4box.js for pure-TS demux/mux/remux.** (A10, needs robustness test)
11. **FFmpeg.wasm escape hatch**, loaded from GitHub Releases only when the pure-TS path can't handle the input. (A11)
12. **No WASM in the default v1 flow.** (A12)
13. **No Rust in v1.** (A13)
14. **Component Model / WASI deferred.** (A14)
15. **Remote extractor service is unavoidable** for TikTok/IG/FB. Does extraction (CORS bypass), Referer injection, cookie handling, media proxying when needed. (A15, A16)
16. **MIT project license.** FFmpeg.wasm (GPL) isolated as optional plugin. (A25, A26)
17. **navigator.share for iOS saves**, `<a download>` + Blob for desktop. (A28)
18. **Background Fetch API as Chrome enhancement.** (A29)

---

## 4. Failed Assumptions

### 4.1 "Browser-first execution" framing (A18) — FAILS
**Pass 1 claim:** The project is "browser-first" — browser/local execution wherever technically possible.
**Challenge:** For the three initial targets, the browser cannot do extraction (CORS), cannot fetch the media bytes directly (Referer-gated CDNs), and cannot run yt-dlp (Python). The browser's actual role is: **UI, orchestration, stream-to-storage, and local media processing (remux/transcode via Mediabunny/WebCodecs).** Extraction and media-byte-fetching are remote. Calling this "browser-first" is technically true (the browser *orchestrates*) but misleadingly implies the browser does the heavy lifting.
**Counter-evidence:** Cluster D confirmed all three platforms block cross-origin HTML fetch and use Referer-gated signed CDN URLs. The `Referer` header is a forbidden header (MDN). No browser API works around this.
**Revision:** Reframe as **"browser-executed, remote-assisted."** The browser runs all UI, processing, and storage; the remote service runs only what the browser *cannot* (extraction + Referer-gated fetching). This is honest and matches reality.

### 4.2 "Minimal remote infrastructure" (A19) — FAILS
**Pass 1 claim:** Minimal remote infrastructure.
**Challenge:** The remote service must do: extraction (per-platform scrapers or yt-dlp), CORS-header injection, Referer/cookie injection, residential-proxy egress (for IG/FB), media-byte proxying, SSRF defense, rate limiting, signed-URL minting, extraction-result caching, and health monitoring. This is the majority of the system's operational complexity and all of its ongoing cost.
**Counter-evidence:** Cobalt (the reference architecture) is a substantial service — Docker, auth, multiple instances, residential proxy integration. It is not "minimal."
**Revision:** Reframe as **"remote service is the operational core; the browser is the execution surface."** Acknowledge the remote service's complexity honestly. The "minimality" goal applies to *what the remote does* (extraction + proxy only — no transcoding, no storage, no user accounts), not to its implementation complexity.

---

## 5. Incorrect Decisions

### 5.1 Including a Service Worker for media interception — INCORRECT
**Pass 1 implication:** SW could intercept fetch() to media URLs and stream to cache/OPFS.
**Why incorrect:** (a) On iOS, SW is evicted in 7 days and killed on background — unreliable. (b) Cross-origin media fetch via SW still hits CORS (SW is same-origin to the app, not to the CDN). (c) The proxy path already handles Referer-gated fetches server-side. (d) SW interception adds complexity (request routing, fallback handling) for no gain over a direct `fetch(proxyUrl)` in a DedicatedWorker.
**Correction:** SW is for **app-shell caching only** (offline UI). Media download goes through a DedicatedWorker that calls the remote proxy directly.

### 5.2 Treating "browser-first" and "remote fallback" as peers — INCORRECT
**Pass 1 implication:** Browser-first with remote fallback where browser can't.
**Why incorrect:** For the initial targets, the remote is not a "fallback" — it's the **primary** path (extraction is always remote). "Fallback" implies the browser tries first and falls back; reality is the browser always needs the remote for these platforms.
**Correction:** Define a **capability-based selection**: for a given platform, the system knows upfront whether extraction is browser-native (rare) or remote (common for TikTok/IG/FB). There's no "try browser, fall back to remote" — there's "use the right tool for this platform's known constraints."

---

## 6. Overengineered Decisions

### 6.1 Plugin architecture (A20) — OVERENGINEERED
**Challenge:** What concrete plugin requirement exists for v1? The three initial platforms are handled by a simple platform-adapter interface (a map of `platformId → adapter`). A full plugin system (WASM plugins via Extism, plugin registry, sandboxed execution, plugin manifest schema) solves a problem that doesn't exist yet.
**Evidence against:** No third-party contributor is waiting to write plugins. The maintenance burden of per-platform extractors is real, but it's handled by the remote service's scraper code, not by a client-side plugin system.
**Verdict:** REJECT for v1. Use a simple `PlatformAdapter` interface (TS). Revisit a plugin system only when (a) there's a demonstrated third-party-contributor use case AND (b) the adapter interface proves insufficient.

### 6.2 Capability Engine / Runtime Registry / Media Planner (A21) — OVERENGINEERED
**Challenge:** These are classic "architecture astronaut" abstractions. A "Capability Engine" that detects browser features, a "Runtime Registry" that selects between browser/remote, a "Media Planner" that decides the processing pipeline — each is a layer of indirection that could be a function.
**Evidence against:** For v1, capability detection is `if ('VideoDecoder' in window)` (one line). Runtime selection is `if (platform.needsRemote) callRemote() else callBrowser()`. Media planning is "if input is MP4 with H.264, passthrough; else remux via Mediabunny; else escape-hatch FFmpeg.wasm." These don't need engines/registries/planners — they need functions.
**Verdict:** REJECT for v1. Use direct functions. Abstract only when duplication appears (rule of three).

### 6.3 Service Worker beyond app-shell cache (A22) — OVERENGINEERED
(See §5.1.) Reduce SW to app-shell caching only.

### 6.4 SQLite WASM for v1 (A23) — OVERENGINEERED
**Challenge:** Does v1's metadata/history/queue need SQL? The data is: download history (list of {url, platform, date, status, file path}), current queue (list of pending downloads), user settings (key-value). This is trivially IndexedDB. SQL's power (joins, complex queries) is unused.
**Verdict:** REJECT for v1. IndexedDB. Revisit SQLite WASM only if we add full-text search across history or complex analytics.

### 6.5 Extism for plugins (A24) — OVERENGINEERED
(See §6.1.) Moot once plugin architecture is rejected.

---

## 7. Underengineered Decisions

### 7.1 Signed-URL expiry + re-extraction flow — UNDERENGINEERED
**Gap:** Pass 1 noted that extracted CDN URLs expire in minutes-hours, but didn't specify the state machine for: download starts → URL expires mid-download → re-extract → resume from byte offset (or restart). This needs explicit design.
**Required design (for Pass 3):**
- Track download progress (bytes written to OPFS) in IndexedDB.
- On fetch failure (403/network), check if URL is likely expired (time since extraction > TTL/2).
- If expired: call remote `/extract` again → get new URL → resume with `Range: bytes=N-` (if CDN supports) or restart.
- Cap re-extraction attempts (e.g., 3) before failing the download.
- Surface to user: "Reconnecting..." status.

### 7.2 Residential-proxy egress cost/sourcing — UNDERENGINEERED
**Gap:** Pass 1 flagged that IG/FB block datacenter IPs, requiring residential-proxy egress. But didn't cost it or identify providers.
**Required investigation (for Pass 3 or experiment):**
- Providers: BrightData, Oxylabs, Smartproxy, Soax — pricing ~$1-15/GB residential.
- Volume estimate: per-extraction fetch is small (HTML/JSON, ~50-500KB); media proxying is the cost driver (if we proxy, we pay for the media bytes through the proxy). **Avoid proxying media bytes through residential proxies** — extract via residential, then fetch media directly from the CDN via the server's datacenter IP if the CDN allows, or via a cheaper datacenter proxy.
- Alternative: run the extractor on a residential-scale IP (home server, user's own device via extension). Out of scope for v1.

### 7.3 iOS download UX — UNDERENGINEERED
**Gap:** Pass 1 said "use navigator.share" but didn't address: what if the user wants to download multiple files? Share sheet is one-file-at-a-time. What about background download failure recovery?
**Required design:** For batch, either (a) zip multiple files in OPFS then share the zip, or (b) trigger multiple share sheets (poor UX). For failure recovery, on re-foreground, check IndexedDB for incomplete downloads and offer resume.

---

## 8. Hidden Constraints

1. **GitHub Releases CORS (UNCERTAIN).** Post-2025 domain change to `release-assets.githubusercontent.com`. If it doesn't send `Access-Control-Allow-Origin`, the browser cannot `fetch()` FFmpeg.wasm from there — we'd need a CORS proxy or a different distribution (jsDelivr CDN mirrors npm/GH releases with CORS). **Must empirically test before committing.**
2. **CDN Range-request support varies.** TikTok/IG/FB CDNs may not honor `Range:` on signed URLs (the signature may cover the full request, not a byte range). Resume may be impossible for some platforms — must restart from zero.
3. **Residential-proxy reliability.** Residential proxies have variable uptime and speed. A flaky proxy degrades extraction success rate.
4. **Platform ToS enforcement is asymmetric.** TikTok is relatively permissive (download tools widely exist). IG/FB are aggressive (rate limits, account bans, legal threats). The architecture must not assume uniform platform behavior.
5. **WASM linear memory cap on iOS.** Emscripten's default `MAXIMUM_MEMORY` (2GB) crashes iOS Safari. Must cap at ~256-512MB for any WASM we ship. This limits FFmpeg.wasm's usable file size on iOS.
6. **`navigator.share({files})` requires user gesture.** Cannot auto-trigger; must be from a click/tap. Affects download-complete UX flow.
7. **Content-hashed asset URLs on Pages.** The 10-min `max-age=600` cache means a new deploy takes up to 10 min to reach all users (cache miss on the HTML, which references new hashed assets). Acceptable but worth noting.
8. **GitHub Actions scheduled-cron unreliability.** A daily maintenance cron may run hours late or skip. Idempotent + triggered-on-push is mandatory.

---

## 9. GitHub Pages Audit

| Claim | Verdict |
|---|---|
| Pages can host the app shell (HTML/JS/CSS) | SURVIVES |
| Pages can serve `.wasm` with correct MIME (with `.nojekyll`) | SURVIVES |
| Pages can serve ES modules | SURVIVES |
| Pages supports custom domains + HTTPS | SURVIVES |
| Pages can set COOP/COEP | FAILS (no header control) |
| Pages can set CSP via headers | FAILS (but `<meta>` works) |
| Pages can host FFmpeg.wasm (~25MB) | SURVIVES technically, but wastes the 1GB site budget; use Releases instead |
| Pages can handle SPA routing | SURVIVES (via 404.html trick) |
| Pages bandwidth (100GB/mo) suffices for app shell | SURVIVES (app shell is small; media goes direct/proxy, not through Pages) |
| Pages is the right host long-term | SURVIVES for app shell; if we ever need COOP/COEP, migrate to Cloudflare Pages/Netlify |

**Net:** Pages is correct for v1 app shell. The COOP/COEP limitation is binding but acceptable because v1 doesn't need SAB.

---

## 10. Browser Extraction Audit

| Claim | Verdict |
|---|---|
| Pure browser extraction viable for TikTok/IG/FB | FAILS (CORS + Referer wall) |
| oEmbed solves extraction | FAILS (returns embed HTML, not media URLs) |
| Browser extension can bypass CORS + Referer | SURVIVES (but extension is a separate distribution, not the Pages site) |
| Remote extractor is unavoidable | SURVIVES |
| Remote can return direct URLs | SURVIVES (rare; permissive CDNs) |
| Remote must proxy media bytes | SURVIVES (common case for TikTok/IG/FB) |
| "Try browser, fall back to remote" model | FAILS (use capability-based selection instead) |

**Net:** Extraction is remote for the initial targets. The browser's role is orchestration + processing, not extraction.

---

## 11. Safari/iOS Audit

| Claim | Verdict |
|---|---|
| 7-day storage eviction is unavoidable | SURVIVES (CONFIRMED, hard limit) |
| `navigator.storage.persist()` survives the 7-day cap | FAILS (unreliable) |
| Background download is possible | FAILS (no web API on iOS) |
| SharedWorker supported | SURVIVES (iOS 16+) |
| WebCodecs video supported | SURVIVES (iOS 16.4+) |
| WebCodecs audio supported | SURVIVES (iOS 26+ only — older iOS needs fallback) |
| OPFS supported | SURVIVES (iOS 15.2+) |
| `<a download>` reliable for media | FAILS (use navigator.share) |
| File System Access pickers | FAILS (not on Safari) |
| FFmpeg.wasm usable | SURVIVES with caveats (cap memory ~256-512MB, single-thread only) |

**Net:** iOS is the constraint that shapes the design. Stream-to-OPFS, share-sheet saves, foreground-only downloads, no local library.

---

## 12. Media Engine Audit

| Claim | Verdict |
|---|---|
| Mediabunny + mp4box.js replace FFmpeg.wasm for v1 | SURVIVES (needs robustness test on real platform media) |
| WebCodecs for decode/encode | SURVIVES (progressive) |
| FFmpeg.wasm as escape hatch | SURVIVES (trigger conditions must be defined) |
| Multi-threaded FFmpeg.wasm | REJECT (needs SAB, unavailable on Pages) |
| FFmpeg.wasm in initial bundle | REJECT (load from Releases on demand) |
| WASM necessary for v1 | FAILS (pure TS suffices) |

**Escape-hatch trigger conditions (proposed):**
- Input container not supported by Mediabunny (e.g., obscure format).
- Codec not decodable by WebCodecs (e.g., ancient codec).
- Operation not supported by pure-TS path (e.g., burn-in subtitles, complex filter graph).
- User explicitly requests a format requiring re-encode (e.g., "convert to MP3").

**Net:** Pure-TS media processing is the v1 default. FFmpeg.wasm is a rare escape hatch.

---

## 13. WASM Audit

| Claim | Verdict |
|---|---|
| WASM not necessary for v1 | SURVIVES |
| Rust not necessary for v1 | SURVIVES |
| Component Model / WASI deferred | SURVIVES |
| Javy for sandboxed extractors | DEFER (use if we adopt a plugin system later) |
| Extism for plugins | REJECT for v1 (premature) |
| Pyodide to run yt-dlp in browser | REJECT (too heavy, native deps don't work) |
| WebContainers | REJECT (overkill, needs COOP/COEP) |
| Go → WASM | REJECT (large binaries, poor fit) |
| AssemblyScript | OPTIONAL (if we write custom WASM later, Rust is preferred) |
| Zig | OPTIONAL (niche) |

**Net:** Zero WASM in v1 default flow. FFmpeg.wasm (C/emscripten) is the only WASM, loaded on demand as escape hatch.

---

## 14. Component Model Audit

| Claim | Verdict |
|---|---|
| WASI 0.3 production-ready on server | SURVIVES (irrelevant to our browser app) |
| Component Model mature enough for v1 | FAILS (browser use is build-time abstraction via jco; adds complexity without value) |
| WIT / jco worth integrating | DEFER (revisit if we adopt cross-language WASM plugins) |
| Component Model solves a real v1 problem | FAILS (no problem identified) |

**Net:** Component Model is noise for v1. Track it; don't adopt it.

---

## 15. Language/Runtime Audit

| Claim | Verdict |
|---|---|
| TypeScript for frontend | SURVIVES (non-negotiable) |
| React | SURVIVES (challenged but defensible — contributor pool) |
| Vite | SURVIVES (clearly correct for Pages SPA) |
| Next.js overkill | SURVIVES |
| Node.js for remote service | SURVIVES (yt-dlp wrapper compatibility) |
| Bun for remote service | OPTIONAL (faster, native TS, but smaller ecosystem) |
| Deno for remote service | OPTIONAL (secure-by-default, smaller ecosystem) |
| Cloudflare Workers for remote service | EXPERIMENTAL (can't run yt-dlp; pure-JS extractors only; 128MB limit; free tier generous; needs experiment) |
| A second runtime is justified | FAILS (pick one for remote service; Node is safe default) |

**Net:** TS everywhere. Node.js for remote service (with Bun as an alternative). No second language.

---

## 16. Cobalt Audit

| Claim | Verdict |
|---|---|
| Cobalt is a useful reference architecture | SURVIVES |
| We can copy Cobalt's frontend | REJECT (CC-BY-NC-SA, NonCommercial) |
| We can fork Cobalt's API | SURVIVES (AGPL — our service would be AGPL, acceptable) |
| Cobalt's API is the right model | SURVIVES (POST /extract → media URLs or proxy) |
| Cobalt uses yt-dlp internally | UNCERTAIN (inferred pure-JS scrapers, unconfirmed) |
| We should depend on Cobalt | OPTIONAL (we can learn from it but build our own) |

**Net:** Study Cobalt's API shape. Don't copy its frontend. Building our own extractor service (thin per-platform JS scrapers + optional yt-dlp fallback) is preferable to forking Cobalt, to avoid AGPL entanglement unless we're willing to AGPL our service.

---

## 17. yt-dlp Audit

| Claim | Verdict |
|---|---|
| yt-dlp is the breadth backend (1000+ sites) | SURVIVES |
| yt-dlp is browser-runnable | FAILS (Python) |
| yt-dlp wrappers (youtube-dl-exec) work in Node | SURVIVES |
| yt-dlp is the right default for the remote service | SURVIVES (with thin per-platform scrapers for the initial targets, falling back to yt-dlp for breadth) |
| A pure-JS port of yt-dlp is feasible | FAILS (no such port exists; effort would be enormous) |
| Javy can run yt-dlp's extractors in WASM | EXPERIMENTAL (untested; yt-dlp's Python extractors may not transpile cleanly) |

**Net:** yt-dlp runs server-side (Node), invoked by the remote service for breadth. Initial targets use thin dedicated scrapers (faster, more maintainable than yt-dlp's generic extractors for these specific sites).

---

## 18. Remote Runtime Audit

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Node.js (VPS/container) | Universal, yt-dlp wrappers, full control | Ops burden, scaling | USE (safe default) |
| Bun (VPS/container) | Fast, native TS | Smaller ecosystem | OPTIONAL |
| Deno (VPS/container) | Secure, native TS | Smaller ecosystem | OPTIONAL |
| Cloudflare Workers | Edge, free tier, sub-ms cold start | 128MB, no yt-dlp, pure-JS only | EXPERIMENTAL |
| Serverless (Lambda/Functions) | Scales to zero | Cold start, harder state | OPTIONAL |
| Container (Fly.io/Railway/Render) | Full control, easy deploy | Cost, ops | USE (if not VPS) |
| VPS (Hetzner/DigitalOcean) | Cheap, full control | Ops, manual scaling | USE (cheapest) |

**Net:** v1 remote service: Node.js on a VPS or container (Fly.io/Railway). Cloudflare Workers as an experiment for the pure-JS extractor path (if it works, it eliminates yt-dlp dependency and gives edge performance).

---

## 19. Storage Audit

| Store | Use | Verdict |
|---|---|---|
| OPFS | Large-file staging (download → process → save) | USE |
| IndexedDB | Metadata, history, queue, settings | USE |
| Cache API | App-shell caching (SW) | USE |
| Service Worker cache | App-shell offline | USE (reduced scope) |
| SQLite WASM | Structured queries on history | REJECT for v1 (IndexedDB suffices) |
| Memory | Transient chunks during stream | USE (implicit) |

**Net:** Three stores: OPFS (files), IndexedDB (metadata), Cache API (app shell). No SQLite WASM in v1.

---

## 20. Worker/Runtime Audit

| Worker type | Use | Verdict |
|---|---|---|
| DedicatedWorker | Media processing, download streaming, WASM | USE (primary) |
| SharedWorker | Cross-tab coordination (queue, single download manager) | OPTIONAL (if multi-tab matters; otherwise DedicatedWorker per tab) |
| ServiceWorker | App-shell caching only | USE (reduced scope) |

**Net:** DedicatedWorker is the workhorse. SharedWorker optional. SW for app shell only.

---

## 21. CI/CD Audit

| Claim | Verdict |
|---|---|
| GitHub Actions for build/test/release/deploy | SURVIVES |
| Actions must not be download runtime | SURVIVES (R8) |
| Scheduled Actions reliable | FAILS (best-effort; idempotent + push-triggered) |
| GitHub Releases for large assets | SURVIVES (needs CORS test) |
| `pull_request_target` safe | SURVIVES with caveats (hardened Dec 2025; still avoid run: interpolation) |
| Pin third-party actions to SHA | SURVIVES |
| OIDC for cloud deploys | SURVIVES |

**Net:** Actions for CI/CD only. Scheduled workflows idempotent + push-triggered. Releases for FFmpeg.wasm.

---

## 22. Security Red Team

| Threat | Mitigation | Verdict |
|---|---|---|
| SSRF (remote fetches user URL) | Allowlist + private-IP reject + cloud-metadata block + redirect re-validation | SURVIVES |
| Open proxy abuse | Signed proxy URLs (HMAC, 60s TTL), per-IP rate limit, host allowlist, size cap | SURVIVES |
| Malicious media | Worker isolation, timeouts, size caps, prefer pure-TS parsers | SURVIVES |
| WASM supply chain | SRI on .wasm, pinned builds (SHA) | SURVIVES |
| Dependency supply chain | Lockfile, audit, pin, Dependabot | SURVIVES |
| Actions injection | Avoid run: interpolation of untrusted input, permissions: read default, pin actions | SURVIVES |
| CSP on Pages | `<meta>` CSP (document-level) | SURVIVES |
| COOP/COEP | Cannot set on Pages → no SAB (accepted) | SURVIVES (accepted limit) |
| Secrets | Zero on client; remote env only | SURVIVES |
| Content legality | No search/discovery, user-initiated, no hosting, personal-use framing | SURVIVES |
| Remote service DoS | Rate limit, size cap, timeout, queue | SURVIVES |

**Net:** Security model is defensible. The remote service is the attack surface; defense-in-depth with allowlists, signed URLs, rate limits, isolation.

---

## 23. Licensing Audit

| Claim | Verdict |
|---|---|
| MIT project license | SURVIVES |
| FFmpeg.wasm (GPL) isolated as optional plugin | SURVIVES (keeps core MIT) |
| Cobalt AGPL on API (if forked) | SURVIVES (service is AGPL; client stays MIT) |
| Cobalt CC-BY-NC-SA frontend | REJECT (don't copy; write our own) |
| H.264/H.265 patent risk for WASM encoder | SURVIVES (mitigate: prefer AV1/VP9/Opus; remux-only avoids encoding) |
| yt-dlp Unlicense | SURVIVES (reference/wrap freely) |
| All TS media libs permissive (BSD/ISC/MIT) | SURVIVES |

**Net:** MIT core. FFmpeg.wasm optional plugin. Don't touch Cobalt's frontend. Prefer royalty-free codecs for any encoding.

---

## 24. Performance Audit

| Aspect | Budget | Verdict |
|---|---|---|
| Initial bundle (gzipped) | <300KB | SURVIVES (React + Vite + app code; lazy-load rest) |
| WASM in initial bundle | 0KB | SURVIVES |
| Mediabunny lazy-load | ~50-100KB | SURVIVES |
| FFmpeg.wasm load (escape hatch) | ~25MB from Releases, on demand | SURVIVES (rare) |
| Worker startup | <50ms | SURVIVES |
| Stream-to-OPFS throughput | Limited by network, not browser | SURVIVES |
| iOS memory | Cap WASM ~256-512MB; never buffer full file | SURVIVES |
| Remote extraction latency | 1-5s typical | SURVIVES (acceptable) |
| Remote proxy overhead | +1 round-trip, +bandwidth | SURVIVES (only when needed) |

**Net:** Performance budgets are achievable. The main risk is iOS memory with FFmpeg.wasm — mitigate by capping and preferring pure-TS.

---

## 25. Large-File Audit

| Scenario | Handling | Verdict |
|---|---|---|
| 1GB+ video download | Stream fetch→OPFS, never buffer | SURVIVES |
| Resume after network drop | Track bytes in IDB, Range request (if CDN supports) | SURVIVES (UNCERTAIN per-CDN Range support) |
| Resume after URL expiry | Re-extract, resume or restart | SURVIVES (needs state machine — §7.1) |
| iOS tab backgrounded | Download stalls; resume on re-foreground | SURVIVES (accepted limit) |
| Chrome tab closed | Background Fetch API survives | SURVIVES (enhancement) |
| Disk full | OPFS write fails; surface error | SURVIVES |
| Malformed media | Parser fails in worker; surface error | SURVIVES |

**Net:** Large-file strategy works with explicit resume-on-expiry state machine.

---

## 26. Mobile Audit

| Platform | Key limits | Verdict |
|---|---|---|
| iOS Safari | 7-day eviction, no background, no FS pickers, unreliable `<a download>` | SURVIVES (design around: share-sheet, foreground-only, ephemeral storage) |
| iPadOS | Same as iOS Safari (desktop UA) | SURVIVES |
| Android Chrome | Background Fetch, OPFS, WebCodecs, FS Access | SURVIVES (enhancements available) |
| Firefox Android | WebCodecs incomplete | SURVIVES (fallback to passthrough/FFmpeg.wasm) |
| PWA (installed) | Same API limits as browser | SURVIVES (installable but no special powers) |

**Net:** iOS shapes the design. Android/Chrome is the easy target.

---

## 27. Zero-Server Feasibility

**Can the system work with ZERO server (pure GitHub Pages + browser)?**

- For platforms with permissive CDNs and CORS-enabled HTML (rare/nonexistent for TikTok/IG/FB): **YES**. The browser fetches HTML, scrapes embedded JSON, fetches media bytes directly, streams to OPFS, saves.
- For TikTok/IG/FB: **NO**. CORS blocks HTML fetch; Referer-gated CDNs block media fetch.
- For platforms where the user supplies a direct media URL (not a post URL): **YES**. If the CDN is CORS-permissive, the browser fetches and saves directly. This is a "URL downloader" mode that works zero-server.

**Verdict:** Zero-server is feasible for a **subset** of use cases (direct-URL download, permissive-CDN platforms). For the initial targets, a remote service is required. The architecture should support a zero-server degraded mode (direct-URL download) as a fallback if the remote service is down.

---

## 28. Minimum-Server Feasibility

**What is the absolute minimum the server must do?**

Minimum (for TikTok/IG/FB):
1. `POST /extract {url}` → `{media[], metadata, expiresAt, proxyRecommended}` (extraction + CORS headers on response)
2. `GET /proxy?u=<url>&sig=<hmac>` → proxies media bytes with Referer/cookies + CORS headers (only when direct browser fetch fails)

That's it. No transcoding, no storage, no user accounts, no database (extraction cache optional, in-memory).

**Expanded/worst-case server:**
- Residential-proxy egress (for IG/FB).
- yt-dlp subprocess (for breadth).
- Rate limiting, signed URLs, SSRF defense.
- Health monitoring, multi-instance, load balancing.
- Extraction-result cache (Redis, 30-60s TTL).

**Verdict:** The minimum is genuinely small (two endpoints). The operational complexity (residential proxies, yt-dlp, rate limiting) is where the real cost lives. "Minimum-server" is honest for the API surface; the implementation is non-trivial.

---

## 29. Plugin Architecture Verdict

**REJECT for v1.** (See §6.1.)

Use a simple `PlatformAdapter` interface:
```ts
interface PlatformAdapter {
  platformId: string;
  matchUrl(url: URL): boolean;
  extract(url: URL): Promise<MediaResult>; // calls remote
  // ... capabilities, display info
}
```
Adapters are static imports (or lazy dynamic imports) in the client. No runtime plugin loading, no WASM plugins, no registry engine. Revisit a full plugin system only when a demonstrated third-party-contributor use case appears AND the adapter interface proves insufficient.

---

## 30. Capability Engine Verdict

**REJECT for v1.** (See §6.2.)

Replace with direct feature detection:
```ts
const capabilities = {
  webCodecsVideo: typeof VideoDecoder !== 'undefined',
  webCodecsAudio: typeof AudioDecoder !== 'undefined',
  opfsSyncHandle: 'createSyncAccessHandle' in FileSystemSyncAccessHandle.prototype,
  backgroundFetch: 'BackgroundFetchManager' in window,
  shareFiles: 'canShare' in navigator && 'files' in navigator.share,
  // ...
};
```
One object, computed once. No engine, no registry.

---

## 31. Runtime Registry Verdict

**REJECT for v1.** (See §6.2.)

Replace with a function:
```ts
function selectExtractionStrategy(platform: PlatformAdapter): 'remote' | 'browser' {
  return platform.needsRemote ? 'remote' : 'browser';
}
```
No registry, no runtime swapping.

---

## 32. Extraction Registry Verdict

**REJECT for v1.** (See §6.2.)

Replace with a map:
```ts
const adapters: Record<string, PlatformAdapter> = {
  tiktok: TikTokAdapter,
  instagram: InstagramAdapter,
  facebook: FacebookAdapter,
  // ...
};
```
A plain object. No registry class.

---

## 33. Media Planner Verdict

**REJECT for v1.** (See §6.2.)

Replace with a function:
```ts
function planMediaProcessing(input: MediaInput): ProcessingPlan {
  if (input.container === 'mp4' && input.codec === 'h264' && input.targetFormat === 'mp4') {
    return { action: 'passthrough' };
  }
  if (Mediabunny.supports(input.container)) {
    return { action: 'remux', tool: 'mediabunny' };
  }
  return { action: 'escape-hatch', tool: 'ffmpeg.wasm' };
}
```
No planner class, no pipeline graph. A switch.

---

## 34. Normalized Media Model Verdict

**SURVIVES (minimal version).**

A normalized result shape is genuinely useful (decouples adapters from UI):
```ts
interface MediaResult {
  platform: string;
  originalUrl: string;
  media: Array<{
    url: string;
    type: 'video' | 'audio' | 'image';
    container?: string;
    codec?: string;
    quality?: string;
    width?: number;
    height?: number;
    durationMs?: number;
    expiresAt?: number;
    proxyRequired?: boolean;
  }>;
  metadata: {
    title?: string;
    author?: string;
    thumbnailUrl?: string;
    durationMs?: number;
  };
  extractedAt: number;
}
```
This is a data type, not an engine. Keep it.

---

## 35. Service Worker Verdict

**REDUCE to app-shell caching only.** (See §5.1.)

- SW registers, caches the app shell (HTML/JS/CSS) for offline UI.
- SW does NOT intercept media fetches (no gain; CORS still blocks; proxy handles it).
- SW does NOT do background sync (unreliable on iOS).
- SW is evicted on iOS after 7 days — accepted (user re-visits → re-register).

---

## 36. Missing Technologies

(Things Pass 1 didn't consider that red-team identifies as needed.)

1. **Extraction-result cache (server-side, in-memory or Redis, 30-60s TTL).** Multiple users may request the same viral URL within a minute; caching avoids re-extraction.
2. **Health-check endpoint on the remote service** (`GET /health`) for monitoring + client-side "service available" check.
3. **jsDelivr CDN** as a CORS-friendly mirror for FFmpeg.wasm if GitHub Releases CORS fails (UNCERTAIN). jsDelivr serves npm + GitHub releases with `Access-Control-Allow-Origin: *`.
4. **Sentry / client-side error reporting** for debugging in-production browser errors (with user opt-in / privacy controls).
5. **Service-worker update notification** — when the SW updates, prompt user to reload (avoid stale-asset bugs).

---

## 37. Technologies To Remove

| Technology | Reason |
|---|---|
| Next.js | Overkill for static Pages (Vite is right) |
| Multi-threaded FFmpeg.wasm | Needs SAB, unavailable on Pages |
| SharedArrayBuffer | Same — needs COOP/COEP |
| `coi-serviceworker` COOP/COEP shim | Flaky, not production-viable |
| Pyodide | Too heavy; yt-dlp native deps don't work in browser |
| WebContainers | Overkill; needs COOP/COEP |
| Go → WASM | Large binaries, poor fit |
| gallery-dl | Python, not browser-relevant |
| Cobalt web frontend | CC-BY-NC-SA (NonCommercial) |
| Component Model / WIT / jco (for v1) | Build-time abstraction, no v1 value |
| Extism (for v1) | Premature plugin system |
| SQLite WASM (for v1) | IndexedDB suffices |
| Capability Engine / Runtime Registry / Extraction Registry / Media Planner | Over-abstracted; use functions |
| Plugin architecture (for v1) | Premature; use PlatformAdapter interface |
| Service Worker media interception | No gain; CORS still blocks |

---

## 38. Technologies To Defer

| Technology | Defer until |
|---|---|
| Rust + WASM | A hot path is profiled and needs it |
| Component Model / WASI 0.3 | Cross-language WASM plugins are needed |
| Javy / QuickJS-in-WASM | A sandboxed third-party-extractor system is needed |
| Extism | A plugin system is justified |
| SQLite WASM (wa-sqlite) | History needs full-text search / complex queries |
| Cloudflare Workers remote runtime | The pure-JS extractor experiment succeeds and edge performance matters |
| Bun / Deno for remote service | Node.js proves insufficient (unlikely) |
| Multi-threaded WASM | We migrate hosting to a platform that allows COOP/COEP (Cloudflare Pages/Netlify) |
| Background Fetch API | Already an enhancement, not deferred |
| Full plugin system | A demonstrated third-party-contributor use case appears |

---

## 39. Technologies Requiring Experiments

| Technology | Experiment | Success criteria |
|---|---|---|
| GitHub Releases CORS | Browser `fetch()` of a release asset from `*.github.io` origin; check `Access-Control-Allow-Origin` | CORS header present; fetch succeeds |
| Mediabunny robustness | Feed it real TikTok/IG/FB media (MP4, fragmented MP4, WebM); check parse/mux success | >90% success on real platform media |
| Cloudflare Workers extractor | Port one platform scraper (TikTok) to pure JS; deploy to CF Workers; test extraction success rate | >80% success; within 128MB/50ms CPU |
| TikTok CDN Range support | Start a download, abort at 50%, resume with `Range: bytes=N-` | Range honored; resume succeeds |
| iOS Safari memory cap for FFmpeg.wasm | Load FFmpeg.wasm with capped memory; process a 500MB file on iOS | No tab crash; processing completes |
| Residential-proxy extraction (IG) | Run IG extractor through BrightData/Oxylabs; measure success rate vs datacenter | >70% success with residential; <20% with datacenter |
| `navigator.share({files})` for large files | Share a 500MB video on iOS; check it saves to Files | Save succeeds within 30s |
| Signed-URL TTL | Extract a TikTok URL; attempt fetch at 0/5/10/30/60 min; find expiry | TTL measured; resume-on-expiry flow validated |

---

## 40. New Risks

(Identified by red-team, beyond Pass 1's list.)

1. **GitHub Releases CORS failure** (UNCERTAIN). If `release-assets.githubusercontent.com` doesn't send CORS headers, the FFmpeg.wasm escape hatch breaks. Mitigation: jsDelivr mirror.
2. **Residential-proxy cost spiral.** If IG/FB extraction volume grows, proxy costs (~$1-15/GB) could exceed budget. Mitigation: cache extraction results; avoid proxying media bytes through residential proxies.
3. **Platform API/HTML changes break extractors.** TikTok/IG/FB change their page structure frequently. Each breakage requires a scraper update. Mitigation: monitor extractor success rate; alert on drop; fast-deploy scraper fixes.
4. **Legal takedown of our remote service.** Unlike a static GitHub repo, a running service that downloads from TikTok/IG/FB is a clearer target. Mitigation: no hosting of media, no search, user-initiated, clear personal-use framing, consider jurisdiction.
5. **FFmpeg.wasm escape-hatch UX.** If a user hits the escape hatch, they wait for a 25MB download + 1-3s init, then slow processing. This could be perceived as broken. Mitigation: clear progress UI; warn before triggering.
6. **iOS WebCodecs audio gap.** Users on iOS < 26 can't use WebCodecs audio. Mitigation: detect and fallback to passthrough (no audio processing) or FFmpeg.wasm.
7. **Multi-tab race on SharedWorker.** If two tabs share a download queue, concurrent state mutation is a risk. Mitigation: use a single SharedWorker as the download manager with a message-passing protocol; or skip SharedWorker and run per-tab DedicatedWorkers (simpler, no sharing).
8. **Remote service as SPOF.** If the remote is down, the whole app is broken for TikTok/IG/FB. Mitigation: zero-server degraded mode (direct-URL download); multiple remote instances; health check + graceful degradation in UI.

---

## 41. New Architecture Requirements

(Emerging from red-team, to feed Pass 3.)

1. **Reframe "browser-first" as "browser-executed, remote-assisted."** Honest framing.
2. **Capability-based extraction selection** (not "try browser, fall back to remote"). Each platform adapter declares its needs upfront.
3. **Explicit resume-on-expiry state machine** for downloads (track bytes, detect expiry, re-extract, resume/restart, cap attempts).
4. **Zero-server degraded mode** — if the remote is down, the app still works for direct-URL downloads and permissive-CDN platforms.
5. **Simple `PlatformAdapter` interface** (no plugin engine).
6. **Direct functions for capability detection, runtime selection, media planning** (no engines/registries/planners).
7. **Service Worker for app-shell caching only.**
8. **Three stores: OPFS (files), IndexedDB (metadata), Cache API (app shell).** No SQLite WASM in v1.
9. **FFmpeg.wasm escape-hatch trigger conditions defined** (unsupported container/codec/operation, or user-requested format).
10. **Remote service API: `/extract` + `/proxy` + `/health`.** Stateless, allowlisted, SSRF-defended, rate-limited, signed-URL proxy.
11. **Residential-proxy egress for IG/FB** (cost-investigated, provider-selected).
12. **Extraction-result cache** (server-side, 30-60s TTL).
13. **jsDelivr as CORS-friendly mirror** for FFmpeg.wasm if GitHub Releases CORS fails (experiment).
14. **MIT core license; FFmpeg.wasm optional plugin; AGPL on remote service only if we fork Cobalt.**
15. **No second language.** TypeScript everywhere (client + remote service).

---

## PROVISIONAL DECISION MATRIX

| Technology / Decision | PASS 1 | PASS 2 | Evidence | Reason | Confidence |
|---|---|---|---|---|---|
| **GitHub Pages (app shell)** | candidate | USE | cluster A | Right host for static SPA; COOP/COEP limit accepted | HIGH |
| **GitHub Releases (large assets)** | candidate | USE | cluster A, F | 2GB/asset, uncapped bandwidth; CORS needs test | MEDIUM |
| **GitHub Actions (CI/CD)** | candidate | USE | cluster A, F | Build/test/release/deploy; not runtime | HIGH |
| **.nojekyll** | candidate | USE | cluster A | Required for correct .wasm MIME | HIGH |
| **Vite** | candidate | USE | cluster C | Fast, ES-module, content-hashed output | HIGH |
| **React** | candidate | USE | cluster C | Contributor pool, ecosystem; bundle acceptable | MEDIUM |
| **TypeScript** | candidate | USE | cluster C | Strict typing, universal | HIGH |
| **Next.js** | candidate | REJECT | cluster C | Overkill for static Pages | HIGH |
| **DedicatedWorker** | candidate | USE | cluster B, E | Primary off-main-thread unit | HIGH |
| **SharedWorker** | candidate | OPTIONAL | cluster B, E | Cross-tab coordination; iOS 16+; optional | MEDIUM |
| **ServiceWorker (app shell only)** | candidate | USE | cluster B, E | Offline app shell; reduced scope | HIGH |
| **Streams API** | candidate | USE | cluster B | Stream fetch→OPFS | HIGH |
| **Fetch / CORS** | candidate | USE | cluster B, D | Core download mechanism | HIGH |
| **OPFS** | candidate | USE | cluster B, E | Large-file staging | HIGH |
| **IndexedDB** | candidate | USE | cluster B, E | Metadata/history/queue | HIGH |
| **Cache API** | candidate | USE | cluster B, E | App-shell caching | HIGH |
| **WebCodecs (progressive)** | candidate | USE | cluster B, C, E | Frame decode/encode; iOS audio 26+ caveat | HIGH |
| **navigator.share** | candidate | USE | cluster E | iOS saves | HIGH |
| **navigator.storage.persist()** | candidate | OPTIONAL | cluster E | Requested, not guaranteed on iOS | MEDIUM |
| **Background Fetch API** | candidate | OPTIONAL | cluster E | Chrome enhancement; not on iOS | HIGH |
| **File System Access API** | candidate | OPTIONAL | cluster B | Chrome-only; fallback for desktop save | MEDIUM |
| **Mediabunny** | candidate | USE | cluster C | Pure-TS media processing | MEDIUM (needs robustness test) |
| **mp4box.js** | candidate | USE | cluster C | ISOBMFF inspection/fragmentation | HIGH |
| **mp4-muxer / webm-muxer** | candidate | OPTIONAL | cluster C | If Mediabunny insufficient for a case | MEDIUM |
| **FFmpeg.wasm (escape hatch)** | candidate | USE-LATER | cluster C, F | Load on demand from Releases; GPL isolated | MEDIUM |
| **Multi-threaded FFmpeg.wasm** | candidate | REJECT | cluster A, C, F | Needs SAB, unavailable on Pages | HIGH |
| **SharedArrayBuffer** | candidate | REJECT | cluster A, B, F | Needs COOP/COEP, unavailable on Pages | HIGH |
| **coi-serviceworker shim** | candidate | REJECT | cluster F | Flaky, not production-viable | HIGH |
| **WASM (general, v1)** | candidate | REJECT | cluster C | Pure TS suffices for v1 | HIGH |
| **Rust** | candidate | USE-LATER | cluster C | Justified only if a hot path needs WASM | HIGH |
| **Zig** | candidate | OPTIONAL | cluster C | Niche; if custom WASM needed | LOW |
| **AssemblyScript** | candidate | OPTIONAL | cluster C | If custom WASM needed; Rust preferred | LOW |
| **Go → WASM** | candidate | REJECT | cluster C | Large binaries, poor fit | HIGH |
| **Component Model / WIT / jco** | candidate | USE-LATER | cluster C | Build-time abstraction; no v1 value | HIGH |
| **WASI 0.2/0.3** | candidate | USE-LATER | cluster C | Server-ready; browser via jco; defer | HIGH |
| **Javy** | candidate | USE-LATER | cluster C | If sandboxed extractors needed | MEDIUM |
| **QuickJS** | candidate | USE-LATER | cluster C | Alternative JS-in-WASM | MEDIUM |
| **Extism** | candidate | USE-LATER | cluster C | If plugin system justified | MEDIUM |
| **Pyodide** | candidate | REJECT | cluster C | Too heavy; yt-dlp deps don't work | HIGH |
| **WebContainers** | candidate | REJECT | cluster C | Overkill; needs COOP/COEP | HIGH |
| **SQLite WASM (wa-sqlite)** | candidate | USE-LATER | cluster E | IndexedDB suffices for v1 | HIGH |
| **yt-dlp (remote)** | candidate | USE | cluster D, F | Breadth backend; server-side | HIGH |
| **Cobalt (reference)** | candidate | OPTIONAL | cluster D, F | Study; don't copy frontend | HIGH |
| **gallery-dl** | candidate | REJECT | cluster D | Python, not browser-relevant | HIGH |
| **Remote extractor service** | candidate | USE | cluster D | Unavoidable for TikTok/IG/FB | HIGH |
| **Remote media proxy** | candidate | USE | cluster D | Referer/cookies required | HIGH |
| **Node.js (remote)** | candidate | USE | cluster C, D | yt-dlp wrapper compatibility | HIGH |
| **Bun (remote)** | candidate | OPTIONAL | cluster C | Faster, native TS; smaller ecosystem | MEDIUM |
| **Deno (remote)** | candidate | OPTIONAL | cluster C | Secure; smaller ecosystem | MEDIUM |
| **Cloudflare Workers (remote)** | candidate | EXPERIMENTAL | cluster C | Pure-JS only; 128MB; free tier; needs test | MEDIUM |
| **Serverless (Lambda/Functions)** | candidate | OPTIONAL | cluster C | Cold start; scales to zero | MEDIUM |
| **Container/VPS** | candidate | USE | cluster C | Full control; yt-dlp | HIGH |
| **Residential proxy** | identified | USE | cluster D | IG/FB datacenter-IP block | MEDIUM (cost) |
| **Plugin architecture** | candidate | REJECT (v1) | red-team §6.1 | Premature; use PlatformAdapter | HIGH |
| **Capability Engine** | candidate | REJECT (v1) | red-team §6.2 | Use direct feature detection | HIGH |
| **Runtime Registry** | candidate | REJECT (v1) | red-team §6.2 | Use a function | HIGH |
| **Extraction Registry** | candidate | REJECT (v1) | red-team §6.2 | Use a map | HIGH |
| **Media Planner** | candidate | REJECT (v1) | red-team §6.2 | Use a function | HIGH |
| **Normalized Media Model** | candidate | USE | red-team §6.2 | Useful data type (not an engine) | HIGH |
| **MIT project license** | candidate | USE | cluster F | Permissive, maximizes adoption | HIGH |
| **Extraction-result cache (server)** | identified | USE | red-team §36 | Avoid re-extraction of viral URLs | HIGH |
| **Health-check endpoint** | identified | USE | red-team §36 | Monitoring + client degraded-mode | HIGH |
| **jsDelivr CDN mirror** | identified | OPTIONAL | red-team §36 | If GH Releases CORS fails | MEDIUM |
| **Client-side error reporting** | identified | OPTIONAL | red-team §36 | Debugging; privacy-conscious | MEDIUM |

---

## EXPERIMENT PLAN

### Experiment 1: GitHub Releases CORS
- **Question:** Does `release-assets.githubusercontent.com` send `Access-Control-Allow-Origin` allowing browser fetch from `*.github.io`?
- **Hypothesis:** It does (GitHub serves releases with permissive CORS for browser fetch).
- **Setup:** Upload a small test asset to a GitHub Release. From a `*.github.io` page, `fetch()` the asset URL. Inspect response headers.
- **Measurement:** Presence of `Access-Control-Allow-Origin`; fetch success/failure.
- **Success criteria:** CORS header present; fetch succeeds.
- **Failure criteria:** No CORS header; fetch blocked. → Mitigation: use jsDelivr mirror (`https://cdn.jsdelivr.net/gh/...`).
- **Architectural consequence:** If fails, FFmpeg.wasm distribution shifts to jsDelivr; affects asset URL configuration.

### Experiment 2: Mediabunny robustness on real platform media
- **Question:** Does Mediabunny parse/mux real TikTok/IG/FB media (MP4, fragmented MP4, WebM) reliably?
- **Hypothesis:** >90% success rate on real platform media.
- **Setup:** Collect 20 sample media files (10 TikTok MP4, 5 IG MP4, 5 FB MP4). Run Mediabunny's parse + remux on each.
- **Measurement:** Success/failure per file; error type on failure.
- **Success criteria:** ≥90% success.
- **Failure criteria:** <90% → investigate failures; may need mp4box.js fallback or FFmpeg.wasm escape hatch for specific cases.
- **Architectural consequence:** If Mediabunny is unreliable, the escape-hatch trigger fires more often, increasing FFmpeg.wasm load.

### Experiment 3: Cloudflare Workers extractor
- **Question:** Can a pure-JS TikTok scraper run within CF Workers' 128MB / 50ms CPU limits?
- **Hypothesis:** Yes, for a single URL extraction (HTML fetch + parse + JSON extract + return).
- **Setup:** Port the TikTok scraper to pure JS. Deploy to CF Workers. Test 50 TikTok URLs.
- **Measurement:** Success rate; CPU time; memory; latency.
- **Success criteria:** ≥80% success; <50ms CPU; <128MB memory.
- **Failure criteria:** Fails → Node.js + yt-dlp on VPS remains the path.
- **Architectural consequence:** If succeeds, remote service can be CF Workers (edge, free tier, no yt-dlp dependency).

### Experiment 4: TikTok CDN Range-request support
- **Question:** Does the TikTok CDN honor `Range:` on signed video URLs?
- **Hypothesis:** Yes for progressive MP4s; uncertain for signed URLs.
- **Setup:** Start a TikTok video download, abort at 50%, attempt resume with `Range: bytes=N-`.
- **Measurement:** 206 Partial Content response vs 200 full / 403.
- **Success criteria:** 206; resume succeeds.
- **Failure criteria:** No Range support → resume restarts from zero.
- **Architectural consequence:** Affects resume-on-expiry state machine (restart vs resume).

### Experiment 5: iOS Safari FFmpeg.wasm memory cap
- **Question:** What's the practical WASM linear-memory cap on iOS Safari before tab crash?
- **Hypothesis:** ~256-512MB is safe; 2GB crashes.
- **Setup:** Load FFmpeg.wasm with `MAXIMUM_MEMORY` set to 256MB / 512MB / 1GB / 2GB on iOS Safari. Process a 500MB file.
- **Measurement:** Tab crash / success per cap.
- **Success criteria:** 256MB or 512MB succeeds; identify the threshold.
- **Failure criteria:** Even 256MB crashes → FFmpeg.wasm unusable on iOS; pure-TS path must cover all iOS cases.
- **Architectural consequence:** Sets the iOS WASM memory budget; may disable escape hatch on iOS.

### Experiment 6: Residential-proxy IG extraction
- **Question:** Does IG extraction succeed with a residential proxy and fail with datacenter?
- **Hypothesis:** Residential >70% success; datacenter <20%.
- **Setup:** Run IG extractor via BrightData residential vs a digitalocean datacenter IP. Test 30 IG URLs.
- **Measurement:** Success rate per egress.
- **Success criteria:** Residential ≥70%; datacenter ≤20%.
- **Failure criteria:** Residential <70% → IG extraction is unreliable; reconsider IG as a v1 target.
- **Architectural consequence:** Determines whether IG is viable in v1 and the proxy cost.

### Experiment 7: Signed-URL TTL measurement
- **Question:** How long are TikTok/IG/FB CDN URLs valid?
- **Hypothesis:** TikTok ~1-2 hours; IG ~5-15 minutes; FB ~1 hour.
- **Setup:** Extract URLs; attempt fetch at 0/5/10/30/60/120 min.
- **Measurement:** Time of first 403/failure.
- **Success criteria:** Measure per-platform TTL.
- **Failure criteria:** N/A (measurement).
- **Architectural consequence:** Sets the resume-on-expiry re-extraction threshold per platform.

### Experiment 8: navigator.share large-file on iOS
- **Question:** Can `navigator.share({files: [500MB video]})` save to Files on iOS without failure?
- **Hypothesis:** Yes, within ~30s.
- **Setup:** On iOS Safari, share a 500MB MP4 from OPFS.
- **Measurement:** Save success/failure; time.
- **Success criteria:** Save succeeds.
- **Failure criteria:** Fails → may need to chunk or use `<a download>` (unreliable) or advise smaller files.
- **Architectural consequence:** Affects iOS large-file UX.

---

## END OF PASS 2 REPORT

**Next:** Pass 3 (Formal Technology + Architecture Decision Ledger) will convert the surviving evidence + this provisional matrix into the final, traceable decision system.
