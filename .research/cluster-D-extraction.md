# Cluster D — Extraction Tools & Browser-Native Extraction Feasibility

**Task ID:** 1-D
**Scope:** yt-dlp / Cobalt / gallery-dl / other extraction tools; browser-native extraction feasibility for TikTok, Instagram, Facebook; minimal viable remote extractor spec.
**Method:** Web search via z-ai web_search (12 successful searches; 6+ attempts rate-limited).
**Date:** 2026 (current).

---

## Part 1 — Tools

### 1.1 yt-dlp

- **Repository:** https://github.com/yt-dlp/yt-dlp
- **Current version (2025/2026):** Latest stable release dated **Dec 7, 2025** per the GitHub releases page. [source: https://github.com/yt-dlp/yt-dlp/releases]
- **License:** **Unlicense** (public-domain equivalent) for the yt-dlp code itself. The Unix executable bundles code from other projects under ISC and MIT licenses. [source: https://github.com/yt-dlp/yt-dlp] [source: https://en.wikipedia.org/wiki/Youtube-dl]
- **Extractor count:** Marketed as supporting "over 1,000 websites" / "thousands of sites" as of Aug 2025. The widely-cited "1800+" figure refers to total extractor entries (some sites have multiple extractors). [source: https://www.ditig.com/yt-dlp-cheat-sheet] [source: https://man.archlinux.org/man/extra/yt-dlp/yt-dlp.1.en] [source: https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md]
- **Python dependency:** Requires Python 3.9+. Python 3.9 reached EOL Oct 2025; yt-dlp has **dropped Python 3.9 support** as of late 2025. [source: https://www.videohelp.com/software/yt-dlp/version-history]
- **Browser feasibility:** **None.** yt-dlp is a Python CLI + per-site extractor classes. It cannot run in a browser. No WASM/Pyodide port of the extractor layer exists (and would still need native networking and TLS fingerprint handling).
- **JS wrappers:** `youtube-dl-exec` (microlinkhq), `yt-dlp-exec` (borodutch-labs), `yt-dlp-exec` on npm — all wrap the **binary** via `child_process`; they require Python + the yt-dlp binary on the host. There is **no pure-JS port** of yt-dlp. [source: https://github.com/microlinkhq/youtube-dl-exec] [source: https://classic.yarnpkg.com/en/package/yt-dlp-exec] [source: https://www.npmjs.com/search?q=keywords:yt-dlp]
- **How extractors work:** Per-site Python classes under `yt_dlp/extractor/`, each implementing `_real_extract(url)` returning a dict of media formats + metadata. Sites are matched by URL pattern. [source: https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md]
- **Maintenance burden:** High. Sites change frequently; YouTube extractor breakages and 403 issues are common (issues #12561 Mar 2025, #13552, #13804 Jul 2025, #16477 Apr 2026). Active nightly release cadence to keep up. [source: https://github.com/yt-dlp/yt-dlp/issues/12561] [source: https://github.com/yt-dlp/yt-dlp/issues/13804] [source: https://github.com/yt-dlp/yt-dlp/issues/16477]
- **Legal/DMCA history:** youtube-dl (the predecessor) was DMCA-taken-down by RIAA in Oct 2020, reinstated by GitHub/EFF in Nov 2020 after pushback on §1201 circumvention claims. yt-dlp itself has not been directly targeted, but the precedent is real for any extractor maintainer. [source: https://github.blog/news-insights/policy-news-and-insights/standing-up-for-developers-youtube-dl-is-back] [source: https://www.eff.org/deeplinks/2020/11/github-reinstates-youtube-dl-after-riaas-abuse-dmca]
- **Self-host scenario:** Run yt-dlp binary server-side behind a small HTTP API (this is the most common pattern; e.g., MeTube, ytdl-org web UIs).

### 1.2 Cobalt (cobalt.tools / imputnet/cobalt)

- **Repository:** https://github.com/imputnet/cobalt
- **License:** API server (processing backend) — **AGPL-3.0**. Frontend — **CC-BY-NC-SA-4.0** ("source-first", **NOT** OSI-approved open source due to the NonCommercial clause; derivative frontends cannot be commercialized). [source: https://cobalt.tools/about/credits]
- **Public API status:** The public v7 API (`api.cobalt.tools/api/json`) was **shut down on November 11, 2024** due to scraping pressure and rising anti-bot countermeasures. The web UI `cobalt.tools` itself remained operational. [source: https://github.com/imputnet/cobalt/discussions/860] [source: https://github.com/imputnet/cobalt/discussions/876]
- **Self-host requirements:** Docker image or bare Node.js. Configuration via env: `API_AUTH_REQUIRED=1`, `API_KEY_URL` to restrict to authorized users. Runs on a configurable port. [source: https://railway.com/deploy/cobalt-self-hosted-fix-youtube-downloads-on-railway--cobalt-youtube-downloader]
- **How it returns media:** Two response modes — (a) redirect to a **direct service URL** (the CDN URL), or (b) a **proxied/remuxed/transcoded stream** served by cobalt itself. [source: https://github.com/imputnet/cobalt/blob/main/docs/api.md]
- **Own extraction vs. yt-dlp proxy:** Cobalt has its **own per-service scrapers** in JavaScript (under `src/core/services/`); it does **not** shell out to yt-dlp. This is important: it means cobalt's coverage ≠ yt-dlp's coverage, and cobalt is independently affected by site breakages. [UNCERTAIN — inferred from code structure and absence of Python in deployment; no official doc explicitly says "no yt-dlp". Should be verified directly from the repo.]
- **Rate limits / auth:** Public community instances require API keys (the `cobalt.tools/settings/instances` page says "make sure the instance supports api keys!"). Instance hosters can set their own rate limits. [source: https://cobalt.tools/settings/instances]
- **Recent controversy (v10/v11):** The v7 shutdown was the major controversy. The railway.com deploy article (Jun 7, 2026) reports that "the public cobalt.tools instance no longer works for YouTube — it was blocked by YouTube's anti-downloader measures and remains blocked as of [2026]." Self-hosted instances with rotated cookies/proxies still work. [source: https://railway.com/deploy/cobalt-self-hosted-fix-youtube-downloads-on-railway--cobalt-youtube-downloader]
- **Local processing (v11.2, Jun 30, 2025):** Cobalt added client-side processing features. [source: https://cobalt.tools/updates]

### 1.3 gallery-dl

- **Repository:** https://github.com/mikf/gallery-dl
- **License:** **GPL-2.0**. [source: https://github.com/mikf/gallery-dl]
- **Language:** Python. Latest PyPI release: 1.3.5. [source: https://pypi.org/project/gallery-dl/1.3.5]
- **Focus:** Image galleries (pixiv, exhentai, deviantart, twitter, reddit, instagram, etc.). 200+ sites.
- **Browser feasibility:** **None** — requires Python runtime; no JS port.

### 1.4 Other tools / alternatives

- **youtube-dl:** Original predecessor to yt-dlp; effectively unmaintained in comparison.
- **NewPipe:** Android-only (Java/Kotlin); not browser-relevant. [UNCERTAIN — not searched]
- **MeTube:** Self-hosted YouTube downloader (Node + yt-dlp); listed as a Cobalt alternative. [source: https://www.reddit.com/r/selfhosted/comments/1en5fg8/i_tried_some_of_the_many_youtube_downloaders]
- **Cobalt forks / community instances:** Many decentralized hosts; listed on cobalt.tools tracker. [source: https://www.reddit.com/r/cobalt_tools]
- **Sunfishcobalt:** [UNCERTAIN — no search results returned; possibly a small fork or misnamed. Needs direct GitHub search.]
- **inv.tux.pizza:** [UNCERTAIN — no search results returned; appears to be an Invidious-style instance, not directly an extractor.]
- **Vette1123/social-media-downloader:** Multi-platform (TikTok, Twitter, Instagram, Facebook, YouTube, Pinterest, Reddit, Threads, Snapchat, Twitch, Vimeo) open-source downloader. [source: https://github.com/Vette1123/social-media-downloader]
- **sh13y/Facebook-Video-Download-API:** Production-ready FB video API. [source: https://github.com/sh13y/Facebook-Video-Download-API]

---

## Part 2 — Browser-Native Extraction Feasibility

### 2.1 What can a browser extract without a backend?

For TikTok / Instagram / Facebook, the layered questions are:

#### (a) Can the page HTML be fetched cross-origin?

**No.** None of these platforms send `Access-Control-Allow-Origin` headers permitting arbitrary third-party origins. A `fetch()` from `our-app.com` to `tiktok.com/.../video/...` will be blocked by the browser's CORS policy before the JS can read the body. This is repeatedly confirmed across FB dev forums, Stack Overflow, and Reddit webdev discussions. [source: https://developers.facebook.com/community/threads/486109443829984] [source: https://www.reddit.com/r/webdev/comments/1n82usz/how_do_so_many_media_downloader_websites_manage]

#### (b) Can we parse oEmbed?

- **TikTok oEmbed:** Public endpoint `https://www.tiktok.com/oembed?url=...` — **no auth required**. BUT it returns only embed HTML, thumbnail URL, author, and title — **NOT direct media URLs**. Insufficient on its own for downloading the video bytes. [source: https://github.com/iamcal/oembed/blob/master/providers/TikTok.yml] [source: https://stackoverflow.com/questions/62767867/embed-video-from-tiktok]
- **Meta oEmbed (Instagram, Facebook, Threads):** As of **June 15, 2026**, Meta reversed its 2020 decision and made oEmbed endpoints **tokenless** (no access token, no App Review required). HOWEVER, oEmbed returns embed HTML/thumbnail/title/author only — **not the direct CDN media URLs**. So even with tokenless access, oEmbed does not solve the extraction problem for downloading media bytes. [source: https://developers.facebook.com/blog/post/2026/06/15/tokenless-access-to-meta-oembed-apis] [source: https://wpmayor.com/meta-tokenless-oembed-wordpress] [source: https://spotlightwp.com/instagram-embed-wordpress]
  - Note: For apps created before certain dates, prior app-review requirements applied; the June 2026 change is a real and significant reversal but does not change what oEmbed *returns*.

#### (c) Can we scrape embedded JSON?

- **TikTok:** The page HTML contains a `<script id="__UNIVERSAL_DATA_FOR_REHYDRATION__" type="application/json">` block with the full page data including the video's direct CDN URL. This is the standard extraction vector used by TikTok-Api (davidteather) and scrapers. **Problem:** the HTML is cross-origin and cannot be fetched by browser JS without an extension or proxy. [source: https://dev.to/hasdata_com/scraping-tiktok-profiles-and-videos-with-python-part-1-4o8i] [source: https://hasdata.com/blog/tiktok-scraping-python] [source: https://github.com/davidteather/TikTok-Api/blob/main/TikTokApi/api/video.py]
- **Instagram:** `window._sharedData` was removed years ago. Instagram now uses GraphQL endpoints with **signed headers** (X-IG-App-Id, X-Asbd-Id, etc.). Multiple new scraping defenses landed in 2024–2025: datacenter IPs are blocked on first request, Python `requests`/`httpx` are fingerprinted and blocked. Reverse-engineered GraphQL still works from residential IPs with proper headers, but **not from arbitrary browser origins**. [source: https://www.socialcrawl.dev/blog/instagram-scraping-2026] [source: https://scrapfly.io/blog/posts/how-to-scrape-instagram] [source: https://medium.com/@seotanvirbd/how-i-built-a-python-tool-that-extracts-instagram-reel-data-without-authentication-api-keys-or-0fcb35cba7b7]
- **Facebook:** Similar GraphQL signing; video pages require session cookies for most content.

#### (d) Are direct CDN URLs CORS-accessible from the browser?

- **TikTok CDN** (`tiktokcdn.com`, `tiktok.com`, `*.akamaized.net`, `*.bytecdntp.com`): Video URLs are **signed and short-lived** (expire within minutes to hours). They often return **403 Forbidden** without the proper Referer/cookies — confirmed by multiple TikTok-Api issues and Stack Overflow posts. [source: https://gist.github.com/devinschumacher/86d2843f0e1f150d79356494bc0b1a1a] [source: https://github.com/davidteather/TikTok-Api/issues/265] [source: https://stackoverflow.com/questions/67746978/getting-error-403-forbidden-despite-using-headers-what-could-be-going-wrong]
- **Instagram CDN** (`scontent-*.cdninstagram.com`, `*.cdninstagram.com`): Frequently requires session cookies; yt-dlp users routinely hit "Requested content is not available" errors without `--cookies-from-browser`. [source: https://github.com/yt-dlp/yt-dlp/issues/4750]
- **Facebook CDN** (`video-*.fbcdn.net` / `fnvtx-*` / `sbull-*` subdomains): Often signed and expiring; many require Referer/cookies.

#### (e) Referer header — can the browser set it?

**No.** `Referer` is a **forbidden request header** per the Fetch standard — JavaScript cannot set it via `fetch()` or `XMLHttpRequest`. The browser can only influence it indirectly via `referrerPolicy` (which controls how the auto-generated Referer is shaped, e.g., `no-referrer`, `origin-only`), but cannot spoof `Referer: https://www.tiktok.com/` from a page on `our-app.com`. This breaks direct browser fetch of CDN URLs that gate on Referer. [source: https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_request_header] [source: https://developer.mozilla.org/en-US/docs/Web/API/Request/referrerPolicy] [source: https://github.com/mdn/content/issues/2660]

#### (f) Cookies / auth

Cookies for `tiktok.com` (the user's logged-in session) **cannot** be sent on a `fetch` from `our-app.com` — different origins. Only two paths can use real-user cookies:
1. A **browser extension** with `host_permissions` for `*://*.tiktok.com/*` can read the page DOM and reuse cookies cross-origin.
2. A **remote service** where the user pastes their session cookie (security risk; better: use the remote service's own residential IP + bot-mitigation).

For private/restricted content, the platform session is mandatory. [source: https://discourse.mozilla.org/t/host-permissions-not-allowing-cors-requests/106959] [source: https://www.kromio.ai/blog/chrome-extension-cross-origin-requests]

#### (g) Anti-bot

Cloudflare, PerimeterX/HUMAN, and Akamai bot detection are active on all three platforms. Browser `fetch` from our app origin will be challenged (JS challenge, TLS fingerprinting, behavioral signals). A real user's browser session passes, but **we cannot reuse that session cross-origin without an extension**. [source: https://www.browserless.io/blog/bypass-cloudflare-with-puppeteer] [source: https://kameleo.io/blog/how-to-bypass-cloudflare-with-puppeteer]

### 2.2 CORS reality and workarounds

| Workaround | Can read body? | Notes |
|---|---|---|
| (a) Browser extension w/ `host_permissions` | **Yes** | Extension's background service worker can issue `fetch` to any permitted origin without CORS restrictions. [source: https://discourse.mozilla.org/t/host-permissions-not-allowing-cors-requests/106959] [source: https://www.kromio.ai/blog/chrome-extension-cross-origin-requests] |
| (b) Remote proxy adding CORS headers | **Yes** | Standard pattern; server fetches the URL and returns it with `Access-Control-Allow-Origin: *`. |
| (c) `fetch(url, {mode: 'no-cors'})` | **No** | Returns an opaque response; body cannot be read. Useless for extraction. |
| (d) `<script>`, `<img>`, `<video>` tags | **No** (bytes) | Can load and display cross-origin media, but JS cannot read the bytes (no programmatic download from these tags without `crossorigin` + server CORS). |
| (e) DevTools copy-as-curl (manual) | N/A | User-initiated only; not a programmatic solution. |

### 2.3 Signed URLs & referer

TikTok/IG/FB video CDN URLs are typically:
- **Signed** (query-string tokens like `?a=...&b=...&Expires=...&Signature=...` or path-based signing).
- **Expiring** within minutes to hours. TikTok in particular uses very short-lived CDN links. [source: https://gist.github.com/devinschumacher/86d2843f0e1f150d79356494bc0b1a1a] [source: https://scrapecreators.com/blog/download-tiktok-videos]
- **Referer-gated**: many return 403 without `Referer: https://www.tiktok.com/` (or matching origin). Since the browser cannot set Referer, even an extension would need to use `declarativeNetRequest` to rewrite the Referer on outgoing requests, or a remote proxy must add it.

### 2.4 Browser automation

Playwright/Puppeteer **cannot run inside a web page** — they require a controlled browser over CDP/WebSocket. WebDriver BiDi is similar. None of these are options for an in-page web app. They are options only for a remote service that drives a headless browser.

---

## Part 3 — Remote Extractor Architecture

### 3.1 What does a minimal remote extractor look like?

**Shape:**
```
POST /extract
Content-Type: application/json
{"url": "https://www.tiktok.com/@user/video/123..."}

→ 200 OK
{
  "media": [
    {"type":"video","url":"https://v16-webapp.tiktok.com/...","quality":"720p","expiresAt":"2026-01-01T12:34:56Z"},
    {"type":"video","url":"https://v19-webapp.tiktok.com/...","quality":"no-watermark","expiresAt":"..."}
  ],
  "metadata": {"title": "...","author": "...","thumbnail": "..."},
  "proxyRecommended": true   // hint to client that direct fetch will fail
}
```

**Properties:**
- Stateless (no per-user session storage beyond rate-limit counters).
- No media-byte proxying by default — returns direct CDN URLs.
- Returns an `expiresAt` so the client knows when to re-extract.
- Returns a `proxyRecommended` flag (true for TikTok/IG/FB) so the client can choose to fetch via `/proxy?u=...` when direct fetch will fail.

### 3.2 When must it proxy media bytes?

The remote service must proxy (i.e., fetch the bytes server-side and stream to client) when **any** of:
1. CDN requires a `Referer` or `Origin` header that the browser **cannot** set (Referer is a forbidden header in browser fetch).
2. The URL is signed but the fetch from the browser fails CORS (the CDN does not send `Access-Control-Allow-Origin`).
3. Cookies / session tokens are required to fetch the media (browser cannot transfer platform cookies to our origin).
4. The platform rate-limits per-IP and the end-user IP is shared/flagged.
5. The CDN does TLS or HTTP/2 fingerprinting that a browser fetch from a different origin cannot satisfy (rare but possible).

For platforms where the CDN is permissive (sends `Access-Control-Allow-Origin: *` and does not gate on Referer) — the service can return direct URLs and let the client fetch directly. **For TikTok/IG/FB, this is frequently NOT the case** — proxying is usually required.

### 3.3 Can it just return direct URLs?

- **TikTok:** Sometimes — some `tiktokcdn.com` URLs work without Referer, but most are signed and short-lived; browser fetch will often 403 or be CORS-blocked. **Frequently needs proxy.**
- **Instagram:** Rarely — `scontent.cdninstagram.com` typically requires session cookies and is CORS-locked. **Almost always needs proxy.**
- **Facebook:** Rarely — video CDN URLs are signed and expiring; CORS-locked. **Almost always needs proxy.**

Honest verdict: a "direct URLs only" remote service is **not viable** as the sole strategy for IG/FB and is **fragile** for TikTok. The remote service should be designed to optionally proxy bytes.

---

## Per-Platform Feasibility Table

Legend: **possible** = browser can do it standalone; **blocked** = cannot; **needs-extension** = browser extension can do it; **needs-remote** = requires server-side fetch.

| Capability | TikTok | Instagram | Facebook |
|---|---|---|---|
| HTML fetch CORS | **blocked** | **blocked** | **blocked** |
| oEmbed endpoint accessible | **possible** (public, no auth) | **possible** (tokenless since Jun 2026) | **possible** (tokenless since Jun 2026) |
| oEmbed returns direct media URLs | **no** (embed HTML/thumbnail only) | **no** (embed HTML/thumbnail only) | **no** (embed HTML/thumbnail only) |
| Embedded JSON scrape (in-page) | **needs-extension or needs-remote** (`__UNIVERSAL_DATA_FOR_REHYDRATION__`) | **needs-remote** (GraphQL + signed headers, datacenter IP blocked) | **needs-remote** (GraphQL + signed headers) |
| Direct CDN URL fetch (browser) | **needs-remote** (signed, short-lived, often 403 without Referer) | **needs-remote** (cookies required, CORS-locked) | **needs-remote** (signed, expiring, CORS-locked) |
| Referer required to fetch media | **yes** (cannot set in browser → needs-remote) | **often yes** | **often yes** |
| Cookies required | **sometimes** (private/restricted) | **yes** (almost always) | **yes** (almost always) |
| Anti-bot (Cloudflare/HUMAN/Akamai) | **active** | **active** (very aggressive 2024-2025) | **active** |

### Honest verdict — "Can extraction be browser-only?"

- **TikTok:** **No, not as a pure web app.** Even though `__UNIVERSAL_DATA_FOR_REHYDRATION__` exposes the CDN URL, the browser cannot fetch tiktok.com HTML cross-origin (CORS), cannot set Referer on the CDN fetch (forbidden header), and CDN URLs are signed/short-lived. **Browser-only is feasible ONLY via a browser extension** with `host_permissions` for `*://*.tiktok.com/*` and `declarativeNetRequest` to spoof Referer. As a pure website, **needs-remote**.
- **Instagram:** **No.** Anti-scraping is too aggressive (datacenter IPs blocked on first request, GraphQL requires signed headers, media CDN requires cookies). Browser extension is technically possible but cookie reuse is fragile. **Needs-remote** for any reliable product.
- **Facebook:** **No.** Same pattern as Instagram: signed/expiring CDN URLs, GraphQL signing, cookies required. **Needs-remote.**

**Overall verdict:** For a browser-native (pure-web) downloader with no extension and no backend, **direct media extraction is not viable for any of the three target platforms.** Either (a) ship as a browser extension, or (b) ship with a remote extractor service. The hybrid (extension for fetch + remote for extraction logic) is the most flexible architecture.

---

## Smallest Viable Remote Service — Spec

**Endpoints:**

1. `POST /extract`
   - Request: `{"url": "<post URL>", "formatPrefs": {...optional}}`
   - Response: `{"media": [{"type","url","quality","expiresAt"}], "metadata": {...}, "proxyRecommended": bool, "source": "tiktok|instagram|facebook|..."}`
   - Stateless. Does not store URLs. Re-extract on every call.

2. `GET /proxy?u=<encoded URL>&platform=<...>` (optional, but required for TikTok/IG/FB)
   - Server fetches `u` with appropriate `Referer`, `Origin`, `User-Agent`, and (optionally) cookies.
   - Streams bytes back to client with `Access-Control-Allow-Origin: *` and `Content-Disposition: attachment` headers.
   - Used when `proxyRecommended === true` or when the client's direct fetch fails.

3. `GET /health` and `GET /platforms` (lists supported platforms).

**Implementation notes:**
- Either wrap **yt-dlp** (richest coverage, AGPL/Unlicense, Python) or build per-platform JS scrapers like Cobalt does.
- For TikTok: scrape `__UNIVERSAL_DATA_FOR_REHYDRATION__` from the page HTML using a residential-IP fetch (or use the unofficial `aweme/v1/aweme/detail` API endpoint with proper signing — complex).
- For Instagram: use the reverse-engineered GraphQL endpoint with `X-IG-App-Id` header from a residential IP; store optional session cookies for private content.
- For Facebook: parse the page HTML for `cdn_url`/`playable_url` in the embedded JSON; cookies often required.
- All three require residential-class egress IPs to avoid datacenter IP blocking; expect to integrate a residential proxy provider.
- Cache extraction results for ~30–60 seconds (shorter than CDN URL lifetime) to amortize cost.
- Return `expiresAt` to client so the client re-extracts before the URL stops working.

**License considerations:**
- If you wrap yt-dlp: yt-dlp is Unlicense (you can use it freely), but your service code can be any license.
- If you fork Cobalt's API: AGPL-3.0 — you must release source if you expose the service over a network. The Cobalt frontend's CC-BY-NC-SA license forbids commercial use of derivative frontends — **do not reuse the Cobalt frontend** in a commercial product.
- Recommended: build your own thin extractors (Unlicense or MIT) and use yt-dlp as a fallback for breadth.

---

## Uncertainties / Open Questions for Pass 2

- Cobalt internal extraction mechanism: **does it shell out to yt-dlp at all, or is it pure JS scrapers?** Inferred pure-JS but not confirmed from primary source.
- Sunfishcobalt, inv.tux.pizza: not surfaced by search; need direct GitHub/domain checks.
- Exact TikTok CDN URL lifetime (minutes vs hours) — search results say "expire quickly" / "short-lived" but no precise TTL citation. Should be measured empirically.
- Meta oEmbed tokenless change: does it cover all public content or only "publicly embeddable"? Some content may still require tokens. [UNCERTAIN]
- Whether a browser extension using `declarativeNetRequest` to spoof Referer is sufficient for TikTok CDN fetches, or whether additional headers/cookies are required.
- Concrete rate-limit numbers for Cobalt community instances.
