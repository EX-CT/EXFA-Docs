# 25 — Deployment options for the EXFA web app

Status: research, recommendation at the bottom. Written 2026-10-05 after the
EXFA-App migration landed (site live at https://ex-ct.github.io/EXFA-App/).

## What is being deployed

A fully static SPA: `apps/web` → `vite build` → `dist/` containing

- `index.html` + hashed JS/CSS bundles (~450 kB gz),
- `data/dataset.json.gz` (~1.3 MB, SDE dataset baked into the deploy),
- `prices/latest.json.gz` (~250 kB, latest EXFA-Data price snapshot; refreshed
  hourly by a rebuild),
- `engines/f/exfa_wasm.wasm` (~5.4 MB) and `exfa_formats_wasm.wasm` (~2.9 MB)
  pulled from the EXFA-Engine release,
- `404.html` copy of `index.html` (SPA fallback; the EVE SSO callback lands on
  `/esi/callback`).

Total ~10 MB, largest single file ~5.4 MB. No server-side code, no secrets in
the payload.

## Requirements the pipeline already satisfies

- The GitHub Actions `pages.yml` build is the gate: it downloads the pinned
  dataset + engine release + price snapshot, builds, then runs the headless
  Chrome smoke test, the wasm-worker e2e suite and the full EXFA-Bench browser
  suites against `baselines/web.json`. Any additional host must therefore be a
  *publish target of that same build* — building again on the host's own
  pipeline would skip the bench gate (headless Chrome is not available on
  Cloudflare's or Vercel's builders).
- SPA fallback (`404.html` or the platform equivalent) for `/esi/callback`.
- Correct headers: `*.wasm` as `application/wasm`, `*.json.gz` may be served
  pre-compressed (the app uses `Content-Encoding`-aware fetch), immutable
  caching for hashed assets, short/no cache for `manifest.json`,
  `build-info.json`, `prices/latest.json.gz`.

## Options

### A. GitHub Pages (current)

- Zero cost, zero extra infra, deploy is `actions/upload-pages-artifact` +
  `deploy-pages` — already working.
- Custom domain possible (`exfa.example.org` + CNAME) but no other knobs:
  fixed headers, no edge logic, `Cache-Control` is GitHub's default
  (10 min for HTML, immutable for hashed assets is automatic).
- Hourly price refresh means an hourly Pages redeploy — works, but Pages
  rebuilds are rate-limited (~10 builds/hour, plenty) and propagation is a CDN
  flush of a few minutes.

### B. Cloudflare Pages (recommended complement)

`wrangler pages deploy dist` from the *same* `pages.yml` job (or a follow-up
`deploy-cf` job that downloads the pages artifact, so the gated bits are
bit-identical). Not the git-connected builder — Direct Upload only.

- 25 MB/file and 20 k-file limits: fine (largest file 5.4 MB).
- SPA fallback: `_redirects` file with `/* /index.html 200`, or keep shipping
  `404.html` (Pages honours it). `engines/`, `data/`, `prices/` paths have real
  files so they are unaffected.
- Headers via `public/_headers` — set `Cache-Control: immutable` on
  `/assets/*`, no-cache on `/prices/*`, `/data/manifest.json`, `build-info.json`.
- Custom domain on any zone you control (`exfa.isilna.net` etc.) with free TLS,
  global CDN, and instant cache purge per deploy — better than GitHub Pages
  for the hourly-updating `prices/latest.json.gz` (GH Pages caches it for
  ~10 min regardless).
- Needs two secrets in the EXFA-App repo: `CF_API_TOKEN` (Pages:Edit scoped)
  and `CF_ACCOUNT_ID`. Optional `CF_PROJECT` name (default `exfa`).
- Free tier: unlimited static requests, 500 builds/month is irrelevant (we
  never build on CF).

### C. Own server (VPS + nginx)

- Works — `rsync dist/` behind nginx, certbot for TLS, fine-grained
  cache rules.
- Downsides vs B: no edge CDN, you own TLS/uptime/DDoS, deploy needs SSH keys
  in CI, hourly redeploy is a `rsync --delete` cron — strictly worse here
  unless the site must be served from inside China without ICP issues (even
  then, CF's China network needs Enterprise). Keep as fallback.

### D. Cloudflare Workers + R2 / Workers Static Assets

- Only worth it once the app needs server logic (ESI token proxy, multi-user
  fit store, Turso sync). Workers Static Assets can already serve `dist/` with
  the same Wrangler deploy, so migrating from B to D later is a config change,
  not a rebuild.

## Recommendation

Keep GitHub Pages as the always-green primary (it is free and already gated),
add Cloudflare Pages as a second publish target of the same gated build:

```
pages.yml: build job (unchanged)
  ├─ deploy job  → GitHub Pages (unchanged)
  └─ deploy-cf   → wrangler pages deploy dist  (needs secrets)
```

The `deploy-cf` job is ~15 lines of workflow and can sit disabled
(`if: vars.CF_ENABLED == 'true'`) until `CF_API_TOKEN` + `CF_ACCOUNT_ID` are
set. Once a custom domain is pointed at CF, that becomes the public URL and
GitHub Pages remains the no-dependency fallback.

When a server-side need appears (ESI proxying with a confidential client,
multi-device sync), revisit as Workers + R2 — do not add Turso for the current
read-only cache shape (see docs/22: prices are an embedded + refreshed snapshot,
not a database).
