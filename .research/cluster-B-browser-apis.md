# Cluster B — Browser APIs (2026 Support Matrix)

**Pass:** 1 (discovery)
**Task ID:** 1-B
**Date compiled:** 2026-08-16
**Scope:** Browser-native API support for an open-source, GitHub-Pages-hosted media downloader that processes media client-side. Target browsers: Chrome, Edge, Firefox (desktop + Android), Safari (macOS), Safari iOS / iPadOS WebView.

All claims below are sourced from MDN, caniuse, WebKit blog, Chrome Developers, GitHub issue threads, or named 2025/2026 secondary sources. Where a claim is uncertain or only partially verified, it is marked `UNCERTAIN`.

---

## 1. WebCodecs (`VideoDecoder`, `AudioDecoder`, `VideoEncoder`, `AudioEncoder`, `ImageDecoder`, `EncodedVideoChunk`)

### Ship status (stable, all-access)
- **Chrome / Edge**: stable since Chrome 94 (Sept 2021). All four codecs interfaces + `ImageDecoder` shipped. [source: https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder]
- **Firefox desktop**: stable since Firefox 130 (Sept 2024). [source: https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder] [source: https://www.testmuai.com/learning-hub/webcodecs-browser-support]
- **Firefox Android**: NOT fully implemented as of 2026 — "Firefox has not fully implemented the WebCodecs API on Android". [source: https://webcodecsfundamentals.org/datasets/codec-analysis-2026] `UNCERTAIN` on exact coverage (decode vs encode).
- **Safari desktop / iOS Safari**: shipped `VideoDecoder`/`VideoEncoder`/`ImageDecoder` earlier (partial — `UNCERTAIN` exact version, possibly Safari 16.4 beta / 17.x). `AudioEncoder` and `AudioDecoder` were added in **Safari 26.0, released September 15, 2025** (iOS 26, iPadOS 26, macOS 26). "Full support" of WebCodecs in Safari therefore requires iOS 26+ / macOS 26+. [source: https://developer.apple.com/documentation/safari-release-notes/safari-26-release-notes] [source: https://webkit.org/blog/17333/webkit-features-in-safari-26-0] [source: https://softvelum.com/2025/09/nimio-safari-ios-macos-26]
- **Samsung Internet 17+**, **Opera 80+**: stable. [source: https://www.testmuai.com/learning-hub/webcodecs-browser-support]

### Codec coverage (decoding / encoding, varies per browser)
- **H.264 (AVC)**: decode + encode in Chrome, Firefox, Safari (hardware-accelerated via OS). Universally available. [source: https://www.w3.org/TR/webcodecs-codec-registry]
- **H.265 (HEVC)**: Chrome added decode support on supported hardware (Chrome 107+, gated by OS codec). Safari supports decode on macOS/iOS where the OS supports HEVC. Firefox does NOT support HEVC decode in WebCodecs (Mozilla refuses to ship patented codec; relies on OS support on Windows — `UNCERTAIN` for Firefox behavior). [source: https://www.w3.org/TR/webcodecs-codec-registry] [source: https://webcodecsfundamentals.org/datasets/codec-analysis-2026]
- **VP9**: decode in Chrome, Firefox, Edge, Safari (Safari added VP9 in macOS Big Sur / iOS 14). Encode in Chrome, Firefox, Edge. [source: https://antmedia.io/video-codecs-streaming-guide] [source: https://www.w3.org/TR/webcodecs-codec-registry]
- **AV1**: decode in Chrome 70+ (with `av1` flag, then default), Firefox 65+ (desktop), Safari 16.4 beta added AV1 hardware decode on supported hardware. Encode via WebCodecs in Chrome/Edge (software + hardware where available). [source: https://www.reddit.com/r/AV1/comments/1147onr/release_notes_safari_164_beta_adds_av1_codec] [source: https://www.w3.org/TR/webcodecs-codec-registry] AV1 encoder availability is hardware-gated; "85% of browsers support AV1 encoding, 90% support AV1 decoding" per real-world dataset of ~200k sessions. [source: https://www.reddit.com/r/AV1/comments/1qlvxn3/85_of_browsers_support_av1_encoding_90_support]
- **AAC**: decode everywhere; encode in Chrome (with OS support). [source: https://www.w3.org/TR/webcodecs-codec-registry]
- **Opus**: decode + encode everywhere (Chrome, Firefox, Safari 17+ with hardware/software). [source: https://www.w3.org/TR/webcodecs-codec-registry]
- **MP3**: decode everywhere (Chrome, Firefox, Safari). No MP3 encoding in WebCodecs (not in registry). [source: https://www.w3.org/TR/webcodecs-codec-registry]
- **FLAC, ALAC**: decode in some browsers. `UNCERTAIN` per-browser.
- **AVIF / WebP / JPEG / PNG via `ImageDecoder`**: Chrome, Edge, Safari (partial). Firefox `UNCERTAIN`.

### Demux / remux without re-encoding
WebCodecs does NOT include a container demuxer/muxer. It operates on raw encoded chunks (`EncodedVideoChunk` / `EncodedAudioChunk`). To demux MP4/WebM/MKV the app needs an in-browser muxer (e.g., `mp4box.js`, `libavcodec` via WASM, or `webm-muxer` / `mp4-muxer` NPM packages). Demux-then-remux (no re-encoding) is fully feasible: feed `EncodedVideoChunk`s from the demuxer straight into the muxer with no `VideoDecoder` call. [source: https://www.w3.org/TR/webcodecs-codec-registry]

### Throughput
Real-time hardware-accelerated decode on modern Chrome/Safari typically handles 4K H.264 / HEVC at 60fps. Software paths in Firefox are slower. `UNCERTAIN` exact throughput for AV1 encode across devices.

---

## 2. OPFS (Origin Private File System)

### Support
- All modern browsers (Chrome 102+, Edge, Firefox 111+, Safari 15.2+) support OPFS. Safari shipped in **Safari 15.2 / iOS 15.2** (Dec 2021). [source: https://webkit.org/blog/12257/the-file-system-access-api-with-origin-private-file-system] [source: https://github.com/mdn/content/issues/37004] [source: https://rxdb.info/rx-storage-opfs.html]
- Firefox supports OPFS from Firefox 111 (March 2023). [source: https://github.com/mdn/content/issues/37004]

### Sync vs async access handles
- `FileSystemFileHandle.createWritable()` — async, stream-style `FileSystemWritableFileStream`. Available on main thread + workers. [source: https://web.dev/articles/origin-private-file-system]
- `FileSystemFileHandle.createSyncAccessHandle()` — **synchronous**, returns `FileSystemSyncAccessHandle` (with `read()`/`write()`/`flush()`/`getSize()`/`truncate()`). **Only available in Web Workers (DedicatedWorker and SharedWorker) — NOT exposed on the main thread.** [source: https://rxdb.info/rx-storage-opfs.html] [source: https://zenn.dev/hori/articles/3abc10be8d9ca0?locale=en] [source: https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle/createSyncAccessHandle]

### Worker access
- Works in **DedicatedWorker** and **SharedWorker**. SharedWorker + OPFS pattern is used by libraries like `wa-sqlite` to share a DB connection across tabs. [source: https://github.com/rhashimoto/wa-sqlite/discussions/81]

### Persistence & quota
- OPFS shares the origin-wide storage quota (with IndexedDB, Cache API). On Chrome this is up to ~60% of free disk; on Firefox similar; on Safari iOS historically ~1 GB and aggressively evicted. [source: https://rxdb.info/articles/indexeddb-max-storage-limit.html]
- Safari "has a stricter strategy and removes OPFS data and local storage more aggressively, for example, if a user hasn't visited the site recently". [source: https://news.ycombinator.com/item?id=42137790]

### Performance
- Sync access handles in workers are the highest-throughput browser storage option — commonly used by SQLite WASM, `wa-sqlite`, and `sql.js` for in-browser database performance. Async `createWritable()` streams large files efficiently. [source: https://blog.tomayac.com/2025/03/08/setting-coop-coep-headers-on-static-hosting-like-github-pages] [source: https://modernwebweekly.substack.com/p/file-system-access-and-persistent]

### Streaming large file writes
- `createWritable()` returns a stream supporting `seek()`/`truncate()`/`write()` — can stream from a `ReadableStream` via `pipeTo()`. Suitable for multi-GB files (subject to quota). [source: https://web.dev/articles/origin-private-file-system]

---

## 3. Web Workers / Shared Workers / Service Workers

### Web Workers (DedicatedWorker)
- Universal support; no concerns. Worker code can use OPFS sync handles, fetch, IndexedDB, Streams, WebCodecs (Chrome, Firefox, Safari 26+).

### SharedWorker
- **Safari**: was supported in Safari 5/6, **removed in Safari 6.1**, **re-added in Safari 16** (April 2022, shipping in macOS 13 / iOS 16). "Safari now fully supports SharedWorkers." [source: https://itnext.io/safari-now-fully-supports-sharedworkers-534733b56b4c] [source: https://www.testmuai.com/learning-hub/sharedworker-browser-support]
- **iOS Safari 16+** ships SharedWorker (per the article above). `UNCERTAIN` whether all PWA/Home-Screen contexts honor it.
- **Chrome / Edge / Firefox desktop**: stable for years.
- **SharedWorker on Firefox Android / Chrome Android**: `UNCERTAIN` — historically disabled on Android browsers. Treat mobile SharedWorker support as `UNCERTAIN`.

### Service Worker
- Universal in Chrome, Edge, Firefox, Safari (Safari 11.1 desktop, iOS 11.3 — March 2018). [source: https://medium.com/@firt/pwas-are-coming-to-ios-11-3-cupertino-we-have-a-problem-2ff49fd7d6ea]
- **Lifetime on iOS Safari**: WebKit removes "unused service worker registrations after a period of a few weeks" (the much-cited "7-day purge" applies to **non-PWA data** like IndexedDB/Cache API for sites added to home screen vs not). For non-installed sites, after 7 days of no use the SW registration and its caches may be purged. [source: https://medium.com/@firt/pwas-are-coming-to-ios-11-3-cupertino-we-have-a-problem-2ff49fd7d6ea] [source: https://webkit.org/blog/14403/updates-to-storage-policy] [source: https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide]
- **Service Worker can intercept fetch** — yes, this is its core function. The SW `fetch` event handler can rewrite, proxy, or fabricate responses for any request the page makes within scope. This is the canonical mechanism for media proxying / header-patching (see §4 `coi-serviceworker`). [source: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers]
- **Background Sync API**: supported in Chrome/Edge. **NOT supported in Safari/Firefox** (Safari explicitly does not implement background sync). [source: https://github.com/GoogleChrome/workbox/issues/2516] [source: https://stackoverflow.com/questions/64677645/how-to-sync-my-saved-data-on-apple-devices-with-service-worker]
- **Periodic Background Sync API**: Chrome-only (Chrome 77+). Not in Firefox or Safari (any platform). [source: https://developer.mozilla.org/en-US/docs/Web/API/Web_Periodic_Background_Synchronization_API] [source: https://developer.chrome.com/docs/capabilities/periodic-background-sync] [source: https://caniuse.com/#search=background%20sync]
- **Does iOS Safari SW survive tab close?**: Yes, while the SW registration is alive (and the device is online). It will be terminated within ~30 seconds of inactivity and may be purged entirely after 7 days of no Safari visit. [source: https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide]

---

## 4. Cross-origin isolation (COOP / COEP / CORP) — **CRITICAL for GitHub Pages**

### Requirement
- `SharedArrayBuffer` (and therefore multi-threaded WASM, e.g. `pthreads` in Emscripten) requires the document to be **cross-origin isolated**:
  - `Cross-Origin-Opener-Policy: same-origin`
  - `Cross-Origin-Embedder-Policy: require-corp` (or `credentialless`)
- Without these headers, `self.crossOriginIsolated === false` and SAB is unavailable in the document (and WASM threads cannot start). [source: https://stackoverflow.com/questions/68609682/is-there-any-way-to-use-sharedarraybuffer-on-github-pages] [source: https://blog.tomayac.com/2025/03/08/setting-coop-coep-headers-on-static-hosting-like-github-pages]

### GitHub Pages does NOT allow custom HTTP headers
- GitHub Pages (and most static hosts like raw nginx-without-config, Netlify without config, etc.) **does not allow setting HTTP response headers**. This is a long-standing open request. [source: https://github.com/orgs/community/discussions/13309]
- Therefore a plain GitHub Pages deployment **cannot** set COOP/COEP at the HTTP layer, and SAB / WASM threads **will not work** by default.

### Workaround: `coi-serviceworker`
- A small SW script (`coi-serviceworker.js`) registers itself at the page root, intercepts every fetch, and rewrites the response with `COOP: same-origin` and `COEP: require-corp` headers. After a one-time reload, the document becomes cross-origin isolated and SAB/WASM-threads become available. [source: https://github.com/gzuidhof/coi-serviceworker] [source: https://docs.wasmer.io/sdk/wasmer-js/how-to/coop-coep-headers] [source: https://github.com/jupyterlite/jupyterlite/issues/1409]
- **Caveats**:
  - First page load requires a reload (the SW must take control before COEP can be enforced).
  - SW scope must cover the whole site; the `coi-serviceworker.js` file must be served from the root path.
  - COEP `require-corp` breaks all cross-origin subresources (fonts, images, scripts, iframes) unless they are loaded with `crossorigin` AND the origin returns `CORP: cross-origin` (or the resource is loaded via `credentialless`). For a media downloader this means: any cross-origin media fetched must either be no-CORS (then CORP rules still apply to the result if cached) or the app must use `Cross-Origin-Embedder-Policy: credentialless` (Chrome 96+, Firefox 110+, Safari 16.4+). [source: https://blog.tomayac.com/2025/03/08/setting-coop-coep-headers-on-static-hosting-like-github-pages] `UNCERTAIN` for full Safari behavior with `credentialless`.

### Impact on the project
- For a GitHub Pages production deployment, the team must either:
  1. Accept single-threaded WASM (no `pthreads`, no SAB) — meaning FFmpeg/libav built without threads. Performance ceiling lowered.
  2. Adopt `coi-serviceworker` and audit every external asset for CORP/`credentialless`.
  3. Use a non-GitHub-Pages host that supports HTTP headers (Cloudflare Pages, Netlify with `_headers`, Vercel with `vercel.json`). For an open-source GitHub-hosted project, Cloudflare Pages is the typical alternative.

---

## 5. Streams API (`ReadableStream`, `WritableStream`, `TransformStream`, `pipeThrough`, `pipeTo`, byte streams, async iteration, `tee()`)

### Support
- All major browsers (Chrome, Edge, Firefox, Safari 10.1+ desktop, Safari 10.3+ iOS) ship `ReadableStream`. [source: https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream]
- `pipeThrough()` / `pipeTo()` shipped in Safari 10.1 (desktop) and Safari 10.3 (iOS) — now universally stable. [source: https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream]
- Byte streams (`type: "bytes"` constructor option, `byobRequest`): Chrome, Firefox, Safari 16+. `UNCERTAIN` exact Safari iOS version for full byte-stream controller parity.
- Async iteration (`for await (const chunk of stream)`): Chrome, Firefox, Safari 16.4+. [source: https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream]
- `tee()`: universally supported.
- `TransformStream` constructor: universally supported (Chrome 80+, Firefox 102+, Safari 14.1+). [source: https://streams.spec.whatwg.org] [source: https://developer.mozilla.org/en-US/docs/Web/API/Streams_API]

### Streaming a fetch body to OPFS
- Standard pattern: `await response.body.pipeTo(fileHandle.createWritable())`. Works in Chrome, Edge, Firefox, Safari 15.2+. [source: https://web.dev/articles/origin-private-file-system]

### Piping across workers
- Streams are transferable across workers via `postMessage` with transfer list (`structuredClone` semantics). Useful for fetch-in-worker → main-thread pipelines.

---

## 6. Fetch / CORS

### Fetch streaming
- `fetch(url)` returns a `Response` whose `body` is a `ReadableStream<Uint8Array>`. Universally supported. [source: https://developer.chrome.com/docs/capabilities/web-apis/fetch-streaming-requests] [source: https://rob-blackbourn.medium.com/beyond-eventsource-streaming-fetch-with-readablestream-5765c7de21a1]
- **Streaming request bodies** (request body is a `ReadableStream` without `Content-Length`) trigger a CORS preflight in all browsers; supported in Chrome, Edge, Firefox. `UNCERTAIN` for Safari. [source: https://developer.chrome.com/docs/capabilities/web-apis/fetch-streaming-requests]

### `no-cors` mode
- `fetch(url, { mode: "no-cors" })` returns an **opaque response**: status 0, no readable headers, body not readable by JavaScript. Useful only for putting a response into a Cache or `<img>`/`<script>` tag. Cannot be used to read media bytes into JS. [source: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch] [source: https://http.dev/cors]

### Range requests
- Browser supports Range requests natively for `<video>`/`<audio>` tags. For `fetch()` you set the `Range` header manually (only allowed in CORS mode, and the `Range` header is NOT in the CORS-safelist, so it triggers a preflight and the origin must return `Access-Control-Allow-Headers: Range`). Many CDNs allow Range, but the *response* must be CORS-permissive.

### Cross-origin media direct fetch
- Cross-origin media **can** be fetched via `fetch(url, { mode: "cors" })` if and only if the origin returns `Access-Control-Allow-Origin`. Most social-media CDNs (TikTok, Instagram, Facebook) do NOT return permissive CORS headers — the browser will refuse to expose the body to JS.
- Workaround used in practice: route the request through a CORS proxy (public like `corsproxy.io`, or a self-hosted remote function). This is the canonical reason a "minimal remote infrastructure" component is needed for a browser-native media downloader.

### Referer / Origin header control
- The browser **cannot spoof `Referer` or `Origin`** for security reasons. The `Referrer-Policy` header/attribute can suppress or shorten the `Referer` (e.g. `no-referrer`, `strict-origin-when-cross-origin`) but cannot replace it with an arbitrary value. [source: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Referrer-Policy] [source: https://www.w3.org/TR/referrer-policy]
- Therefore, sites that gate requests on a specific `Referer` (e.g. CDN requires `Referer: https://www.tiktok.com/`) cannot be fetched directly from a different origin site — a remote proxy that sets the correct headers is required. **This is a critical architectural constraint.**

---

## 7. File System Access API (`showSaveFilePicker`, `showOpenFilePicker`, `showDirectoryPicker`)

### Support
- **Chrome / Edge** (and other Chromium browsers): stable since Chrome 86 (save picker) / Chrome 95 (open picker, directory picker). [source: https://www.testmuai.com/learning-hub/file-system-access-api-browser-support]
- **Firefox**: **NOT supported**, in any desktop or Android version. Mozilla has formally taken a standards position **against** shipping user-facing file pickers (concerns about phishing / drive-by file access). [source: https://www.testmuai.com/learning-hub/file-system-access-api-browser-support] [source: https://github.com/mozilla/standards-positions/issues/154] [source: https://stackoverflow.com/questions/74618688/showsavefilepicker-not-in-firefox-what-can-i-use-instead]
- **Safari (desktop & iOS)**: **partial**. Safari ships the **OPFS portion** of the File System API (since Safari 15.2) but **does not** ship `showSaveFilePicker` / `showOpenFilePicker` / `showDirectoryPicker` for user-visible file system access. WebKit has the bug open since 2021 but no ship. [source: https://bugs.webkit.org/show_bug.cgi?id=231706] [source: https://web-platform-dx.github.io/web-features-explorer/features/file-system-access]
- **Samsung Internet, Opera**: same as Chrome (Chromium-based).

### Fallback
- Browsers without the picker API must use:
  - `<a href="blob:..." download="name.ext">` (anchor download) for saving.
  - `<input type="file">` for opening (single or multiple files, but no directory).
  - The `File System Access API` ponyfill (`use-strict/file-system-access` library) wraps pickers + fallbacks into a single interface. [source: https://github.com/use-strict/file-system-access]

---

## 8. Browser downloads (`<a download>`, blob URLs, `saveAs`)

### `<a download>` attribute
- **Same-origin**: respected by all browsers. Causes the browser to download instead of navigate, with the supplied filename. [source: https://macarthur.me/posts/trigger-cross-origin-download]
- **Cross-origin**: Chrome (since Chrome 65, ~March 2018) and other browsers IGNORE the `download` attribute on cross-origin anchors — the link becomes a navigation. Same-origin, `blob:`, and `data:` URLs are unaffected. [source: https://groups.google.com/a/chromium.org/g/blink-dev/c/Iw3_SUcagGg] [source: https://stackoverflow.com/questions/49474775/chrome-65-blocks-cross-origin-a-download-client-side-workaround-to-force-down]
- **Safari**: historically ignored `download` attribute entirely; now "Safari honors the attribute only for same-origin, blob, and data URLs". [source: https://www.testmuai.com/posts/trigger-cross-origin-download] [source: https://www.testmuai.com/learning-hub/html-download-attribute-browser-support]

### Blob URL downloads (the reliable path)
- For a same-origin app that has the bytes in memory/OPFS, creating `const url = URL.createObjectURL(blob)` and clicking an `<a download href=url>` is the universally-working save pattern. Works in Chrome, Firefox, Safari (desktop + iOS).
- **Safari iOS quirk**: programmatic click on `<a download>` from a non-user-gesture context (e.g. inside `setTimeout` after async work) is blocked. Must be triggered from within a user-gesture call stack (e.g. user taps a button → fetch → blob → click synchronously, or use `navigator.share` with files for iOS). [source: https://www.testmuai.com/learning-hub/html-download-attribute-browser-support] `UNCERTAIN` for exact current iOS 26 behavior.

### Max download size
- No explicit spec limit; limited by available memory for the `Blob`. For very large files (multi-GB) the app should stream into OPFS first and then either use File System Access API save picker (Chrome) or trigger a chunked download (multiple `<a download>` invocations from user gestures — awkward UX).

### Download resumption
- Browser-driven download resumption is not exposed to JavaScript. The `<a download>` triggers the browser's native downloader, which *may* support resume for HTTP responses with proper `Accept-Ranges`/`ETag` headers — but JS cannot observe or control this.

---

## 9. IndexedDB

### Quota
- IndexedDB shares the origin storage pool with OPFS, Cache API, and Service Worker registrations.
- **Chrome / Edge**: up to ~60% of total disk per origin (after `navigator.storage.persist()`); ~80% global cap for all origins. [source: https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria]
- **Firefox**: up to ~50% of disk per origin, max 2 GB per origin on Firefox Android historically.
- **Safari / iOS Safari**: typically **~1 GB** per origin (older iOS was less); Safari is "historically more aggressive" with eviction on low space. [source: https://rxdb.info/articles/indexeddb-max-storage-limit.html] [source: https://diragb.dev/blog/indexeddb-vs-localstorage-vs-cookies]

### Persistence
- `navigator.storage.persist()` requests "durable" storage, exempting the origin from best-effort eviction. Supported in Chrome, Firefox, Safari 15.2+ (desktop + iOS). [source: https://dexie.org/docs/StorageManager] [source: https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria]
- **Safari caveat**: even with `persist()`, Safari may still evict if total device storage is critically low. Some sources report Safari does NOT always honor the persist grant on iOS — `UNCERTAIN`. [source: https://stackoverflow.com/questions/78474823/does-navigator-storage-persist-only-protect-against-data-removal-in-the-case-o]
- **iOS Safari 7-day purge**: For sites NOT added to home screen as a PWA, WebKit deletes IndexedDB, OPFS, local storage, and service worker registrations after 7 days of no use. [source: https://webkit.org/blog/14403/updates-to-storage-policy] [source: https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide]

### Structured clone limits
- IndexedDB stores values via structured clone, which supports `ArrayBuffer`, `Blob`, `File`, `TypedArray` natively. Large `Blob` storage is supported. The bottleneck is quota, not serialization.

### Eviction risk
- For a media downloader that may store large intermediate files, IndexedDB is **less suitable** than OPFS for large blob storage (no streaming writes, smaller effective quota due to indexing overhead). Use OPFS as the primary large-object store; use IndexedDB only for metadata.

---

## 10. Cache API

### Support
- Available in Service Workers, Web Workers (DedicatedWorker & SharedWorker), and the main thread (Chrome, Firefox, Safari 11.1+ desktop / Safari 11.3+ iOS). [source: https://web.dev/articles/service-workers-cache-storage] [source: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers]

### Storage quota
- Shares the same origin storage pool as IndexedDB / OPFS. Same quotas and eviction rules apply.

### Storing large media `Response` bodies
- `cache.put(request, response)` accepts any `Response`, including one whose `body` is a stream. The stream is consumed and stored. [source: https://web.dev/articles/service-workers-cache-storage]
- Internally, Cache API stores body data as compressed files on disk (in Gecko: "Body data for both Requests and Responses are stored directly in individual snappy compressed files"). [source: https://blog.wanderview.com/blog/2014/12/08/implementing-the-serviceworker-cache-api-in-gecko]
- Practical upper bound: same as origin storage quota (multi-GB feasible). 260 MB write observed on Cloudflare Workers (server-side, not browser). [source: https://community.cloudflare.com/t/use-cache-api-to-put-260mb-file-with-db-extension/248160]
- Cache API is **not** a good fit for very large media caches because keys are `Request` objects (URL-keyed); eviction is LRU and global. Prefer OPFS for media files, Cache API for HTTP replay.

### Streaming `put`
- Yes — `cache.put(req, new Response(readableStream))` works. The stream is consumed and body is written to disk. Useful for caching the result of a streaming fetch.

---

## 11. MediaSource Extensions (MSE)

### Support
- **Chrome / Edge / Firefox**: full `MediaSource` since long ago. [source: https://caniuse.com/mediasource]
- **Safari desktop**: full `MediaSource` since Safari 8.
- **Safari iOS / iPadOS (iPhone)**: **historically NO `MediaSource` support on iPhone**. Apple added **`ManagedMediaSource`** in **Safari 17.1 (iOS 17.1, October 2023)** for iPhone. `ManagedMediaSource` is a power-efficient variant of MSE that requires the media element to be in the DOM and visible; otherwise it pauses. [source: https://webkit.org/blog/14735/webkit-features-in-safari-17-1] [source: https://bitmovin.com/blog/managed-media-source] [source: https://github.com/AlexxIT/WebRTC/issues/573]
- **MSE in workers**: `MediaSource` constructor in workers — Chrome 100+ supports `MediaSourceHandle` for worker-side MSE; Safari does NOT support worker MSE. `UNCERTAIN` on Firefox.

### Use in this project
- MSE/MMS is for **playback** streaming, not downloading. It is relevant only if the project includes an in-browser preview player. For pure download/demux/remux, MSE is not needed.

---

## 12. Web Audio API (`decodeAudioData`)

### Support
- `AudioContext.decodeAudioData()` is universally supported (Chrome, Edge, Firefox, Safari desktop + iOS, with `webkitAudioContext` prefix in old Safari). [source: https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/decodeAudioData]
- Decodes MP3, AAC, Opus, Vorbis, FLAC, WAV, etc. — depending on browser codec availability.
- **Limits**:
  - The entire audio file must be loaded into an `ArrayBuffer` before decoding — no streaming decode.
  - Decoded data is held as `AudioBuffer` (Float32 PCM), which is much larger than the encoded source (e.g. 3 min stereo 44.1 kHz = ~60 MB in memory). For long audio this is impractical.
  - For longer audio, prefer WebCodecs `AudioDecoder` (streaming, lower memory).

---

## Support matrix

Legend: **S** = stable, **P** = partial / behind flag / recent ship, **—** = missing / not shipping. `?` = uncertain.

| API | Chrome (desktop/Android) | Firefox (desktop) | Firefox (Android) | Safari desktop | Safari iOS |
|---|---|---|---|---|---|
| WebCodecs `VideoDecoder` | S (94+) | S (130+) | P/? | S (≥17, full 26+) | S (≥17, full 26+) |
| WebCodecs `VideoEncoder` | S (94+) | S (130+) | P/? | S (≥17, full 26+) | S (≥17, full 26+) |
| WebCodecs `AudioDecoder` | S (94+) | S (130+) | P/? | S (26.0+, Sep 2025) | S (26.0+, Sep 2025) |
| WebCodecs `AudioEncoder` | S (94+) | S (130+) | P/? | S (26.0+, Sep 2025) | S (26.0+, Sep 2025) |
| WebCodecs `ImageDecoder` | S (94+) | P/? | P/? | S | S |
| OPFS | S (102+) | S (111+) | S | S (15.2+) | S (15.2+) |
| OPFS `createSyncAccessHandle` (worker only) | S | S | S | S (15.2+) | S (15.2+) |
| OPFS `createWritable` (streaming) | S | S | S | S | S |
| DedicatedWorker | S | S | S | S | S |
| SharedWorker | S | S | —/? | S (16+) | S (16+) |
| Service Worker | S | S | S | S (11.1+) | S (11.3+) |
| Background Sync API | S (Chrome only) | — | — | — | — |
| Periodic Background Sync | S (Chrome only) | — | — | — | — |
| Cross-origin isolation via HTTP headers | S | S | S | S | S |
| Cross-origin isolation on GitHub Pages | — (no header support) | — | — | — | — |
| `coi-serviceworker` workaround | S | S | S | S | S |
| `ReadableStream` | S | S | S | S (10.1+) | S (10.3+) |
| `WritableStream` | S | S | S | S (14.1+) | S (14.5+) |
| `TransformStream` | S (80+) | S (102+) | S | S (14.1+) | S (14.5+) |
| `pipeThrough` / `pipeTo` | S | S | S | S (10.1+) | S (10.3+) |
| Byte streams (`type:"bytes"`) | S | S | S | S (16+) | S (16+) |
| Async iteration over streams | S | S | S | S (16.4+) | S (16.4+) |
| `fetch` streaming response body | S | S | S | S | S |
| `fetch` streaming request body | S | S | ? | ? | ? |
| `fetch` `no-cors` opaque response | S | S | S | S | S |
| Range header on `fetch` (CORS) | S | S | S | S | S |
| File System Access `showSaveFilePicker` | S (86+) | — | — | — | — |
| File System Access `showOpenFilePicker` | S (95+) | — | — | — | — |
| File System Access `showDirectoryPicker` | S (95+) | — | — | — | — |
| `<a download>` same-origin | S | S | S | S | S |
| `<a download>` cross-origin | — (ignored) | — (ignored) | — (ignored) | — (ignored) | — (ignored) |
| Blob URL download | S | S | S | S | S (with user-gesture caveat) |
| IndexedDB | S | S | S | S | S |
| `navigator.storage.persist` | S | S | S | S (15.2+) | S (15.2+) |
| Cache API in workers | S | S | S | S | S |
| `MediaSource` (MSE) | S | S | S | S (8+) | — (iPhone) |
| `ManagedMediaSource` | — | — | — | S (17.1+) | S (17.1+, iPhone) |
| MSE in workers (`MediaSourceHandle`) | S (100+) | ? | ? | — | — |
| `AudioContext.decodeAudioData` | S | S | S | S | S |
| SharedArrayBuffer (cross-origin isolated) | S | S | S | S | S |
| SharedArrayBuffer (non-isolated) | — | — | — | — | — |

---

## Critical blockers for GitHub Pages deployment

1. **No custom HTTP headers on GitHub Pages** — `COOP: same-origin` and `COEP: require-corp` cannot be set. This breaks `SharedArrayBuffer` and therefore **multi-threaded WASM** (FFmpeg/libav built with pthreads, SQLite WASM with OPFS + threads, etc.). [source: https://github.com/orgs/community/discussions/13309] [source: https://stackoverflow.com/questions/68609682/is-there-any-way-to-use-sharedarraybuffer-on-github-pages]
   - **Workaround A**: ship single-threaded WASM. Lower performance, larger binary differences vs. threads, no SAB.
   - **Workaround B**: use `coi-serviceworker` to patch headers at runtime. Adds a one-time reload, requires all cross-origin assets to be CORP-compatible or use `credentialless`. [source: https://github.com/gzuidhof/coi-serviceworker] [source: https://docs.wasmer.io/sdk/wasmer-js/how-to/coop-coep-headers]
   - **Workaround C**: deploy to Cloudflare Pages / Netlify / Vercel instead of GitHub Pages. Loses GitHub-native hosting but unblocks headers.

2. **File System Access API unsupported in Firefox and Safari** — users on those browsers cannot pick a save destination via a native picker. Must fall back to `<a download>` from a Blob URL, which is awkward for multi-GB files and on iOS requires a user-gesture. [source: https://www.testmuai.com/learning-hub/file-system-access-api-browser-support]

3. **Cross-origin `download` attribute is universally ignored** — the app cannot simply redirect the browser to a remote media URL with `<a download>`. It must first fetch the bytes (CORS-problematic for many CDNs) into a same-origin Blob and then download. [source: https://groups.google.com/a/chromium.org/g/blink-dev/c/Iw3_SUcagGg]

4. **Browser cannot spoof `Referer` / `Origin` headers** — sites that gate media on a specific Referer cannot be fetched directly client-side. **Requires a remote proxy** that sets the correct Referer/Origin/User-Agent. This is the architectural reason for "minimal remote infrastructure" in the project goals. [source: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Referrer-Policy]

5. **CORS on media CDNs** — most social-media CDNs do NOT return `Access-Control-Allow-Origin`. Without a remote CORS proxy, the browser will refuse to expose the response body to JavaScript. Same conclusion as #4: remote proxy required.

6. **iOS Safari 7-day purge of non-PWA storage** — IndexedDB, OPFS, Cache API, and SW registrations are wiped after 7 days of no Safari visit for non-installed sites. For a downloader that stores intermediates between sessions, this is a real risk. Mitigation: encourage "Add to Home Screen", or design stateless per-session. [source: https://webkit.org/blog/14403/updates-to-storage-policy]

7. **Safari iOS does not support Background Sync or Periodic Background Sync** — no way to resume a partial download after the tab is closed or the device reboots. Downloads must complete in the foreground. [source: https://github.com/GoogleChrome/workbox/issues/2516] [source: https://developer.mozilla.org/en-US/docs/Web/API/Web_Periodic_Background_Synchronization_API]

8. **Safari WebCodecs `AudioDecoder`/`AudioEncoder` only since Safari 26 (Sep 2025)** — for older iOS Safari users (iOS < 26), audio decoding via WebCodecs is unavailable; must fall back to `AudioContext.decodeAudioData` (non-streaming, memory-heavy). [source: https://developer.apple.com/documentation/safari-release-notes/safari-26-release-notes]

9. **Firefox Android does not fully implement WebCodecs** (per 2026 dataset) — Firefox-on-Android users may have degraded media processing capability. `UNCERTAIN` exact gap. [source: https://webcodecsfundamentals.org/datasets/codec-analysis-2026]

---

## Notes for downstream passes

- The COOP/COEP-on-GitHub-Pages decision is the single largest architectural fork. Pass 2 red-team should pressure-test `coi-serviceworker` reliability (especially first-load reload UX on mobile Safari) vs. the cost of moving hosting off GitHub Pages.
- The "remote proxy for Referer/CORS" requirement is unavoidable for TikTok/Instagram/Facebook targets. Pass 2 should quantify cost and legal/abuse considerations.
- SharedWorker on iOS Safari 16+ appears stable but mobile-Safari WebView contexts may behave differently — verify in Pass 2 with a real iOS device.
- WebCodecs availability on Safari 26 is recent (Sep 2025); many real iOS users are still on iOS 17/18. Pass 3 decision ledger must define the browser support floor (e.g., "iOS 17 minimum for video, iOS 26 minimum for audio remux").
