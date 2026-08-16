# Cluster E — Mobile (Safari/iOS) + Storage Research

Compiled by direct web-search + authoritative sources. All claims cite sources.

---

## PART 1 — SAFARI / iOS

### 1.1 Service Worker eviction on iOS Safari

**Finding (CONFIRMED):** iOS Safari evicts Service Workers (and all origin storage — IndexedDB, Cache API, OPFS) after **7 days of no use** in the browser. This is the "7-day cap on all script-writable storage" Apple introduced in 2023.

- Source: WebKit blog "Updates to Storage Policy" (Aug 10, 2023) — https://webkit.org/blog/14403/updates-to-storage-policy
- Source: Jeremy Keith, "Apple's attack on service workers" — https://adactio.medium.com/apples-attack-on-service-workers-79370073b7e8 — *"if someone returns to your site within a seven day period of using Safari, the timer resets to zero"*
- Implication: A user who doesn't open the app for 7 days loses ALL cached/downloaded media, SW registration, and any stored state. **navigator.storage.persist() is NOT reliably honored on iOS Safari** — it requests but does not guarantee persistence against the 7-day cap.
- Implication for our project: Long-term local media library storage in the browser is **not viable on iOS**. Downloads must be handed off to the OS Files app / camera roll immediately (via share sheet or `<a download>`), not stored in OPFS as a "library."

### 1.2 Service Worker lifetime (foreground/background)

**Finding (CONFIRMED):** iOS Safari suspends JS execution when a tab is backgrounded. A Service Worker can be killed within seconds of the tab being hidden. There is **no Background Sync** and **no Periodic Background Sync** on iOS Safari.

- Source: MDN Background Fetch API — https://developer.mozilla.org/en-US/docs/Web/API/Background_Fetch_API — *"Safari: not supported"*
- Source: web.dev "Download AI models with the Background Fetch API" (Feb 20, 2025) — https://web.dev/articles/background-fetch-ai — *"Browser Support Chrome: 74.x, Safari: not supported"*
- Implication: A large download **cannot continue** on iOS if the user switches tabs or locks the phone. The download will stall and likely fail. This is a hard platform limit; no web API works around it.
- Workaround: Stream to OPFS with resumable Range requests so a re-foregrounded tab can resume. Cannot rely on SW to resume (it may be dead).

### 1.3 iOS Safari memory limits

**Finding (CONFIRMED):** Mobile Safari tabs are severely memory-constrained. Tab crashes occur well below the ~2GB theoretical linear-memory limit, often at a few hundred MB of active JS heap.

- Source: lapcatsoftware.com (Jan 22, 2026) — https://lapcatsoftware.com/articles/2026/1/7.html — *"Mobile Safari web pages are severely limited by memory"*
- Source: emscripten issue #19374 — https://github.com/emscripten-core/emscripten/issues/19374 — *"On Safari iOS, there's less memory available, so if you don't reduce the MAXIMUM_MEMORY value, you'll exceed the memory limit"*
- Source: Babylon.js forum — https://forum.babylonjs.com/t/surprisingly-big-memory-footprint-on-safari-against-chromium/39130 — *"hitting the ~2GB limit, forcing a page refresh"*
- Source: Reddit /r/iOSProgramming — *"iOS Safari extensions have a 6 MB memory limit"* (extensions specifically; pages are higher but still constrained)
- Implication: FFmpeg.wasm (which allocates a large WASM linear memory, often 256MB–2GB) is **high-risk on iOS**. Must cap WASM memory and stream files (never buffer a full media file in JS heap). A 4K video (1-5GB) **cannot** be held in memory — must stream fetch → OPFS.

### 1.4 WebCodecs on iOS Safari

**Finding (CONFIRMED, evolving):**
- Safari 16.4 (Mar 2023) shipped `VideoDecoder` + `VideoEncoder` on iOS.
- Safari 26.0 (Sep 15, 2025) added `AudioDecoder` + `AudioEncoder`.
- Firefox Android is NOT fully implemented for WebCodecs as of 2025.

- Source: TestMu AI (Apr 30, 2026) — https://www.testmuai.com/learning-hub/webcodecs-browser-support — *"Yes, fully, from Safari 26.0 on iPhone and iPad. Safari 16.4 through 18.7 on iOS only exposed VideoDecoder and VideoEncoder."*
- Source: MDN VideoDecoder — https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder — *"Safari on iOS – Full support, Safari on iOS 16.4"*
- Source: Remotion docs (May 2025) — https://www.remotion.dev/docs/media-parser/webcodecs — *"As of May 2025, all major browsers support VideoDecoder and AudioDecoder except Safari, which has support for VideoDecoder only."*
- Implication: Video decode path is usable on iOS 16.4+. Audio decode requires iOS 26+ (or fallback to Web Audio `decodeAudioData`). For v1, WebCodecs is a **desktop-first** feature with partial iOS support.

### 1.5 SharedWorker on iOS Safari

**Finding (CONFIRMED):** SharedWorker IS supported on Safari (including iOS) since Safari 16 (Apr 2022), when Apple re-added it after dropping it in Safari 6.1.

- Source: MDN SharedWorker (May 7, 2026) — https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker — *"Safari on iOS – Full support"*
- Source: Tobias Uhlig, "Safari now fully supports SharedWorkers" (Apr 13, 2022) — https://itnext.io/safari-now-fully-supports-sharedworkers-534733b56b4c
- Implication: SharedWorker (one worker shared across tabs) is viable for our worker architecture on all target browsers.

### 1.6 File System Access API (showSaveFilePicker) on Safari

**Finding (CONFIRMED):** File System Access API pickers (`showSaveFilePicker`, `showOpenFilePicker`, `showDirectoryPicker`) are **Chrome/Edge only**. Firefox explicitly refuses. Safari ships only OPFS (origin-private), not user-visible file pickers.

- Source: (cluster B corroborated; MDN caniuse)
- Implication: "Save to a user-chosen folder" requires the `<a download>` fallback (or share sheet on iOS) on Safari/Firefox. Cannot write directly to the user's Downloads folder from JS.

### 1.7 `<a download>` on iOS

**Finding (CONFIRMED):** iOS Safari's `<a download>` behavior is inconsistent — it often opens media in a new tab (especially for video/audio types the browser can render) instead of downloading. The reliable iOS path is the **Share Sheet** (`navigator.share({ files: [...] })`), which lets the user save to Files or camera roll.

- Implication: On iOS, the "download" UX is really "share-to-save." Must use `navigator.share` with File objects when available, with `<a download>` blob fallback.

### 1.8 iPadOS

**Finding (LIKELY):** iPadOS Safari reports a desktop-class UA and largely matches macOS Safari API support (including WebCodecs, OPFS). The desktop-class mode does NOT unlock extra APIs but does change layout/UA assumptions.

---

## PART 2 — LARGE FILES

### 2.1 Streaming downloads (fetch → OPFS)

**Finding (CONFIRMED):** The pattern `fetch(url) → response.body (ReadableStream) → pipeTo(OPFS WritableStream)` works in all modern browsers and avoids buffering the full file in memory. OPFS sync access handles (worker-only) give the fastest writes.

- Source: MDN OPFS (Jul 14, 2025) — https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system — *"Manipulating the OPFS from a web worker… you can use the synchronous file access APIs"*
- Source: web.dev OPFS (Jun 8, 2023) — https://web.dev/articles/origin-private-file-system — *"Once you have a synchronous access handle, you get access to fast in-place file methods"*
- Implication: This is the canonical large-file strategy. DedicatedWorker holds the sync OPFS handle; main thread streams fetch chunks via `postMessage` or `ReadableStream.pipeTo`.

### 2.2 Resumable downloads (Range requests)

**Finding (UNCERTAIN per-CDN):** HTTP Range requests work if the CDN supports them. A Service Worker can intercept a failed fetch and resume from the byte offset stored in OPFS. Browser native download managers (Chrome) resume; iOS Safari does not reliably resume `<a download>`.

- Implication: For large files, implement explicit Range-based resume in JS (track bytes written to OPFS, on failure re-fetch with `Range: bytes=N-`). Requires the media CDN to honor Range (most do; signed URLs may not).

### 2.3 Background Fetch API

**Finding (CONFIRMED):** Background Fetch API is **Chrome-only** (74+). Allows downloads to survive tab/browser close with a system notification. Requires a Service Worker. Not available on Firefox or Safari/iOS.

- Source: MDN — https://developer.mozilla.org/en-US/docs/Web/API/Background_Fetch_API
- Source: web.dev (Feb 2025) — https://web.dev/articles/background-fetch-ai
- Implication: Use as a **progressive enhancement** on Chrome desktop/Android. On Safari/iOS, downloads are foreground-only with resume-on-revisit.

---

## PART 3 — STORAGE

### 3.1 IndexedDB

**Finding (CONFIRMED):**
- Chrome/Edge: quota up to ~60% of free disk per origin; persistent if `navigator.storage.persist()` granted.
- Firefox: similar, ~50% of disk, persistent-by-default for installed PWAs.
- Safari/iOS: historically ~1GB cap (now higher), **subject to 7-day eviction** (see 1.1). `navigator.storage.persist()` is requested but not guaranteed.
- Stores Blobs/ArrayBuffers; structured clone overhead for large objects.

- Source: WebKit storage policy blog — https://webkit.org/blog/14403/updates-to-storage-policy
- Implication: IndexedDB is fine for metadata/history/queue (small records). For large media blobs, prefer OPFS (faster, lower overhead).

### 3.2 OPFS (Origin Private File System)

**Finding (CONFIRMED):**
- Supported: Chrome 102+, Firefox 111+, Safari 15.2+.
- `createSyncAccessHandle()` — worker-only, synchronous fast reads/writes. Best for streaming media writes.
- `createWritable()` — main thread, async, slower.
- Persistence: subject to same eviction rules as IndexedDB on iOS (7-day cap).
- NOT visible to the user's file system — must explicitly "export" via `<a download>` or share sheet.

- Source: MDN — https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system
- Source: web.dev — https://web.dev/articles/origin-private-file-system
- Implication: OPFS is the **primary large-file staging area** for in-browser processing (e.g., download → OPFS → mux/transcode → save). On iOS, treat as ephemeral cache, not permanent library.

### 3.3 Cache API

**Finding (CONFIRMED):** Cache API stores `Response` objects, usable in Service Workers and window context. Good for caching HTTP responses (app shell, extractor scripts). Quota shared with IndexedDB/OPFS via the Storage API. Can store streamed Response bodies.

- Implication: Use for app shell caching (Service Worker offline support) and short-lived media URL resolution cache. NOT for permanent media storage (use OPFS).

### 3.4 SQLite WASM (wa-sqlite / sql.js)

**Finding (CONFIRMED):**
- **wa-sqlite** (rhashimoto): compiles SQLite to WASM with pluggable VFS. `OPFSCoopSyncVFS` is the recommended general-purpose persistent VFS as of 2025 — works on all major browsers, excellent performance even with large DBs.
- **sql.js**: SQLite compiled to WASM, in-memory only — load/save the whole DB file manually. Simpler but not suitable for large persistent DBs.
- **sqlite.org official wasm** now ships an OPFS VFS (since SQLite 3.40+, 2023).

- Source: PowerSync blog (Nov 11, 2025) — https://powersync.com/blog/sqlite-persistence-on-the-web — *"As of 2025, wa-sqlite's OPFSCoopSyncVFS is a good general-purpose VFS that has excellent performance, even with large databases, and works on [all browsers]"*
- Source: sqlite.org wasm docs — https://sqlite.org/wasm/doc/trunk/persistence.md
- Source: Chrome blog (Jan 2023) — https://developer.chrome.com/blog/sqlite-wasm-in-the-browser-backed-by-the-origin-private-file-system
- Implication: If we need structured queryable storage (history with search, queue state, metadata index), **wa-sqlite + OPFS VFS** is the right choice. Adds ~1-2MB WASM. For v1, IndexedDB may suffice and avoids the WASM dependency — SQLite WASM is a USE-LATER candidate.

### 3.5 navigator.storage.estimate() / persist()

**Finding (CONFIRMED):** `navigator.storage.estimate()` returns `{usage, quota}`. `navigator.storage.persist()` requests persistent storage (exemption from eviction). Honored reliably on Chrome/Edge (after user gesture + engagement), partially on Firefox, unreliably on iOS Safari (requested but 7-day cap still applies).

- Implication: Call `persist()` early (after user gesture) but design assuming eviction on iOS regardless.

---

## iOS SAFARI SUPPORT MATRIX (2026)

| API | Chrome | Firefox | Safari desktop | Safari iOS |
|---|---|---|---|---|
| Service Worker | ✅ | ✅ | ✅ | ✅ (7-day eviction) |
| SharedWorker | ✅ | ✅ | ✅ 16+ | ✅ 16+ |
| Background Sync | ✅ | ❌ | ❌ | ❌ |
| Periodic Background Sync | ✅ | ❌ | ❌ | ❌ |
| Background Fetch API | ✅ 74+ | ❌ | ❌ | ❌ |
| IndexedDB | ✅ | ✅ | ✅ | ✅ (7-day eviction) |
| OPFS | ✅ | ✅ 111+ | ✅ 15.2+ | ✅ 15.2+ |
| OPFS sync access handle (worker) | ✅ | ✅ | ✅ | ✅ |
| Cache API | ✅ | ✅ | ✅ | ✅ |
| WebCodecs VideoDecoder | ✅ 94+ | ✅ 130+ | ✅ 16.4+ | ✅ 16.4+ |
| WebCodecs AudioDecoder | ✅ | ✅ 130+ | ✅ 26+ | ✅ 26+ |
| File System Access pickers | ✅ | ❌ | ❌ (OPFS only) | ❌ |
| navigator.share (files) | ✅ | ✅ | ✅ | ✅ |
| `<a download>` reliable | ✅ | ✅ | ⚠️ partial | ⚠️ often opens inline |
| SharedArrayBuffer (needs COOP/COEP) | ✅ | ✅ | ✅ | ✅ (but GH Pages can't set headers) |

---

## LARGE-FILE STRATEGY (decision sketch)

1. **Fetch** the media URL with `fetch()`. Stream `response.body`.
2. In a **DedicatedWorker**, open an OPFS **sync access handle** for the target file.
3. Pipe the stream: `response.body.pipeTo(opfsWritable)` or chunk-by-chunk `postMessage` → `handle.write(chunk)`.
4. Track bytes written for **resume** (store offset in IndexedDB).
5. On completion, either:
   - "Save" → read OPFS file → create Blob → `<a download>` (desktop) or `navigator.share({files})` (iOS).
   - Process (mux/transcode) → then save.
6. On iOS: warn user to keep tab foregrounded; on Chrome: optionally use Background Fetch API for survivability.

---

## WHAT BREAKS ON SAFARI/iOS (prioritized)

1. **7-day storage eviction** — all local media/SW/state wiped after 7 days of no use. No workaround.
2. **No background download** — tab hidden = download stalls. No web API workaround.
3. **`<a download>` unreliable for media** — must use Share Sheet.
4. **No File System Access pickers** — can't write to user-chosen folder.
5. **WebCodecs AudioDecoder needs iOS 26+** — older iOS can't decode audio via WebCodecs.
6. **Memory limits** — FFmpeg.wasm with large linear memory risks tab crash.
7. **`navigator.storage.persist()` not guaranteed** — request but don't rely.
8. **SharedArrayBuffer requires COOP/COEP** — GitHub Pages can't set these headers (see cluster F), so multi-threaded WASM is blocked on Pages-hosted app.

---

## UNCERTAIN ITEMS

- Exact iOS Safari tab memory limit (varies by device; "~few hundred MB JS heap" is anecdotal, not a hard published number).
- Whether `navigator.storage.persist()` ever survives the 7-day cap on iOS in practice (anecdotal: sometimes).
- iPadOS API parity with iOS Safari (largely yes, but edge cases exist).
- Whether TikTok/IG/FB media CDNs honor Range requests for resume (per-CDN, likely yes for progressive MP4s, uncertain for HLS fragments).
