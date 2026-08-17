# Experiment 1 — GitHub Releases CORS — EXECUTED

**Date:** 2026-08-17
**Status:** EXECUTED (not simulated)
**Classification:** NON-BLOCKING for architecture (fallback existed), but RUN because cheap/safe and reduces real uncertainty.

## Question
Does `release-assets.githubusercontent.com` (and the `github.com/.../releases/download/...` redirect) send `Access-Control-Allow-Origin` allowing a browser `fetch()` from a `*.github.io` origin?

## Setup
1. Created a temporary GitHub Release on `Alot1z/open-media-tools` (tag `cors-probe`, prerelease).
2. Uploaded a 24-byte test asset `cors-probe.txt`.
3. `browser_download_url`: `https://github.com/Alot1z/open-media-tools/releases/download/cors-probe/cors-probe.txt`
4. Probed unauthenticated (as a browser would) with `curl -sIL -H "Origin: https://alot1z.github.io"`.
5. Also probed jsDelivr npm and unpkg npm for the proposed fallback.
6. Cleaned up: deleted the release and tag (verified 0 releases / 0 tags after).

## Results (raw headers)

### Probe 1: GitHub Releases direct fetch (the thing under test)
```
HTTP/2 302   (from github.com)
  location: https://release-assets.githubusercontent.com/github-production-release-asset/...
  cache-control: no-cache
  x-frame-options: deny
  x-content-type-options: nosniff
  >>> NO access-control-allow-origin header <<<

HTTP/2 200   (from release-assets.githubusercontent.com, Azure Blob / Fastly)
  content-type: application/octet-stream
  content-length: 24
  x-ms-blob-type: BlockBlob
  x-cache: MISS, MISS
  >>> NO access-control-allow-origin header <<<
```

### Probe 2: CORS preflight OPTIONS on the release-download URL
```
HTTP/2 404   (github.com does not handle OPTIONS on the release-download path)
```

### Probe 3: jsDelivr npm (proposed fallback) — @ffmpeg/core WASM
```
HTTP/2 200
  access-control-allow-origin: *
  access-control-expose-headers: *
  cache-control: public, max-age=31536000, s-maxage=31536000, immutable
  content-type: application/wasm
  x-cache: HIT, HIT
```

### Probe 4: unpkg npm (alternative fallback) — @ffmpeg/core WASM
```
HTTP/2 200
  access-control-allow-origin: *
  access-control-allow-headers: *
  access-control-allow-methods: GET, HEAD, OPTIONS
  access-control-expose-headers: *
  cache-control: public, max-age=31536000
  content-type: application/wasm
  server: cloudflare
```

### Probe 5: jsDelivr gh-repo mirror of a release asset
```
HTTP/2 404   (jsDelivr /gh/ mirrors repo files at a path, NOT release assets)
  access-control-allow-origin: *   (CORS present, but 404 — wrong path for releases)
```

## Conclusion

**Experiment 1 RESULT: FAILS for direct GitHub Releases browser fetch.**

- GitHub Releases (`github.com/.../releases/download/...` → `release-assets.githubusercontent.com`) does **NOT** send `Access-Control-Allow-Origin`. A browser `fetch()` from a `*.github.io` origin is **CORS-blocked** (the browser receives the bytes but refuses to expose them to JS).
- The CORS preflight (OPTIONS) also fails (404) — so even non-simple requests (e.g., with a `Range` header) are blocked at preflight.
- **jsDelivr npm** (`cdn.jsdelivr.net/npm/@ffmpeg/core@VERSION/dist/...`) sends `access-control-allow-origin: *`, correct `content-type: application/wasm`, and `cache-control: immutable` (1 year). **This is the working distribution path.**
- **unpkg npm** is an equivalent working alternative (Cloudflare-backed).
- **jsDelivr gh mirror** does NOT mirror release assets (only repo files) — so we cannot mirror our own release assets via jsDelivr gh; we must use npm packages via jsDelivr npm.

## Architectural impact (honest, evidence-based)

This is a **NEW ARCHITECTURAL DECISION** driven by the experiment result (Pass 3 marked jsDelivr as OPTIONAL "if Exp 1 fails"; Exp 1 DID fail):

- **NEW-AD-A: FFmpeg.wasm escape-hatch distribution via jsDelivr npm CDN (USE), not GitHub Releases.** The browser fetches `@ffmpeg/core` from `https://cdn.jsdelivr.net/npm/@ffmpeg/core@<pinned-version>/dist/umd/ffmpeg-core.wasm` with SRI hash verification. GitHub Releases is demoted from "large browser-fetched asset host" to "source tarballs + release metadata only."
- **GitHub Releases role revised (AD-20/AD-21 caveat):** still USE, but NOT for browser-fetched WASM. Used for: source-code tarballs, release notes/version metadata, and any non-browser-fetched artifacts (e.g., CI-internal).
- **Custom FFmpeg.wasm builds (future, LGPL-only):** if we ever build a custom FFmpeg.wasm (e.g., LGPL-only without x264), it must be published as an npm package (or hosted on a CORS-friendly static host) to be browser-fetchable. GitHub Releases alone won't work. This is a USE-LATER concern.

## Confidence
HIGH — empirically measured against the actual user's repo with real release assets and real CDN responses. Not assumed.

## What would change this decision
- If GitHub begins sending `Access-Control-Allow-Origin` on release-asset responses (unlikely; would require a platform change), we could revert to direct Releases fetch. Re-test periodically.
- If jsDelivr/unpkg become unreliable or change CORS policy, switch to the other, or self-host on a CORS-friendly static host (Cloudflare Pages, Netlify, or a VPS with a `Access-Control-Allow-Origin: *` header).
