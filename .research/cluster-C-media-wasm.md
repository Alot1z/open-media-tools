# Cluster C — Media Processing + WASM Ecosystem Research (Pass 1)

**Scope:** In-browser media processing (muxing/demuxing/transcoding/metadata) and the WASM language/runtime ecosystem relevant to building a browser-native media downloader.

**Research date:** Pass 1, mid-2026 snapshot.
**Status:** All claims are sourced. Items that remain fuzzy are explicitly marked `UNCERTAIN`.

---

## PART 1 — MEDIA

### 1.1 FFmpeg.wasm (`@ffmpeg/ffmpeg` / `@ffmpeg/core`)

**Current version.** `@ffmpeg/ffmpeg` 0.12.15 on npm. The 0.12 line is "fully compatible with ffmpegwasm v0.11.x" but updates emsdk to the latest and fixes bugs [source: https://www.jsdelivr.com/package/npm/@ffmpeg.wasm/main] [source: https://cloudsmith.com/navigator/npm/@ffmpeg/ffmpeg]. Two distributions: single-threaded (`@ffmpeg/core`) and multi-threaded (`@ffmpeg/core-mt`) [source: https://ffmpegwasm.netlify.app/docs/faq].

**Single-threaded vs multi-threaded.** Multi-threaded build requires `SharedArrayBuffer`, which in turn requires the page to be cross-origin isolated via COOP (`Cross-Origin-Opener-Policy: same-origin`) + COEP (`Cross-Origin-Embedder-Policy: require-corp` or `credentialless`) headers [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/137] [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/353]. The mt build is "around 2x faster" than single-threaded but consumes "a lot more memory and CPU" [source: https://ffmpegwasm.netlify.app/docs/faq].

**GitHub Pages blocker.** GitHub Pages does not let you set arbitrary HTTP response headers, so SAB-based mt builds do not work out of the box on `*.github.io`. Workarounds: deploy through a service worker that injects COOP/COEP, or host on Netlify/Vercel/Cloudflare Pages where headers are configurable [source: https://stackoverflow.com/questions/68609682/is-there-any-way-to-use-sharedarraybuffer-on-github-pages] [source: https://dannadori.medium.com/how-to-deploy-ffmpeg-wasm-application-to-github-pages-76d1ca143b17]. This is a real deployment headache for any GitHub Pages-hosted OSS demo site.

**WASM size.** ~25 MB for the core `.wasm` asset (downloaded once and cached). The mt build is larger. The 25 MB download is repeatedly cited as the headline pain point [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/83] [source: https://www.reddit.com/r/programming/comments/vo9g58/ffmpegwasm_a_pure_webassembly_javascript_port_of].

**Performance vs native.** ffmpeg.wasm's own docs measure `ffmpeg.exec()` against native ffmpeg on the same machine. The published numbers show ~5.2 s native vs ~128.8 s wasm — i.e. roughly **20–25× slower** for transcoding workloads [source: https://ffmpegwasm.netlify.app/docs/performance]. Independent HN/anecdotal reports put ffmpeg.wasm at ~3–5× slower for light tasks and ~25× slower for full transcodes; a 720p MPEG-2→x264 conversion was observed at ~40 fps on a modern desktop [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/326]. General academic numbers for WASM-vs-native put the gap at ~45 % on Firefox for SPEC CPU [source: https://ar5iv.labs.arxiv.org/html/1901.09056], but ffmpeg.wasm is much worse than that headline number because of memory-model and threading friction.

**SIMD.** As of the 0.12 line, SIMD intrinsics are still on the roadmap as the most promising path to improving single-threaded performance, but the maintainers describe them as "not easy" and unshipped in the default build [source: https://github.com/ffmpegwasm/ffmpeg.wasm/discussions/415]. UNCERTAIN whether a SIMD-enabled build is published under a separate tag.

**`-c copy` (remux without re-encode).** Yes, this works — ffmpeg.wasm runs the real ffmpeg CLI, so `ffmpeg -i in.mkv -c copy out.mp4` is supported and is the recommended pattern for "no quality loss, no re-encode" workflows [source: https://cyberstock.lol/free-mov-to-mp4] [source: https://www.codeitbro.com/tool/ts-to-mp4-converter] [source: https://fei.lv/ffmpeg]. This is fast because no decode/encode happens, only container repackaging.

**Memory limits.** "2 GB hard limit" on input file size due to wasm32 linear memory; "might become 4 GB in the future" [source: https://ffmpegwasm.netlify.app/docs/faq]. Practitioners report being able to exceed 2 GB using `WORKERFS` for file input on newer browsers, since the 4 GB wasm memory ceiling is now widely supported [source: https://github.com/ffmpegwasm/ffmpeg.wasm/discussions/755] [source: https://news.ycombinator.com/item?id=33794122]. Effective practical ceiling is "much lower" than the theoretical 2 GB due to allocation overhead [source: https://medium.com/@nikunjkr1752003/ffmpeg-wasm-integration-debugging-journey-report-e23d579e81a0] [source: https://renderio.dev/blogs/ffmpeg-wasm-guide]. UNCERTAIN how much real-world headroom exists on mobile Safari.

**Startup time.** Loading + instantiating the ~25 MB WASM module takes several seconds on first visit (cached afterward). A 720p transcode of a 4-minute clip ran at ~40 fps, which implies initial-load + per-clip latency dominates for short clips [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/326] [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/83].

**License (important for OSS distribution).** FFmpeg itself is LGPL v2.1+, but the ffmpeg.wasm distribution includes libx264 and libx265, which are GPL. The maintainer's stance: "wasm is designed for use in browsers only and includes components such as libx264 and libx265, which are licensed under the GPL. Therefore, YOU [must comply with GPL]" [source: https://github.com/ffmpegwasm/ffmpeg.wasm/issues/107] [source: https://www.ffmpeg.org/legal.html]. Practical implications:
- Shipping ffmpeg.wasm in a GPL-incompatible product is a legal risk.
- x264/x265 also carry patent-licensing concerns (H.264/H.265 patent pools) that are independent of the GPL question.
- An LGPL-only build (without x264/x265) is theoretically possible by recompiling but not what's shipped on npm.

### 1.2 WebCodecs for muxing/demuxing

WebCodecs (`VideoDecoder`, `VideoEncoder`, `AudioDecoder`, `AudioEncoder`) gives JavaScript direct access to the browser's hardware-accelerated codec pipeline [source: https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder]. Crucially, **WebCodecs decodes/encodes individual frames or chunks but does NOT demux or mux containers** — you need a separate library to walk the MP4/WebM/MKV boxes and feed `EncodedVideoChunk`/`EncodedAudioChunk` objects into WebCodecs [source: https://webcodecsfundamentals.org/basics/muxing].

**Browser support.** VideoDecoder is "full support" in Chrome 94+, Edge 94+, Firefox 130+; Safari support arrived in Safari 16.4 (older caniuse data showed "limited" through 16.3, now full) [source: https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder] [source: https://www.testmuai.com/learning-hub/webcodecs-browser-support] [source: https://caniuse.com/webcodecs]. Real-world reach is ~85 % of browsers for AV1 encode, ~90 % for AV1 decode in mid-2026 [source: https://www.reddit.com/r/AV1/comments/1qlvxn3]. Good enough to treat as baseline-available for a 2026 product.

### 1.3 Muxer/demuxer libraries

| Library | Lang | Status | Read | Write | Containers | Notes |
|---|---|---|---|---|---|---|
| **Mediabunny** | TS | Active (v1, July 2026) | yes | yes | MP4, MOV, WebM, MKV, HLS, WAVE, MP3, Ogg, ADTS, FLAC, MPEG-TS | Zero-dep, tree-shakable, supersedes mp4-muxer/webm-muxer [source: https://github.com/Vanilagy/mediabunny] [source: https://mediabunny.dev/guide/introduction] |
| **mp4-muxer** | TS | Deprecated → Mediabunny | no | yes | MP4 | Superseded by Mediabunny [source: https://github.com/Vanilagy/mp4-muxer] [source: https://github.com/Vanilagy/mp4-muxer/blob/main/MIGRATION-GUIDE.md] |
| **webm-muxer** | TS | Deprecated → Mediabunny | no | yes | WebM | Superseded by Mediabunny [source: https://github.com/Vanilagy/webm-muxer] |
| **mp4box.js** | JS (GPAC port) | Active, v1.0.0 (June 2025) | yes | yes (frag) | MP4/ISOBMFF | Official GPAC project, now TypeScript-typed. Demuxing/segmentation/sample-extraction is its strength [source: https://gpac.io/2025/06/19/announcing-mp4box-js-1-0-0-with-typescript-support] [source: https://github.com/gpac/mp4box.js] |
| **mux.js** | JS | Largely maintenance-only | yes | yes (limited) | MP4, TS, FLV | Older Brightcove library, predates WebCodecs |
| **webcodecs-encoder** | TS | Active | no | yes | MP4, WebM | Wraps WebCodecs encode + mux for H.264/VP9/VP8/AAC/Opus [source: https://classic.yarnpkg.com/en/package/webcodecs-encoder] |

**Verdict on muxers.** Mediabunny is the new default for "read/write media files in pure TS" (released July 2026 by the same author as mp4-muxer/webm-muxer, which it replaces). mp4box.js remains the gold standard for deep MP4/ISOBMFF inspection (segmenting for MSE, fragmenting DASH/HLS, sample extraction). For an MP4/WebM/MKV demuxer + muxer that hooks into WebCodecs, Mediabunny is the strongest single choice as of mid-2026.

### 1.4 MediaSource Extensions (MSE)

MSE is for **streaming playback**, not file output. It lets JavaScript feed `SourceBuffer` chunks to a `<video>` element for adaptive streaming (DASH/HLS-ish flows) [source: https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API] [source: https://web.dev/articles/media-mse-basics]. Relevance to a downloader: limited. MSE is useful if you want to *preview* a downloaded fragment before saving it, but you cannot get a finalized container file out of MSE — you need a real muxer for that.

### 1.5 Web Audio API

`AudioContext.decodeAudioData()` decodes compressed audio to `AudioBuffer` for analysis/playback. It is a decode-only convenience API and **cannot mux or remux** audio containers. Useful for metadata extraction (duration, channels, sample rate) and waveform rendering, not for any packaging task.

### 1.6 Browser-native codecs (what `<video>`/`<audio>` can play)

Source-stream playback compatibility dictates whether a downloaded stream can be saved "as-is" without re-encoding:

- **H.264 (AVC)** + **AAC**: universally supported across all browsers including Safari. Safe default.
- **VP9**: Chrome, Firefox, Edge, Safari 14+ (recent). Roughly as universal as H.264 in 2026 [source: https://webcodecsfundamentals.org/datasets/codec-analysis-2026].
- **AV1**: ~90 % of browsers decode, ~85 % encode. AV1+HEVC combined decode coverage ≈ 99.7 % [source: https://www.reddit.com/r/AV1/comments/1qlvxn3] [source: https://webcodecsfundamentals.org/datasets/codec-analysis-2026].
- **HEVC (H.265)**: native in Safari (macOS High Sierra+), Edge on Windows (with hardware), Chrome 107+ (with OS support). Coverage improving but inconsistent [source: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Video_codecs].
- **Opus**: Chrome, Firefox, Edge, Safari 17+ [source: https://evilmartians.com/chronicles/better-web-video-with-av1-codec].

**Relevance to "can we just save the source stream".** For YouTube/Vimeo/etc. the source is typically already H.264/AAC in MP4 or WebM/VP9/Opus — both playable by `<video>` and widely muxable. In those cases a downloader can save bytes verbatim and only needs to (a) demux the adaptive segments and (b) mux them back into a single MP4/WebM container. No re-encoding required, no FFmpeg.wasm required.

### 1.7 mp4box.js / ISOBMFF state

mp4box.js 1.0.0 was released June 19, 2025 with first-class TypeScript support [source: https://gpac.io/2025/06/19/announcing-mp4box-js-1-0-0-with-typescript-support]. Maintained by the GPAC team (the upstream MP4Box C tool). Strengths: parsing ISOBMFF box structure, fragmenting MP4 for MSE playback, sample-level demuxing (extracting EncodedVideoChunk-ready samples). Limitations: write support is mostly limited to fragmentation, not full remuxing or transcode. Good complement to Mediabunny.

---

## PART 2 — WASM LANGUAGES & RUNTIMES

### 2.1 Rust + wasm-bindgen / wasm-pack

**Maturity.** Rust is the de facto default for new browser WASM. The `wasm-bindgen` interop layer and `web-sys`/`js-sys` bindings are mature.

**Important 2025 tooling change.** `wasm-pack` (the rustwasm working group's build tool) was **sunset and archived in July 2025**. New projects must use cargo directly with `wasm32-unknown-unknown` target, or migrate to alternatives like `cargo-component` or roll-their-own build glue [source: https://nickb.dev/blog/life-after-wasm-pack-an-opinionated-deconstruction]. This is a meaningful ecosystem disruption to flag for any new project started today.

**Binary size.** Real-world Rust→WASM output is commonly 100 KB–3 MB after optimization. Tree-shaking is essential; debug builds balloon to multi-MB easily [source: https://rustwasm.github.io/book/reference/code-size.html] [source: https://medium.com/beyond-localhost/wasm-size-diet-rust-binaries-under-one-megabyte-9104c1bc30b2] [source: https://github.com/rustwasm/wasm-bindgen/issues/2856]. Smaller-than-100 KB binaries are achievable for tight, single-purpose modules.

**Interop.** Excellent: `wasm-bindgen` exposes Rust functions/structs to JS and lets Rust call back into the DOM. The generated JS glue is small.

**Fit for purpose.** Best-in-class for new browser WASM where you need performance and are willing to write Rust. If you're binding to existing C media libraries, emscripten is still the only path.

### 2.2 Zig

Zig has a first-class `wasm32` target. Zig's design (no implicit control flow, no exceptions, explicit memory management, comptime) aligns unusually well with WASM's flat semantics [source: https://0xkiire.com/wasm-with-zig] [source: https://medium.com/@daxx5/zig-was-built-for-webassembly-before-webassembly-even-existed-807a866be773]. Released builds can be smaller than equivalent Rust because of no `std` bloat and no allocator overhead by default.

**Maturity.** Still pre-1.0 (Zig 0.16 in mid-2026). The wasm target is functional but the ecosystem (wasm-bindgen-equivalent, web-sys-equivalent) is much thinner than Rust's. Community bindings exist (e.g. `zig-wasm-ffi`) but are not standardized [source: https://github.com/hotschmoe/zig-wasm-ffi]. Some users report "huge" wasm build sizes by default until they manually configure release flags [source: https://www.reddit.com/r/Zig/comments/1eony2f/wasm_build_size_is_huge].

**Why choose over Rust?** Smaller binaries, simpler language, no macro/lifetime learning curve. Why not? Much smaller ecosystem, no `wasm-bindgen` equivalent, no `cargo-component` story.

### 2.3 AssemblyScript

TypeScript-variant that compiles to WASM via Binaryen. Generates "lean WebAssembly" output and is approachable for JS developers [source: https://www.youtube.com/watch?v=97ej9-CE3Gc] [source: https://www.assemblyscript.org/status.html]. Limitations: requires manual memory management semantics on top of TS syntax, ecosystem of ready-made libraries is small, and important WASM features (e.g. GC, stack switching) lag behind Rust. Practical reality: good for small, single-purpose modules (codec utilities, math kernels); not great for complex applications.

### 2.4 C/C++ via emscripten

The only path for binding existing C media libraries (FFmpeg, GPAC/MP4Box, libvpx, x264, dav1d, mp3lame, etc.). Heavy toolchain (LLVM + clang + emsdk), produces large binaries with JS glue, but unmatched for reuse of mature native code. Both ffmpeg.wasm and the WASI builds of mp4box/dav1d come via emscripten.

### 2.5 Go → WASM

Poor fit. Standard `GOOS=js GOARCH=wasm` produces 10 MB+ binaries routinely; users report 25 MB in real projects [source: https://go.dev/wiki/WebAssembly] [source: https://www.reddit.com/r/golang/comments/1ipu4wd/webassembly_and_go_2025]. `TinyGo` reduces this to ~2.5 MB but compiles slowly, has limited stdlib coverage, and the GC overhead remains [source: https://news.ycombinator.com/item?id=43045698] [source: https://dev.to/eminetto/webassembly-using-go-code-in-the-browser-5ng]. Avoid for browser-side media work unless there's a hard "we already have this Go library" constraint.

### 2.6 Component Model / WIT / jco

**Where we are.**
- WASI 0.2 (preview2, Component-Model-based) launched early 2024; `jco 1.0` shipped April 2024 as the JS-native toolchain for components [source: https://blog.yoshuawuyts.com/jco-1.0-wasi-0.2] [source: https://github.com/bytecodealliance/jco].
- WASI 0.3.0 was ratified June 11, 2026, adding **native async support** to components. Available in Wasmtime 43+ and jco [source: https://bytecodealliance.org/articles/WASI-0.3] [source: https://github.com/bytecodealliance/wasi.dev/blob/main/docs/roadmap.md].
- Wasmtime was the first major runtime with full WASI 0.2 support [source: https://eunomia.dev/blog/2025/02/16/wasi-and-the-webassembly-component-model-current-status].

**Browser reality check.** Browsers only execute **core WASM modules**, not components. To run a component in a browser you must transpile it to core WASM + JS glue with `jco transpile` [source: https://news.ycombinator.com/item?id=48448083] [source: https://dev.to/topheman/webassembly-component-model-building-a-plugin-system-58o0]. So in the browser, the Component Model is a *build-time* abstraction, not a runtime feature. The composability benefits (cross-language WIT interfaces, pluggable imports) do survive the transpile step in a limited form.

**Production-readiness.** Honest take from the Rust/Bytecode-Alliance community: "No, it's not dead — multiple companies are using it in production" (server-side, Wasmtime) [source: https://www.reddit.com/r/rust/comments/1nld2a7/is_the_wasms_component_model_wasip2_is_already_dead]. Known limitations remain:
- WASI programs cannot spawn threads by default — bad for compute/IO-heavy workloads [source: https://eunomia.dev/blog/2025/02/16/wasi-and-the-webassembly-component-model-current-status].
- WASI in the browser needs a polyfill (jco); there is no native browser WASI host [source: https://component-model.bytecodealliance.org/language-support/building-a-simple-component/javascript.html].
- Cross-compilation to non-standard platforms (Android, embedded) is "not smooth" [source: https://www.reddit.com/r/rust/comments/1nld2a7].

**Verdict.** Production-ready **on the server** (Wasmtime, WasmCloud, Fermyon Spin, etc.). In the **browser**, it's a build-time toolchain for plugin systems, not a runtime. Realistic use for our project: a Component-Model-based plugin architecture where extractors are authored as components in any language (Rust/Go/JS via Javy) and transpiled to JS+WASM via jco at build time. The async story got significantly better in 0.3 (June 2026).

### 2.7 WASI 0.2 / 0.3 in the browser

Browsers do not implement WASI directly. To run WASI 0.2/0.3 programs in a browser you must use `jco` (transpiles + provides JS host implementations of `wasi-cli`, `wasi-http`, `wasi-filesystem`, etc.). This is workable but adds glue weight. For most browser-side media work, you're better off writing core WASM modules directly (no WASI) and exposing only the narrow functions you need.

### 2.8 Javy (Bytecode Alliance)

Javy compiles JavaScript to WASM by embedding QuickJS as the runtime. Output modules are **1–16 KB** for small scripts — impressively small [source: https://github.com/bytecodealliance/javy] [source: https://bytecodealliance.org/articles/javy-hosted-project]. Originally built by Shopify for "Shopify Functions," adopted as a Bytecode Alliance hosted project in June 2023 [source: https://bytecodealliance.org/articles/javy-hosted-project]. In 2025 the Javy CLI was overhauled for extensibility [source: https://blogs.igalia.com/compilers/2026/05/25/five-years-of-javascript-on-webassembly].

**Use case for us.** Run yt-dlp-style site extractors (which are currently Python/JS) inside a WASM sandbox. Javy supports modern JS, including async. Limitations: QuickJS is single-threaded and slower than V8; some Node-specific APIs (fs, net) aren't available unless explicitly provided by the host.

### 2.9 QuickJS + emscripten

`quickjs-emscripten` is the alternative JS-in-WASM runtime. Two build variants: sync (smaller, faster) and async (uses Asyncify transform, ~2× size, ~40 % the speed of sync) [source: https://github.com/justjake/quickjs-emscripten/blob/main/doc/quickjs-emscripten-core/README.md]. Javy is essentially QuickJS compiled to WASM via a different toolchain. For a sandbox that needs to run untrusted JS, either works; Javy is more modern and produces smaller modules.

### 2.10 Extism

Extism (by Dylibso) is a "universal plugin system" using WASM. Plugins are WASM modules built with PDKs (Plugin Development Kits) in Rust, Go, JS, Python, C, etc. Hosts embed plugins via Host SDKs in 15+ languages including **JavaScript for browsers** (Firefox/Chrome/WebKit) [source: https://github.com/extism/js-sdk] [source: https://dylibso.com/products/extism] [source: https://extism.org/docs/concepts/plug-in]. Features: stateful plugins (memory persists between calls), unified guest/host API, sandboxed execution. Real-world adoption: several projects use it for WASM plugin systems [source: https://github.com/extism/extism/discussions/684].

**Fit for us.** Plausible foundation for a plugin architecture where users (or the community) contribute site extractors or post-processors as small WASM modules in any language. The browser JS SDK is officially supported. Maturity: production-grade for plugin-system use cases, less battle-tested for very high-throughput media processing.

### 2.11 Pyodide

CPython compiled to WASM via emscripten. Current version: 314.0.4. Initial download: 6.4 MB; environment init: 4–5 seconds; runtime is ~16× slower than native CPython [source: https://pyodide.org/en/stable/project/roadmap.html] [source: https://news.ycombinator.com/item?id=27127387]. Pure-Python packages installable via `micropip`; packages with C extensions need to be pre-built for emscripten.

**Can it run yt-dlp?** yt-dlp's hard dependencies are: `mutagen` (pure Python — should work), `certifi` (pure Python), `websockets`/`requests`/`urllib3` (mostly pure Python with some C accelerators), and an external **`ffmpeg` binary** (not a Python package) [source: https://pypi.org/project/yt-dlp] [source: https://manpages.ubuntu.com/manpages/noble/man1/yt-dlp.1.html]. yt-dlp is ~3000 Python source files plus 8 optional dependencies [source: https://forum.endeavouros.com/t/unable-to-install-yt-dlp/35476/12].

**Realistic verdict.** Pure-Python yt-dlp extractors could probably be imported into Pyodide after some packaging work, but: (1) yt-dlp shells out to `ffmpeg` for muxing/merging, which Pyodide can't do natively — you'd need ffmpeg.wasm as a sidecar and a Python↔JS bridge; (2) yt-dlp relies on stdlib modules (`socket`, `ssl`, `subprocess`, `ctypes`) that Pyodide emulates imperfectly via JS shims; (3) the 6.4 MB + 4–5 s init cost is heavy if it's not the core of the app. **UNCERTAIN** whether a meaningful subset of yt-dlp's extractors can run unmodified in Pyodide. A Javy/QuickJS-based port of extractors to JS is likely a lighter-weight path.

### 2.12 WebContainers (StackBlitz)

WebContainers runs Node.js natively in the browser, supporting `node`, `npm`, `yarn`, and full-stack frameworks like Next.js/Express — entirely in the browser security sandbox [source: https://blog.stackblitz.com/posts/introducing-webcontainers] [source: https://webcontainers.io]. Requires SharedArrayBuffer → COOP + COEP `credentialless` headers [source: https://blog.stackblitz.com/posts/bringing-webcontainers-to-all-browsers]. Limitations: no native modules (no Node N-API addons), no direct TCP/UDP sockets, network egress restricted to fetch/WebSocket, large runtime download.

**Fit for us.** Overkill. WebContainers is designed as a dev environment, not as a runtime for shipping a media downloader to end users. Could you `npm install yt-dlp-exec` inside a WebContainer and run it? Theoretically yes, but you'd be shipping a full Node runtime (~10 MB+), would still need ffmpeg.wasm sidecar (since ffmpeg's native binary can't run in WebContainer), and you'd inherit the COOP/COEP deployment tax. Not recommended.

---

## SYNTHESIS — Tables and Verdicts

### WASM Language Fit Table

| Language | Binary size (typical) | Ecosystem | Maturity (browser WASM) | Fit-for-purpose (media downloader) |
|---|---|---|---|---|
| **Rust** | 100 KB – 3 MB | Best-in-class (wasm-bindgen, web-sys; but wasm-pack sunset Jul 2025) | Mature | Strong: for performance-critical utilities (parsing, demuxing, hashing) |
| **Zig** | < 100 KB achievable | Thin; no wasm-bindgen equivalent | Pre-1.0, working | Niche: smaller-than-Rust modules where ecosystem doesn't matter |
| **AssemblyScript** | Small (lean by design) | Small; lags Rust on WASM features | Stable but narrow | Acceptable for tiny utility modules; not for app logic |
| **C/C++ (emscripten)** | 1–30 MB (heavy) | Reuses every existing C lib | Mature | Required if binding ffmpeg/gpac/dav1d/x264; otherwise heavy |
| **Go (std)** | 10–25 MB | Large but bloated output | Mature output, poor size | Poor fit (size, GC) |
| **Go (TinyGo)** | ~2 MB | Limited stdlib | Maturing | Better than std Go, still poor fit |
| **JS via Javy** | 1–16 KB | Full JS via QuickJS | Production (Shopify) | Excellent for sandboxed extractors |
| **JS via quickjs-emscripten** | ~500 KB–2 MB | Full JS | Production | Same as Javy but larger |
| **Python via Pyodide** | 6.4 MB+ (CPython) | Full PyPI pure-Python subset | Production (JupyterLite) | Heavy; could run yt-dlp extractors with effort, but ffmpeg sidecar needed |

### Component Model / WASI Readiness — Honest Assessment

- **Server (Wasmtime, WasmCloud, Spin):** Production-ready. WASI 0.2 stable since early 2024, 0.3 with async since June 2026. Multiple companies running it in production.
- **Browser:** Components are a **build-time** abstraction only. Browsers run core WASM modules; you must `jco transpile` your component to JS+WASM. Composability benefits (cross-language WIT interfaces) survive transpilation but lose some runtime flexibility.
- **Known gaps:** No threads by default in WASI; Asyncify overhead for sync↔async bridging; cross-platform compile to non-Linux/macOS/Windows targets still rough.
- **Recommendation for our project:** Use the Component Model only if we commit to a plugin architecture (multiple languages, third-party contributors). For our own first-party code, plain core WASM modules via Rust + wasm-bindgen (or emscripten for C bindings) are simpler and ship smaller.

### "Is FFmpeg.wasm actually necessary?" — Verdict

**Often NO.** Three observations:

1. **For remux-only workflows (`-c copy`)** the source stream is usually already a browser-playable codec (H.264/AAC, VP9/Opus, AV1). For these, a pure-TS muxer/demuxer (Mediabunny, mp4box.js) running on top of the File System Access API is **faster, smaller (~50 KB vs 25 MB), and avoids the GPL/x264 patent exposure** of ffmpeg.wasm.

2. **For genuine transcoding** (rarely required for a downloader — most sources already ship browser-friendly codecs), ffmpeg.wasm is the only mature general-purpose option, but: ~25 MB download, ~20–25× slower than native, 2 GB practical file ceiling, GPL/patent license encumbrance, and the COOP/COEP/SAB deployment tax for the multithreaded build.

3. **For WebCodecs-based transcoding** (e.g. AV1→H.264 to broaden compatibility), use `VideoEncoder` + `AudioEncoder` + Mediabunny as the muxer. Hardware-accelerated, ~10× faster than ffmpeg.wasm for the codecs browsers support, no GPL baggage. Trade-off: only handles codecs the host browser knows.

**Bottom line:** Use ffmpeg.wasm as a **fallback only**, for the rare "the source is some exotic codec and we must transcode" case. Default path should be: (a) demux source segments with Mediabunny/mp4box.js, (b) if codec is browser-playable, remux to MP4/WebM with Mediabunny, (c) only if a transcode is unavoidable and the codec is supported by WebCodecs, use WebCodecs + Mediabunny, (d) ffmpeg.wasm as the escape hatch.

### "Is WASM actually necessary?" — Verdict

**For most of the downloader, NO.** The core flow — HTTP fetching, segment downloading, demuxing MP4/WebM/MKV boxes, remuxing to MP4, writing to disk via File System Access API — is achievable in pure TypeScript with libraries like Mediabunny (zero-dep TS) and mp4box.js (pure JS). No WASM needed.

**Where WASM earns its place:**
- **Sandboxed third-party plugin execution** (extractors contributed by the community) — Javy or Extism.
- **Performance-critical codecs that WebCodecs doesn't expose** (rare in 2026 — H.264, HEVC, VP9, AV1, Opus, AAC are all in WebCodecs).
- **Reusing existing C libraries** (libvpx, dav1d, GPAC) when a pure-TS port doesn't exist or isn't fast enough.
- **Hashing/CRC/encryption** utilities (edge case; pure JS is usually adequate).

For a v1 of the project, **start without WASM** and add it only where profiling proves it's needed. This keeps the bundle small and avoids the COOP/COEP/SAB deployment headaches that ffmpeg.wasm users have been fighting for years.

### "Is Rust actually necessary?" — Verdict

**NO, not for v1.** Rust is the right choice *if* you decide to write custom WASM modules. But:

- Mediabunny + mp4box.js cover muxing/demuxing in pure TS.
- WebCodecs covers codec encode/decode in the browser's native implementation.
- The `yt-dlp` extractor logic is Python and can be reimplemented in TS (or run via Javy if a sandbox is needed) without Rust.

**If/when Rust becomes justified:**
- High-throughput parsing (e.g. an MP4 box walker that needs to scan 4 GB files in milliseconds).
- Custom codec (e.g. a niche container format Mediabunny doesn't support).
- Cryptographic/DRM-related work where you want a memory-safe, audited implementation.

When that day comes, note that `wasm-pack` was archived in July 2025 — plan to use `cargo` directly with the `wasm32-unknown-unknown` target, or `cargo-component` if you've adopted the Component Model for plugins.

---

## OPEN QUESTIONS (for Pass 2 / other clusters)

1. Does Mediabunny handle live/fragmented MP4 (fMP4) for DASH-style sources? [UNCERTAIN — needs hands-on test]
2. Does the WebCodecs `VideoEncoder.isConfigSupported()` accept the codec strings YouTube/Vimeo actually use? [Needs verification across browsers]
3. For HEVC sources on Safari, does WebCodecs `VideoDecoder` produce frames that can be re-muxed into a playable MP4 on Chrome? [Cross-browser codec interoperability UNCERTAIN]
4. Realistic cold-start time for ffmpeg.wasm 0.12 mt on mid-range Android Chrome? [UNCERTAIN — no published mobile benchmark found]
5. Does Javy support enough of the modern JS feature set (top-level await, `Promise.allSettled`, `structuredClone`) for a real yt-dlp extractor port? [UNCERTAIN — needs test]
6. Is there a working `cargo-component` story for browser-targeted components yet, post-wasm-pack sunset? [UNCERTAIN — flagged for follow-up]

---

## SUMMARY

- **Media:** Mediabunny (July 2026, pure-TS, supersedes mp4-muxer/webm-muxer) + mp4box.js 1.0 (June 2025) cover muxing/demuxing/remuxing for ~95 % of "save the source stream" cases without any WASM. WebCodecs (Chrome 94+/FF 130+/Safari 16.4+) handles hardware-accelerated codec encode/decode. ffmpeg.wasm 0.12 remains the escape hatch for exotic codecs but carries 25 MB download, ~25× slowdown, GPL/x264-patent exposure, and the COOP/COEP/SAB GitHub-Pages blocker.
- **WASM languages:** Rust is best for new modules (but `wasm-pack` was archived July 2025 — use cargo directly). Zig is interesting but ecosystem-thin. C/C++ via emscripten is mandatory for binding existing C libs. Go is a poor fit (10–25 MB binaries). Javy (1–16 KB modules, JS-in-WASM) is excellent for sandboxed extractors.
- **Component Model / WASI:** Production-ready on the server (Wasmtime 43+, WASI 0.3 with async landed June 2026). In the browser, it's a build-time abstraction — `jco transpile` converts components to JS+core WASM. Use it for plugin architectures, not for runtime browser features.
- **Verdicts:** FFmpeg.wasm — not necessary for v1, escape hatch only. WASM — not necessary for v1, add only where profiling demands. Rust — not necessary for v1, justified later if custom WASM modules are needed.

**File written:** `/home/z/open-media-tools/.research/cluster-C-media-wasm.md`
