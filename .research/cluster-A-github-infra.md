# Cluster A — GitHub Infrastructure Research (Pass 1)

**Task ID:** 1-A
**Date:** 2026 (current)
**Scope:** GitHub Pages + GitHub Actions + GitHub Releases + git-LFS + GitHub CDN — capabilities and limits relevant to an open-source browser-native media downloader platform.
**Method:** Web search via z-ai web_search skill, ~12 queries against official docs and current community discussions.

> Unless otherwise noted, all numbers reflect GitHub.com (not GitHub Enterprise Server). Items marked **[UNCERTAIN]** need a deeper doc read or empirical test before being relied on for architecture decisions.

---

## 1. GitHub Pages

### 1.1 Hard limits

| Limit | Value | Notes |
|---|---|---|
| Source repository size (recommended) | 1 GB | Soft/recommended. [source: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits] |
| Published site size (hard) | 1 GB | "Published GitHub Pages sites may be no larger than 1 GB." [source: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits] |
| Per-month bandwidth | 100 GB (soft) | "GitHub Pages sites have a soft bandwidth limit of 100 GB per month." Paid plans have the same published soft limit; exceeding may cause the site to be disabled. [source: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits] |
| Builds per hour | 10 (soft) | "GitHub Pages sites have a soft limit of 10 builds per hour." [source: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits] |
| Build timeout | 10 minutes | "Builds … may not take longer than 10 minutes." [source: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits] |
| File count | Not officially published | Community reports issues only at very large file counts (tens of thousands). **[UNCERTAIN]** — no official cap documented. |
| Individual file size in repo | 100 MiB hard block by Git platform | Applies to all of github.com. [source: https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github] |

### 1.2 Custom domain, HTTPS, HTTP/2

- Custom domains supported via `CNAME` file at repo root. [source: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site]
- HTTPS is enforced by default on `*.github.io` and supported on custom domains (Let's Encrypt certs, auto-renewed). HTTPS enforcement toggle available. [source: https://github.com/isaacs/github/issues/156 + official docs]
- **HTTP/2 is served** for both `*.github.io` and custom domains over HTTPS. [source: https://github.com/isaacs/github/issues/1204]
- HSTS is sent by GitHub Pages. **[UNCERTAIN — needs curl test]** whether `max-age` is platform-default.

### 1.3 Jekyll processing & `.nojekyll`

- GitHub Pages runs Jekyll by default. Jekyll ignores files/directories that match its defaults (e.g., `_layouts`, files beginning with `_`) and **mangles some binary / dotfile assets**.
- Adding an empty `.nojekyll` file at the root of the published branch disables Jekyll processing — files are served verbatim. This is required for any modern bundler output (Vite, esbuild, Next.js static export) and **especially for `.wasm` / `.js` module files**. [source: https://til.simonwillison.net/github/github-pages]
- WASM served through Jekyll processing historically got the wrong MIME type (`text/plain` or octet-stream); `.nojekyll` fixes this. [source: https://community.latenode.com/t/wasm-file-serving-with-incorrect-content-type-on-github-pages/32127 ; https://github.com/github/pages-gem/issues/695]

### 1.4 MIME types

- GitHub Pages serves **more than 750 MIME types** across thousands of file extensions, derived from the `mime-db` package. [source: https://stackoverflow.com/questions/15951012/can-mime-types-of-github-pages-files-be-configured]
- Confirmed: `.wasm` → `application/wasm` (correct). [source: https://www.reddit.com/r/WebAssembly/comments/9297tg/wasm_and_github_pages]
- MIME types **cannot be customized** per-file (no `_headers` support — see §1.7).
- `.js` modules served with `Content-Type: text/javascript` (correct for ES modules). **[UNCERTAIN — needs curl test for current `Content-Type` and `charset`.]**

### 1.5 SPA fallback / redirects

- No native server-side rewrite. SPA deep-link refresh will 404.
- Workaround: a `404.html` file at root is served for any unmatched path. Putting a small JS redirect inside `404.html` that rewrites to `index.html` + hash/route is the de-facto SPA pattern on Pages. [source: https://github.com/orgs/community/discussions/27676 ; https://devactivity.com/insights/decoding-github-pages-404s-essential-fixes-for-broken-links]
- Pattern from isaacs/github issue #408 (2015) is the canonical request; GitHub has **not** added native SPA support. [source: https://github.com/isaacs/github/issues/408]

### 1.6 `_headers` / `_redirects` (Netlify-style)

- **Not supported.** Pages does not parse `_headers` or `_redirects` files. [source: https://www.sanity.io/answers/migrating-from-wordpress-should-i-stick-with-github-actions-and-pages ; https://www.luckymedia.dev/compare/cloudflare-workers-vs-github-pages]
- No way to set `Cache-Control`, `Content-Security-Policy`, `Strict-Transport-Security`, `Cross-Origin-Embedder-Policy`, `Cross-Origin-Opener-Policy`, etc. at the platform level.
- This is a **major constraint** for serving WASM that needs COOP/COEP headers for `SharedArrayBuffer` (see Risks).

### 1.7 Cache headers / CDN behavior

- Default `Cache-Control: max-age=600` (10 minutes) on static assets. [source: https://github.com/orgs/community/discussions/11884]
- The value cannot be customized (no `_headers`).
- Redeploys do **purge** edge cache as the underlying objects change SHA, but browsers and intermediate caches that respect `max-age=600` may serve stale content for up to 10 min. **[UNCERTAIN]** whether Pages emits `ETag`/`Last-Modified` strong validators for 304 negotiation — community suggests yes.
- Immutable asset pattern: serving files under content-hashed paths (e.g. `/assets/<sha>.js`) works around the short `max-age` because the URL never changes content. **Highly recommended for this project.**

### 1.8 Large files on Pages

- Pages is **not** designed to serve large binaries. Each file must be <100 MiB (platform Git limit) and the published site total must be <1 GB.
- For FFmpeg.wasm and similar: a single `ffmpeg-core.wasm` is ~30 MB (single-threaded) or ~120 MB (multi-threaded + SIMD). Multi-threaded FFmpeg.wasm **will not fit comfortably** under both the 100 MiB single-file limit (the `.wasm` alone is borderline) and the 1 GB total site budget if multiple versions are shipped. [source for size: known FFmpeg.wasm releases — confirm in cluster-D research]

---

## 2. GitHub Actions

### 2.1 Workflow / job limits

| Limit | Value | Source |
|---|---|---|
| Job execution timeout | 6 hours per job | https://docs.github.com/en/actions/reference/limits |
| Workflow run timeout | 35 days max (the run wall-clock cap) | https://docs.github.com/en/actions/reference/limits |
| Matrix jobs per workflow | 256 | https://docs.github.com/en/actions/reference/limits |
| Reusable workflow depth | up to 4 levels (incl. top-level) | https://docs.github.com/en/actions/reference/limits |
| Reusable workflows per run | 30 (referenced from limits doc) | https://docs.github.com/en/actions/reference/limits |
| Queue size per concurrency group | 100 jobs/runs (`queue: max`) | https://docs.github.com/en/actions/reference/limits |
| Concurrent jobs (Free) | 20 (non-macOS), 5 macOS | https://github.com/orgs/community/discussions/184661 ; https://earthly.dev/blog/concurrency-in-github-actions |
| Concurrent jobs (Pro) | 40 non-macOS | https://github.com/orgs/community/discussions/184661 |
| Concurrent jobs (Team) | 60 non-macOS | https://github.com/orgs/community/discussions/184661 |
| Free minutes/month (Free) | 2,000 | https://tenki.cloud/blog/github-actions-cost-optimization |
| Free minutes/month (Pro) | 3,000 | https://www.blacksmith.sh/blog/how-to-reduce-spend-in-github-actions |
| Free minutes/month (Team) | 3,000 (some sources say 50k for Enterprise) | https://tenki.cloud/blog/github-actions-cost-optimization |
| **Public repos** | **Free, unlimited** Actions minutes on GitHub-hosted runners | https://docs.github.com/billing/managing-billing-for-github-actions/about-billing-for-github-actions |
| Self-hosted runners | Free, unlimited minutes | https://docs.github.com/billing/managing-billing-for-github-actions/about-billing-for-github-actions |

> **Critical for our OSS project**: a public repo gets free, unlimited Actions minutes for CI/CD, release building, and maintenance tasks. We will not run out of build minutes.

### 2.2 Cache & artifacts

- **Actions cache**: 10 GB per repository (free allowance). Eviction is LRU when exceeded. [source: https://docs.github.com/billing/managing-billing-for-github-actions/about-billing-for-github-actions ; https://medium.com/@akhilesh-mishra/your-github-actions-runners-are-slow-and-you-are-paying-too-much-for-them-5406577314fe]
- **Artifact retention**: default 90 days, configurable 1–400 days (free) / up to 400 days (paid). [source: https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization]
- Artifacts stored in `actions` storage also count against a 500 GB-per-org (Free) / 2 TB (Pro) packaged storage allowance. **[UNCERTAIN on exact Free/Pro split — needs billing doc confirmation]**

### 2.3 Scheduled (cron) workflows

- **Minimum interval**: GitHub documents that scheduled workflows "will wait at least 5 minutes between workflow runs" (i.e. cron `*/5 * * * *` is the effective floor, even though syntax accepts `*/1`). [source: https://stackoverflow.com/questions/79534419/reliability-issues-with-github-actions-with-cron-based-schedule]
- **Reliability is poor and explicitly best-effort.** GitHub may delay runs by 5–30 minutes during high load, and may **silently skip** runs entirely. Documented community reports of multi-hour delays and entire days being dropped. [source: https://github.com/orgs/community/discussions/156282 ; https://github.com/orgs/community/discussions/201738 ; https://dev.to/krissv/monitoring-github-actions-scheduled-workflows-a-practical-guide-31h7]
- Runs scheduled at top-of-hour (`:00`) are worst-affected.
- **Implication**: scheduled Actions workflows are fine for low-frequency maintenance (e.g. nightly dependency rebuild, weekly checksum refresh), but must NOT be relied on for anything user-facing or time-critical.

### 2.4 Secrets, OIDC, environments, reusable workflows

- **Secrets**: repository-, environment-, and organization-level. Masked in logs. Not passed to reusable workflows by default (must `secrets: inherit` or pass explicitly). [source: https://docs.github.com/actions/security-guides/using-secrets-in-github-actions]
- **OIDC**: supported. Workloads can mint short-lived OIDC tokens to authenticate to AWS/GCP/Azure/HashiCorp Vault without long-lived secrets. [source: https://docs.github.com/en/actions/concepts/security/openid-connect]
- **Dependabot**: now supports OIDC for private registries (Feb 2026). Dependabot-triggered workflows **cannot access non-Dependabot secrets** by default. [source: https://github.blog/changelog/2026-02-03-dependabot-now-supports-oidc-authentication ; https://docs.github.com/actions/security-guides/using-secrets-in-github-actions]
- **Environments**: support required reviewers (up to 6), deployment branches, wait timers, and environment-scoped secrets. Required reviewers enforce a manual approval gate before any job using that environment runs. [source: https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment]
- **Reusable workflows**: can be called via `workflow_call`. Up to 4 levels of nesting. Supports matrix passthrough. [source: https://docs.github.com/en/actions/reference/limits]
- **Matrix builds**: max 256 jobs per workflow run; can be combined with reusable workflows (with caveats — nested matrices can exceed 256 but UI breaks around ~600). [source: https://github.com/orgs/community/discussions/38704]

### 2.5 Security (token permissions, `pull_request_target`, injection)

- `GITHUB_TOKEN` is auto-issued per workflow run. **Best practice**: set `permissions:` block at workflow/job level to least-privilege. Default for new repos is read-only (changed in 2023); old repos may still default to write-all. [source: https://docs.github.com/en/actions/reference/security/secure-use]
- **`pull_request_target` is dangerous**: runs with the base branch's token and secrets but checks out PR head code — classic "pwn-request" vector. Avoid; if required, never `run:` PR-supplied code or build outputs in such workflows. [source: https://www.stepsecurity.io/blog/github-actions-pwn-request-vulnerability ; https://blog.gitguardian.com/github-actions-security-cheat-sheet ; https://www.wiz.io/blog/github-actions-security-guide]
- **Expression injection** via `github.event.issue.body`, `github.event.pull_request.title`, etc. — never interpolate untrusted input into `run:` blocks. Use environment variables or `actions/github-script`. [source: https://blog.gitguardian.com/github-actions-security-cheat-sheet]
- **Dependabot** is GitHub-native and recommended for keeping Actions, npm deps, and Docker base images current. [source: https://docs.github.com/en/code-security/dependabot]
- **GitHub Advanced Security (GHAS) + CodeQL** is paid for private repos but **free for public repos** — useful for our OSS project. [source: https://docs.github.com/en/code-security/code-scanning]

### 2.6 "Actions must not be runtime"

This is enforced by **architecture, not platform**: GitHub Actions runners exist only during a workflow run; they are not a hosting tier. Pages serves the static output. Releases serve binaries on demand from `objects.githubusercontent.com` / `release-assets.githubusercontent.com`. There is no "always-on" Actions endpoint users can hit. As long as we never put user-downloadable logic in a `workflow_dispatch`-or-scheduled path that proxies downloads, we are safe.

---

## 3. GitHub Releases

### 3.1 Asset limits

- Per-asset size: **2 GiB hard limit**. Each file in a release must be under 2 GiB. [source: https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases ; https://github.com/orgs/community/discussions/165616]
- Total release size: **no limit**. Total number of assets: no published limit (community has projects with thousands of releases). [source: https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases]
- Bandwidth: **no published bandwidth cap** on release asset downloads. [source: https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases ; https://www.reddit.com/r/github/comments/12pj32j/is_there_really_no_limit_on_github_release]
  - Caveat: GitHub's Acceptable Use Policy still applies and they reserve the right to throttle "abuse" patterns (e.g. terabytes/day from a single project). For normal OSS distribution this is not an issue.

### 3.2 Checksums / digests

- As of **June 3, 2025**, GitHub **automatically computes and displays SHA-256 digests** for every uploaded release asset. Visible in the release UI and retrievable via API (`GET /repos/{owner}/{repo}/releases/{id}/assets` → `digest` field). [source: https://github.blog/changelog/2025-06-03-releases-now-expose-digests-for-release-assets]
- Prior to this, projects had to ship their own `SHA256SUMS` file as a separate release asset — still a useful pattern for verifying in-band.
- API does not expose MD5/SHA-1.

### 3.3 Programmatic access

- REST API: `GET /repos/{owner}/{repo}/releases/latest`, `GET /repos/{owner}/{repo}/releases/{release_id}/assets`. [source: https://docs.github.com/en/rest/releases/releases]
- **Unauthenticated API rate limit**: 60 requests/hour per IP. Authenticated (token or GitHub App): 5,000/hour. [source: https://docs.github.com/en/rest/overview/rate-limits ; https://github.com/krkn-chaos/krknctl/issues/120]
- Direct asset download URLs of the form `https://github.com/{owner}/{repo}/releases/download/{tag}/{file}` 302-redirect to `https://objects.githubusercontent.com/…` (or the newer `https://release-assets.githubusercontent.com/…` domain rolled out 2025). These redirect URLs are not API-rate-limited and work for curl/wget/fetch. [source: https://github.com/orgs/community/discussions/141907 ; https://www.stepsecurity.io/blog/harden-runner-detects-new-traffic-to-release-assets-githubusercontent-com-across-multiple-customers ; https://support.hashicorp.com/hc/en-us/articles/42203329876883]
- **CORS**: release-asset downloads **support cross-origin `fetch`** in browsers for the redirected `objects.githubusercontent.com` URL — historically used by auto-update frameworks. **[UNCERTAIN — needs empirical CORS test against `release-assets.githubusercontent.com` since the 2025 domain change.]**

### 3.4 Releases vs LFS vs Pages for distributing large binaries

| Channel | Per-file cap | Total cap | Bandwidth cap | Best for |
|---|---|---|---|---|
| GitHub Pages | 100 MiB | 1 GB | 100 GB/mo (soft) | App shell, small JS/CSS, small WASM glue |
| GitHub Releases | 2 GiB | none | none (AUP only) | FFmpeg.wasm, yt-dlp binary, large model files |
| git-LFS | 2 GiB per file | 10 GB free / paid packs | 10 GB/mo free / paid packs | Source-controlled large assets — **avoid** for downloads |

**Recommendation**: ship the web app (HTML/JS/CSS + small WASM glue) on Pages; ship every large binary (FFmpeg.wasm core, yt-dlp executable, optional model files) as GitHub Release assets, and have the client fetch them at runtime via direct `releases/download/...` URLs. This keeps Pages under its 1 GB / 100 GB bandwidth envelope and lets GitHub's CDN absorb the heavy traffic.

---

## 4. git-LFS

### 4.1 Free/paid quotas (as of mid-2023 changes, still current 2026)

- **Free and Pro accounts**: 10 GB storage + 10 GB bandwidth/month included. [source: https://github.com/orgs/community/discussions/61362 — "GitHub is raising the free, included amount of Git LFS resources to 10 GB storage and bandwidth for Free and Pro accounts"]
- **Older docs/blog posts** still cite 1 GB / 1 GB — this is outdated; the 10/10 increase happened July 2023. [source: https://github.com/orgs/community/discussions/61362]
- Additional data packs: $5/month per 50 GB (storage + bandwidth). [source: https://wlcx.cc/blog/github-lfs-rant ; https://jamesoclaire.com/2024/12/06/github-large-file-storage-git-lfs-is-basically-paid-only]
- Bandwidth does **not** roll over month-to-month. [source: https://ryp.github.io/2023/git-lfs-and-github-free-tier]

### 4.2 Does LFS count against Pages?

- **No.** Pages does not serve LFS objects; if you put LFS-tracked files in a Pages branch they will be served as LFS pointer files (tiny text stubs), not the actual content. Pages bandwidth is independent of LFS bandwidth. [source: inferred from platform design + https://github.com/orgs/community/discussions/50594]
- LFS bandwidth is consumed on `git clone`/`pull`/`fetch` of LFS objects, not on Pages traffic.

### 4.3 LFS on forks

- **Problematic.** GitHub historically blocks pushing LFS objects to a fork unless the parent repo already has those objects (or you have push access to the parent). This breaks the normal "fork → PR" OSS contribution model. [source: https://github.com/git-lfs/git-lfs/issues/1449 ; https://docs.ropensci.org/piggyback/articles/alternatives.html]
- Implication: **do not use LFS for any asset contributors might modify.** For our project, large binaries should live in Releases, not LFS.

### 4.4 LFS asset size limits

- Per-file: 2 GiB (same as Git platform limit applies via LFS pointer mechanism). [source: https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github]

---

## 5. GitHub CDN (Fastly-backed)

### 5.1 Edge provider

- GitHub.com (including Pages and `*.githubusercontent.com` asset domains) is fronted by **Fastly** as its CDN. [source: https://www.fastly.com/customers/github — "GitHub's CDN needed to be replaced quickly. See how Fastly was able to efficiently configure GitHub's server…"]
- Global edge presence via Fastly's POP network. **[UNCERTAIN]** whether Pages-specific responses come from the same Fastly config as the rest of github.com or a dedicated config.

### 5.2 Cache TTL behavior for Pages

- Default `Cache-Control: max-age=600` on static assets (10 min). HTML may be cached shorter. [source: https://github.com/orgs/community/discussions/11884]
- Not customizable (no `_headers`).
- Browsers and intermediate caches honor the 10-min `max-age`; this means **post-deploy staleness window of up to 10 minutes for non-URL-changed assets**.

### 5.3 Immutable assets via commit-SHA / content-hash URLs

- Pattern that defeats the short TTL: serve hashed assets under `/assets/<sha>.{js,wasm,css}`. The URL changes only when content changes, so the 10-min `max-age` is effectively irrelevant — clients fetch once per content version and cache forever.
- For per-commit URLs like `https://<user>.github.io/<repo>/@<sha>/app.js` — these work but are **not officially documented** as a CDN-busting pattern. **[UNCERTAIN]** whether such URLs get stronger caching headers automatically; community evidence suggests they get the same `max-age=600`. [source: inferred — needs empirical test]

### 5.4 Do redeploys purge cache?

- A redeploy swaps the underlying objects; the Fastly edge re-fetches on cache expiry (≤10 min for `max-age=600` assets). For HTML (typically shorter or `no-cache`), propagation is near-immediate. [source: https://github.com/orgs/community/discussions/49753]
- There is **no manual purge button** in the Pages UI; you must redeploy or wait for TTL expiry.

### 5.5 Release asset CDN

- Release assets are served from `objects.githubusercontent.com` (and the newer `release-assets.githubusercontent.com` rolled out 2025). These are also Fastly-fronted object-storage endpoints, distinct from the Pages edge config. [source: https://github.com/orgs/community/discussions/141907 ; https://github.com/orgs/community/discussions/58455]
- Cache headers on release assets: **[UNCERTAIN — needs curl test]**. Community reports HEAD requests behave inconsistently; GET requests generally work.

---

## Key Limits Summary

| Resource | Free plan | Pro plan | Notes / Source |
|---|---|---|---|
| Pages published site size | 1 GB | 1 GB | hard limit |
| Pages source repo size (recommended) | 1 GB | 1 GB | soft |
| Pages bandwidth / month | 100 GB (soft) | 100 GB (soft) | excess may disable site |
| Pages builds / hour | 10 (soft) | 10 (soft) | |
| Pages build timeout | 10 min | 10 min | |
| Pages single-file size | 100 MiB | 100 MiB | platform Git limit |
| Actions job timeout | 6 h | 6 h | hard |
| Actions matrix jobs / run | 256 | 256 | |
| Actions concurrent jobs | 20 | 40 | non-macOS |
| Actions minutes / month | 2,000 | 3,000 | **unlimited for public repos** |
| Actions cache / repo | 10 GB | 10 GB | LRU eviction |
| Actions artifact retention | 90 d default (1–400 d configurable) | same | |
| Scheduled cron min interval | 5 min | 5 min | best-effort, unreliable |
| Releases per-asset size | 2 GiB | 2 GiB | hard |
| Releases total size | unlimited | unlimited | |
| Releases bandwidth | unlimited (AUP applies) | unlimited (AUP applies) | |
| Releases SHA-256 digest | auto (since Jun 2025) | auto | exposed via API |
| REST API unauthenticated | 60 req/h per IP | 60 req/h per IP | 5,000/h authenticated |
| git-LFS storage | 10 GB | 10 GB | data packs $5/50 GB |
| git-LFS bandwidth / month | 10 GB | 10 GB | no rollover |
| Pages custom HTTP headers | not supported | not supported | no `_headers`/`_redirects` |
| Pages default cache TTL | max-age=600 | max-age=600 | not customizable |
| Pages HTTP/2 | yes | yes | Fastly edge |
| Pages HTTPS | enforced | enforced | custom-domain certs auto |
| Pages SPA fallback | via `404.html` trick only | same | no native rewrite |
| GitHub Advanced Security / CodeQL | free for public repos | free for public repos | |
| GitHub CDN provider | Fastly | Fastly | Pages + release-assets |

---

## Risks for This Project

1. **Pages 100 GB/month bandwidth ceiling.** A media downloader app fetches FFmpeg.wasm (~30 MB single-thread / ~120 MB multi-thread) per new visitor. At 30 MB per fresh load, 100 GB/mo ≈ **~3,300 unique first-load visits/month** before hitting the soft cap. Multi-threaded FFmpeg.wasm would cut this to ~830 visits/month. **Mitigation**: ship FFmpeg.wasm + yt-dlp binaries via **GitHub Releases** (no bandwidth cap) and have the app fetch them from `releases/download/...` URLs — Pages only serves the small app shell.

2. **Pages 1 GB published-site limit + 100 MiB per-file Git cap.** Multi-threaded FFmpeg.wasm (~120 MB) **exceeds the per-file Git limit** and cannot live in the repo at all. Even single-threaded FFmpeg.wasm (~30 MB) plus yt-dlp binary plus model files would rapidly consume the 1 GB Pages budget across versions. Reinforces risk #1's mitigation.

3. **No custom HTTP headers on Pages → no COOP/COEP → no `SharedArrayBuffer` for multi-threaded WASM.** Multi-threaded FFmpeg.wasm requires `Cross-Origin-Opener-Policy: same-origin` AND `Cross-Origin-Embedder-Policy: require-corp` on the page. Pages **cannot set these headers**. Options:
   - Accept single-threaded FFmpeg.wasm (slower, but works on Pages).
   - Host the actual worker/app shell on a different origin that supports COOP/COEP (Cloudflare Pages, Netlify, self-hosted) and use Pages only for the marketing/landing page.
   - Use a service worker + `crossOriginIsolated` workaround — **still requires the response headers**, so not viable on Pages.
   This is a **first-class architectural risk** for the whole project.

4. **Pages default `Cache-Control: max-age=600` (10 min).** Without content-hashed asset URLs, deploys have a 10-minute stale-content window. **Mitigation**: every JS/CSS/WASM asset must be emitted under `/<hash>.ext` paths (Vite/esbuild default). HTML entry must use cache-busting query strings or short cache.

5. **No `_redirects`/`_headers` on Pages.** SPA deep links break on refresh. **Mitigation**: ship a `404.html` that JS-redirects to `index.html` preserving the path. No clean solution for canonical redirects.

6. **Scheduled Actions are unreliable.** Any "auto-update the yt-dlp binary every night" or "refresh release checksums hourly" workflow will be delayed or skipped. **Mitigation**: design maintenance workflows to be **idempotent and self-healing** — also trigger them on `push`, on `workflow_dispatch`, and via an external cron (e.g. a free UptimeRobot/cron-job.org hitting a `workflow_dispatch` webhook). Never rely on the schedule alone for user-visible freshness.

7. **`pull_request_target` / expression injection.** If we accept community PRs that modify build scripts, workflows, or asset lists, a malicious PR could exfiltrate `GITHUB_TOKEN` or secrets. **Mitigation**: set `permissions: contents: read` by default; never use `pull_request_target` to run PR code; require maintainer approval before running any CI on fork PRs; use `tj-actions/changed-files` or similar to gate.

8. **git-LFS on forks breaks OSS contributions.** Don't store any contributor-modifiable large asset in LFS. Keep all large binaries in Releases (read-only for contributors; new versions uploaded by maintainers via Actions).

9. **Unauthenticated GitHub API = 60 req/hour/IP for release metadata.** If the app calls `GET /releases/latest` at runtime to find the latest FFmpeg.wasm version, a shared NAT (school, office) could exhaust 60/hour quickly. **Mitigation**: hardcode latest-known version in the app shell (updated at build time), or use the direct `releases/download/<tag>/<file>` URL pattern (not API-rate-limited) and update the tag via Actions.

10. **Release-asset CORS after the 2025 `release-assets.githubusercontent.com` rollout.** Browser `fetch()` of release assets worked historically, but the new domain may have different CORS headers. **Needs empirical test** before committing to runtime-fetching release assets from the browser.

11. **Pages 10 builds/hour soft limit.** A busy CI pipeline pushing many commits could exhaust Pages build quota if each commit triggers a Pages deploy. **Mitigation**: only deploy to Pages on `main` branch + tags, not on every PR.

12. **No manual CDN purge on Pages.** If a bad deploy ships (e.g. broken JS), users will see it for up to 10 minutes even after a fix push. **Mitigation**: stage releases; use feature flags; consider a `?v=<sha>` cache-bust on the HTML entry.

13. **2 GiB per release asset.** Sufficient for FFmpeg.wasm and yt-dlp for all plausible platforms. Only an issue if we ever ship bundled models > 2 GiB (unlikely). Splitting is the documented workaround.

---

## Sources Index (canonical URLs)

- GitHub Pages limits: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits
- About large files on GitHub: https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github
- GitHub Pages: The Missing Manual (Simon Willison): https://til.simonwillison.net/github/github-pages
- WASM MIME on Pages: https://community.latenode.com/t/wasm-file-serving-with-incorrect-content-type-on-github-pages/32127 and https://github.com/github/pages-gem/issues/695
- MIME types configurable? https://stackoverflow.com/questions/15951012/can-mime-types-of-github-pages-files-be-configured
- HTTP/2 on Pages: https://github.com/isaacs/github/issues/1204
- SPA 404 workaround: https://github.com/orgs/community/discussions/27676 and https://github.com/isaacs/github/issues/408
- No `_headers`/`_redirects` on Pages: https://www.sanity.io/answers/migrating-from-wordpress-should-i-stick-with-github-actions-and-pages and https://www.luckymedia.dev/compare/cloudflare-workers-vs-github-pages
- Pages default `Cache-Control: max-age=600`: https://github.com/orgs/community/discussions/11884
- Actions limits: https://docs.github.com/en/actions/reference/limits
- Actions billing (cache 10 GB, free for public repos): https://docs.github.com/billing/managing-billing-for-github-actions/about-billing-for-github-actions
- Actions artifact retention (90 days default): https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization
- Concurrent jobs per plan (Free 20 / Pro 40 / Team 60): https://github.com/orgs/community/discussions/184661
- Free/Pro minutes per month (2,000 / 3,000): https://tenki.cloud/blog/github-actions-cost-optimization and https://www.blacksmith.sh/blog/how-to-reduce-spend-in-github-actions
- Scheduled workflow unreliability: https://github.com/orgs/community/discussions/156282 , https://github.com/orgs/community/discussions/201738 , https://dev.to/krissv/monitoring-github-actions-scheduled-workflows-a-practical-guide-31h7
- 5-minute minimum scheduled interval: https://stackoverflow.com/questions/79534419/reliability-issues-with-github-actions-with-cron-based-schedule
- OIDC: https://docs.github.com/en/actions/concepts/security/openid-connect
- Secrets & reusable workflows: https://docs.github.com/actions/security-guides/using-secrets-in-github-actions
- Dependabot OIDC (Feb 2026): https://github.blog/changelog/2026-02-03-dependabot-now-supports-oidc-authentication
- Environments / required reviewers: https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment
- Security: `pull_request_target` risk: https://www.stepsecurity.io/blog/github-actions-pwn-request-vulnerability , https://blog.gitguardian.com/github-actions-security-cheat-sheet , https://www.wiz.io/blog/github-actions-security-guide
- Releases about (2 GiB/asset, no total/bandwidth limit): https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases
- Releases auto SHA-256 digests (Jun 2025): https://github.blog/changelog/2025-06-03-releases-now-expose-digests-for-release-assets
- Release asset 2 GiB enforcement: https://github.com/orgs/community/discussions/165616 and https://github.com/orgs/community/discussions/196657
- Unauthenticated API 60/h: https://github.com/krkn-chaos/krknctl/issues/120 and https://github.com/rust-lang/rust-analyzer/issues/3094
- Release asset CDN domains: https://github.com/orgs/community/discussions/141907 , https://github.com/orgs/community/discussions/58455 , https://www.stepsecurity.io/blog/harden-runner-detects-new-traffic-to-release-assets-githubusercontent-com-across-multiple-customers , https://support.hashicorp.com/hc/en-us/articles/42203329876883
- git-LFS 10 GB free/Pro (Jul 2023): https://github.com/orgs/community/discussions/61362
- LFS data packs pricing: https://wlcx.cc/blog/github-lfs-rant , https://jamesoclaire.com/2024/12/06/github-large-file-storage-git-lfs-is-basically-paid-only
- LFS bandwidth no rollover: https://ryp.github.io/2023/git-lfs-and-github-free-tier
- LFS broken on forks: https://github.com/git-lfs/git-lfs/issues/1449 , https://docs.ropensci.org/piggyback/articles/alternatives.html
- Fastly as GitHub's CDN: https://www.fastly.com/customers/github

---

*End of Cluster A research.*
