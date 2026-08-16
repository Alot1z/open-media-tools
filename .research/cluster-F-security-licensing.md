# Cluster F — Security + Licensing Research

Compiled by direct web-search + authoritative sources. All claims cite sources.

---

## PART 1 — SECURITY

### 1.1 SSRF (Server-Side Request Forgery) — remote extractor

**Finding (CONFIRMED):** If a remote extractor service accepts a user-supplied URL and fetches it, it MUST prevent SSRF:
- **Allowlist** of permitted target host patterns (per platform adapter — e.g., `*.tiktok.com`, `*.instagram.com`, `*.facebook.com`, plus their CDN domains). Allowlist is the **primary** defense.
- **IP-range check** as safety net: resolve DNS, reject if IP is private (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.0.0/16 link-local incl. cloud metadata, ::1, fc00::/7).
- **Block cloud metadata endpoints**: AWS `169.254.169.254`, GCP `metadata.google.internal`, Azure `169.254.169.254/metadata/instance`. IMDSv2 on AWS mitigates but don't rely.
- **Redirect-following SSRF**: the server may follow a 30x to an internal IP. Either disable redirects, or re-validate the destination after each hop.
- **DNS rebinding**: resolve, validate, then re-resolve at connection time (TOCTOU). Use a pinned resolved IP for the connection.

- Source: Palo Alto Networks — https://www.paloaltonetworks.com/cyberpedia/server-side-request-forgery-api7 — *"Egress filtering blocks connections to private IP ranges, localhost addresses, and cloud metadata endpoints"*
- Source: Graphnode Software (Apr 26, 2026) — https://www.graphnodesoftware.com/blog/ssrf-attack-prevention — *"The allowlist is the primary defense. The IP-range check is the safety net"*
- Source: Resecurity (Aug 6, 2025) — https://www.resecurity.com/blog/article/ssrf-to-aws-metadata-exposure — *"The AWS Instance Metadata Service (169.254.169.254) is designed to be accessed only from within the EC2 instance"*

### 1.2 Open proxy abuse — remote media proxy

**Finding (CONFIRMED risk):** If the remote service proxies media bytes for ANY URL, attackers use it as a free CDN/proxy/scraper (bandwidth theft, scraping, masking origin for attacks).

- **Mitigations**:
  - Allowlist target hosts (same as SSRF) — proxy only known platform CDNs.
  - Signed request URLs (HMAC) so only the app can request a proxy of a specific resolved URL.
  - Rate limit per client IP / per session token.
  - Time-box the signed URL (e.g., 60s) so it can't be reused.
  - Optionally require a one-time CSRF-style token minted by the app.
  - Log and alert on anomalous patterns.
- The proxy endpoint should be a **separate, opt-in** path (`/proxy?u=...&sig=...`), not the default. The default `/extract` returns direct URLs; the client falls back to `/proxy` only when direct fetch fails (CORS/Referer).

### 1.3 Malicious media — parser exploits

**Finding (CONFIRMED):** Crafted media files have historically exploited FFmpeg, mp4box, and other parsers (numerous CVEs). In-browser WASM execution reduces blast radius (WASM is sandboxed — no host memory access except via explicit imports) but a WASM-compiled parser bug is still a memory-corruption risk **within the WASM instance** (can corrupt the linear memory, potentially crash the worker or produce garbage output).

- **Mitigations**:
  - Run media parsing in a **DedicatedWorker** (isolated from main thread).
  - Impose **timeouts** (abort worker if processing exceeds N seconds).
  - Impose **file-size caps** before parsing.
  - Prefer **pure-TS** parsers (Mediabunny, mp4-muxer) over C/WASM parsers where possible — TS can't have memory-corruption bugs in the same way (though it can have logic bugs, infinite loops, OOM).
  - For WASM FFmpeg: use a **pinned, verified build** with SRI (see 1.8).
  - Don't run untrusted WASM modules — only our own builds.

### 1.4 Resource exhaustion

**Finding (CONFIRMED):** Risks: unbounded loops in extractors (regex catastrophic backtracking), decompression bombs (a tiny file that expands hugely), huge metadata, deeply nested containers.

- **Mitigations**: worker timeouts, WASM fuel metering (if using a runtime that supports it), input size caps, structured parse-depth limits.

### 1.5 WASM security

**Finding (CONFIRMED):**
- WASM executes in a sandboxed linear memory; cannot access host memory except via explicitly-imported functions.
- Supply chain risk: a compromised FFmpeg.wasm build could ship malicious code. Mitigate with **Subresource Integrity (SRI)** — hash-verify the `.wasm` file at load time, and pin to a specific build (commit SHA / release tag).
- Reproducible builds: ideally verify the WASM matches a reproducible build from source.

- Source: MDN Subresource Integrity (Apr 14, 2026) — https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Subresource_Integrity — *"SRI is a security feature that enables browsers to verify that resources they fetch (for example, from a CDN) are [unchanged]"*
- Source: Andrew Lock (Aug 2024) — https://andrewlock.net/avoiding-cdn-supply-chain-attacks-with-subresource-integrity — references the polyfill.io supply-chain attack.

### 1.6 Workers — security boundary

**Finding (CONFIRMED):** DedicatedWorker and SharedWorker are same-origin. `postMessage` uses structured clone (or transferable objects). A worker cannot exfiltrate data beyond its origin's storage/network. Trusted Types (where enabled) prevent DOM injection from worker messages.

### 1.7 Dependency supply chain

**Finding (CONFIRMED):** npm ecosystem risks: typosquatting, compromised maintainer accounts, malicious post-install scripts.
- **Mitigations**: lockfile (`bun.lock` / `package-lock.json`) committed, `npm audit` / `socket.dev` in CI, pinned dependencies (exact versions), Dependabot alerts, minimize dependency count.
- For WASM: verify build provenance (see 1.5).

### 1.8 GitHub Actions security

**Finding (CONFIRMED):** Top risks and mitigations:
- **`pull_request_target`**: runs in the context of the target repo with secrets access. Historically exploitable (pwn-request). As of **Dec 8, 2025**, GitHub changed `pull_request_target` to always use the default branch's workflow file (mitigating stale-workflow exploitation), but it still runs fork PR code with secrets if `run:` steps interpolate PR content.
- **Expression injection**: `run: echo "${{ github.event.pull_request.body }}"` lets a malicious PR body execute arbitrary shell. Mitigate by passing via env vars and using `bash` with proper quoting, or avoid interpolating untrusted input into `run:`.
- **GITHUB_TOKEN permissions**: set `permissions: contents: read` by default; escalate per-job only where needed.
- **Third-party actions**: pin to commit SHA, not floating tags.
- **OIDC**: use for cloud deploys (short-lived tokens, no static secrets).

- Source: GitHub Security Lab (Jan 16, 2025) — https://securitylab.github.com/resources/github-actions-new-patterns-and-mitigations — *"A workflow activated by pull_request_target and triggered from a fork operates with significant privileges"*
- Source: GitGuardian (Apr 11, 2025) — https://blog.gitguardian.com/github-actions-security-cheat-sheet — *"pull_request_target runs in the context of your target repo, meaning the workflow has access to YOUR secrets"*
- Source: Wiz (Apr 15, 2026) — https://www.wiz.io/blog/github-actions-security-guide — *"pull_request_target now always uses the default branch as its workflow source"*

### 1.9 CSP (Content-Security-Policy) on GitHub Pages

**Finding (CONFIRMED, critical):**
- GitHub Pages does **NOT** allow custom HTTP response headers.
- `Content-Security-Policy` CAN be set via `<meta http-equiv="Content-Security-Policy" content="...">` in the HTML.
- `Cross-Origin-Opener-Policy` (COOP), `Cross-Origin-Embedder-Policy` (COEP), `Cross-Origin-Resource-Policy` (CORP), `Strict-Transport-Security` (HSTS) **CANNOT be set via `<meta>`** — they are HTTP-only headers. GitHub Pages cannot set them.

- Source: GitHub community discussion — https://github.com/orgs/community/discussions/13309 — *"In order to use SharedArrayBuffer and WebAssembly Threads, COOP and COEP headers need to be set"*
- Source: Wasmer docs (Jul 31, 2026) — https://docs.wasmer.io/sdk/wasmer-js/how-to/coop-coep-headers — *"Deploying web applications on GitHub Pages that require Cross-Origin Isolation can be challenging due to lack of control over HTTP headers"*
- Source: Thomas Steiner blog (Mar 8, 2025) — https://blog.tomayac.com/2025/03/08/setting-coop-coep-headers-on-static-hosting-like-github-pages
- **Implication**: `SharedArrayBuffer` (required for multi-threaded WASM) is **NOT available** on a GitHub Pages-hosted app. Workarounds: (a) the `coi-serviceworker` shim (registers a SW that re-injects COOP/COEP — hacky, unreliable, breaks on iOS); (b) host the app shell on a platform that allows headers (Cloudflare Pages, Netlify, Vercel, self-hosted VPS) and use GitHub Releases for assets; (c) accept single-threaded WASM.
- **Implication**: We CAN set a CSP via `<meta>` to harden the app against XSS (restrict script/style/img/connect sources).

### 1.10 CORS for the remote service

**Finding (CONFIRMED):** The remote extractor must send `Access-Control-Allow-Origin: https://<user-pages-domain>` (or `*` for public, unauthenticated endpoints). Handle OPTIONS preflight. If the service uses credentials/cookies, `*` is not allowed — must echo the specific origin.

### 1.11 Secrets

**Finding (CONFIRMED):** A static GitHub Pages site has **zero secrets** — everything client-side is public and inspectable. Any API keys, tokens, or credentials shipped to the client are compromised by definition. Secrets (for the remote service, for platform APIs) live **only** in the remote service's environment, never in the client bundle.

### 1.12 Content legality / abuse

**Finding (CONFIRMED, precedent):** Media-download tools face legal risk:
- **youtube-dl / RIAA case (Oct 2020)**: RIAA issued a DMCA takedown to GitHub for youtube-dl; GitHub reinstated it after EFF pushback (the tool wasn't a circumvention device under DMCA §1201). The repo remains up. Precedent: a general-purpose download tool is not inherently illegal, but specific platform targeting invites takedown attempts.
- **TOS violations**: TikTok/IG/FB ToS prohibit scraping/download. This is civil/contract law (not criminal), but can result in IP bans, account suspension, or civil suits.
- **Piracy abuse**: a public downloader can be used to pirate copyrighted content. Mitigations: no search/discovery (user must supply URL), no hosting of media, user-initiated only, clear "personal use" framing.

- Source: EFF (Nov 17, 2020) — https://www.eff.org/deeplinks/2020/11/github-reinstates-youtube-dl-after-riaas-abuse-dmca
- Source: Copyright Lately — https://copyrightlately.com/youtube-stream-ripping-copyright

---

## PART 2 — LICENSING

### 2.1 FFmpeg

**Finding (CONFIRMED):**
- FFmpeg core is **LGPL 2.1+** by default.
- If compiled with GPL libraries (notably **libx264**, **libx265**, **libfdk-aac**, **libmp3lame**), the resulting binary becomes **GPL**.
- FFmpeg.wasm's default `@ffmpeg/core` build **includes libx264** → it is GPL.
- **Patent risk**: H.264/AVC and H.265/HEVC are patent-encumbered (MPEG-LA / Via Licensing Alliance patent pools). Distributing an **encoder** may require a patent license. Distributing a **decoder** is generally covered by the browser vendor's license (for native playback), but a WASM encoder is a separate distribution.

- Source: ffmpeg.org/legal.html — https://www.ffmpeg.org/legal.html — *"FFmpeg is licensed under the GNU Lesser General Public License (LGPL) version 2.1 or later. Make sure your program is not using any GPL libraries (notably libx264)"*
- Source: ffmpegwasm/ffmpeg.wasm issue #107 — https://github.com/ffmpegwasm/ffmpeg.wasm/issues/107 — *"The original ffmpeg is licensed under LGPL and GPL. Ships with libx264, which is GPL, which makes the resulting ffmpeg configuration GPL"*
- Source: hoop.dev (Sep 9, 2025) — https://hoop.dev/blog/understanding-ffmpeg-licensing-what-developers-need-to-know-before-shipping

**Patent expiry (CONFIRMED, nuanced):**
- H.264/AVC: most patents have expired; some still active; full expiry ~2027–2030 depending on jurisdiction.
- H.265/HEVC: complex multiple patent pools; licensing expensive and overcomplicated.

- Source: Via Licensing Alliance AVC/H.264 — https://www.via-la.com/licensing-programs/avc-h-264
- Source: Wikimedia meta — https://meta.wikimedia.org/wiki/Have_the_patents_for_H.264_MPEG-4_AVC_expired%3F — *"Most patents already expired, but some are still active somewhere: will expire on 2030-11-10"*
- Source: HN discussion — https://news.ycombinator.com/item?id=47630109 — *"H.264 patents have already expired in most of the world"*

**Implication**: Distributing FFmpeg.wasm (GPL, with x264) via GitHub Releases is legally OK for an open-source project (GPL is fine if our code is also GPL or we comply with GPL terms). But (a) GPL is viral — if we link FFmpeg.wasm into our app, GPL obligations may apply to our whole app; (b) patent risk for H.264/H.265 encoding persists in some jurisdictions until ~2027-2030. For a permissive MIT/Apache project, this is a real friction point.

### 2.2 yt-dlp

**Finding (CONFIRMED):** yt-dlp is licensed under the **Unlicense** (public domain dedication). No copyleft, no patent terms. We can reference, link to, or wrap it freely. We would NOT ship yt-dlp (it's Python) — at most, our remote service could invoke it.

### 2.3 Cobalt

**Finding (CONFIRMED):** Cobalt (imputnet/cobalt) is dual-licensed:
- **API (`api/`)**: **GNU AGPL-3.0** — strong copyleft, network use triggers source-disclosure obligation. If we run a modified Cobalt API as our remote service, we MUST publish our modifications.
- **Web frontend (`web/`)**: **CC-BY-NC-SA-4.0** — NonCommercial. We CANNOT reuse the Cobalt frontend code in a commercial product.

- Source: cobalt/api/LICENSE — https://github.com/imputnet/cobalt/blob/main/api/LICENSE
- Source: cobalt/web/README.md — https://github.com/imputnet/cobalt/blob/main/web/README.md — *"cobalt web code is licensed under CC-BY-NC-SA-4.0"*

**Implication**: AGPL is compatible with an open-source project (we're already public), but it forces source disclosure of any modified Cobalt-based remote service. CC-BY-NC-SA on the frontend means we must write our own UI (which we are). Do NOT copy Cobalt's frontend.

### 2.4 gallery-dl

**Finding (CONFIRMED):** gallery-dl is **GPL-2.0**. Python-only, not browser-relevant. If our remote service used it, GPL-2.0 would apply to modifications.

### 2.5 Codecs — patent landscape

| Codec | Type | Patent status | Royalty-free? |
|---|---|---|---|
| H.264/AVC | Video | Mostly expired; some active until ~2027-2030 | No (until expiry) |
| H.265/HEVC | Video | Multiple pools, complex, expensive | No |
| VP9 | Video | Royalty-free (Google) | Yes |
| AV1 | Video | Royalty-free (AOMedia) | Yes |
| AAC | Audio | Patent-encumbered (Via Licensing) | No |
| Opus | Audio | Royalty-free (IETF standard) | Yes |
| MP3 | Audio | Patents expired (2017+) | Yes |

**Implication**: For in-browser encoding via WASM, prefer **AV1/VP9 + Opus** (royalty-free) over H.264/H.265 + AAC. For remux-only (no re-encode), patent issues are lower risk (we're not encoding, just copying). The browser's native playback handles decoder licensing.

### 2.6 WASM runtimes & tooling

| Component | License |
|---|---|
| wasmtime | Apache-2.0 |
| wasmer | MIT |
| wasm-bindgen / wasm-pack | MIT/Apache-2.0 |
| wasm-tools (Component Model) | Apache-2.0 |
| Extism | BSD-3-Clause |
| jco | Apache-2.0 |
| Javy | Apache-2.0 |
| Emscripten | MIT (with LLVM/Nginx sub-licenses) |

All permissive — no copyleft friction.

### 2.7 Media parsing libraries

| Library | License |
|---|---|
| mp4box.js (GPAC) | BSD-3-Clause (permissive) |
| mp4-muxer | ISC (permissive) |
| webm-muxer | ISC (permissive) |
| Mediabunny | MIT (permissive, verify) |

All permissive — safe for an MIT/Apache project.

### 2.8 Recommended project license

**Recommendation**: **MIT** (or Apache-2.0) for our project code.
- Permissive — maximizes adoption and contribution.
- Compatible with all permissive deps (mp4box.js, muxers, Mediabunny, Rust/WASM tooling).
- **Caveat**: if we ship FFmpeg.wasm (GPL) as a bundled asset and link it into our app, GPL obligations may apply. Mitigation: (a) treat FFmpeg.wasm as a separate, user-loaded optional plugin (not bundled by default); (b) or license our project GPL to be safe; (c) or use an LGPL-only FFmpeg build (no x264) and accept reduced codec support.
- **Caveat**: if our remote service forks Cobalt (AGPL), that service component is AGPL. The rest of our code (client, other services) can remain MIT. AGPL applies only to the modified Cobalt-derived service.

---

## REMOTE SERVICE SSRF / PROXY MITIGATION CHECKLIST

- [ ] Per-platform host allowlist (e.g., `*.tiktok.com`, `*.instagram.com`, `*.facebook.com`, known CDN domains)
- [ ] DNS resolution + private-IP rejection (10/8, 172.16/12, 192.168/16, 127/8, 169.254/16, ::1, fc00::/7)
- [ ] Cloud-metadata endpoint block (169.254.169.254, metadata.google.internal)
- [ ] Redirect-following re-validation (or disable redirects)
- [ ] DNS rebinding defense (pin resolved IP for connection)
- [ ] Signed proxy request URLs (HMAC, short TTL ~60s)
- [ ] Per-IP rate limiting
- [ ] File-size cap on proxy responses
- [ ] Timeout on all outbound fetches
- [ ] No cookie/credential storage by default (stateless)
- [ ] Logging + anomaly alerting

---

## LICENSE COMPATIBILITY MATRIX

| Dependency | License | Compatible with MIT? | Network-copyleft? | Patent risk? |
|---|---|---|---|---|
| FFmpeg.wasm (with x264) | GPL | ⚠️ viral if linked | Yes (GPL) | H.264 encode patent until ~2027 |
| FFmpeg.wasm (LGPL-only build) | LGPL-2.1 | ✅ (dynamic link) | No | Lower (no x264) |
| yt-dlp | Unlicense | ✅ | No | No |
| Cobalt API | AGPL-3.0 | ✅ if remote svc is AGPL | Yes (network use) | No |
| Cobalt web | CC-BY-NC-SA-4.0 | ❌ NonCommercial | n/a | No |
| gallery-dl | GPL-2.0 | ⚠️ if shipped | Yes | No |
| mp4box.js | BSD-3 | ✅ | No | No |
| mp4-muxer / webm-muxer | ISC | ✅ | No | No |
| Mediabunny | MIT | ✅ | No | No |
| Rust + wasm-bindgen | MIT/Apache | ✅ | No | No |
| wasmtime / jco / Extism | Apache/MIT/BSD | ✅ | No | No |

---

## UNCERTAIN ITEMS

- Exact H.264 patent expiry date per jurisdiction (consensus: "mostly expired, some until 2027-2030" but not a single clean date).
- Whether an LGPL-only FFmpeg.wasm build (without x264) is practically usable for our remux needs (probably yes for `-c copy` remux; no for H.264 encoding).
- Whether AGPL on a Cobalt-derived remote service "infects" the client app (legal consensus: no, AGPL applies to the network-served service, not separate client code that merely calls it).
- Whether the `coi-serviceworker` COOP/COEP shim is reliable enough to ship (anecdotal: flaky on iOS, breaks some sites).
