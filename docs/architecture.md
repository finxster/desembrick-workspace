# Desembrick — Architecture

Overview of how the three repositories fit together. See `CLAUDE.md` for
change routing and the JSON contract.

## Online flow (production, free tiers)

```
  ┌──────────────┐
  │   Browser    │  user uploads an official instruction manual (PDF)
  └──────┬───────┘
         │ HTTPS
         ▼
  ┌──────────────────────────────┐
  │ web  (repos/desembrick-web)  │  Vue 3 + Vite SPA on Cloudflare Pages
  │ upload → wait → download     │  Turnstile widget (managed mode)
  └──────┬───────────────────────┘
         │ /api/*
         ▼
  ┌──────────────────────────────┐        ┌────────────────────────────────┐
  │ api  (repos/desembrick-api)  │        │ R2 bucket: desembrick-jobs     │
  │ Cloudflare Worker, Hono      │ ─────▶ │  jobs/<id>/status.json         │
  │                              │        │  jobs/<id>/input/<n>  (PDFs)   │
  │ POST /api/jobs   (Turnstile, │        │  jobs/<id>/out/...  (results)  │
  │                   rate limit)│        │    (1-day lifecycle)           │
  │ PUT  /api/jobs/:id/input/:n  │        │  audit/<id>/api.json           │
  │ POST /api/jobs/:id/start     │        │  audit/<id>/runner.json        │
  │ GET  /api/jobs/:id           │        │    (90 days, hashed IPs)       │
  │ GET  /api/jobs/:id/files/*   │        └───────────────▲────────────────┘
  └──────┬───────────────────────┘                        │
         │ workflow_dispatch (GitHub API)                 │ reads input,
         ▼                                                │ writes status + out/
  ┌──────────────────────────────────────────────┐        │
  │ engine (repos/desembrick-engine)             │        │
  │ GitHub Actions: .github/workflows/process.yml│ ───────┘
  │   runner/process.sh  ──wraps──▶  engine CLI  │
  └──────────────────────────────────────────────┘
```

Job state (`jobs/<id>/status.json`, no database):

```
uploading ──▶ queued ──▶ running ──▶ done
                                 └─▶ error
```

## Engine pipeline (standalone CLI)

```
 manual.pdf
     │
     ▼
 ┌────────────────┐   ┌──────────────────────┐   ┌───────────────────┐
 │ read PDF pages │──▶│ detect bag pages     │──▶│ parse step parts  │
 └────────────────┘   │ (bag number → pages) │   │ boxes + OCR qty   │
                      └──────────────────────┘   └─────────┬─────────┘
                                                           ▼
                                               ┌───────────────────────┐
                                               │ aggregate per bag     │
                                               │ (sum quantities)      │
                                               └─────────┬─────────────┘
                                                         ▼
                              --out/  ├─ reverse manual PDF (brand layout)
                                      ├─ result JSON  { set, bags[...] }
                                      ├─ part images (*.png)
                                      └─ --debug artifacts (optional)
```

The engine has no server, database or network dependency: the runner only
wraps the CLI and copies `--out` to `jobs/<id>/out/` as is.

## Local development (`dbrick start`)

```
 web  :5173  ──▶  api :8787 (wrangler, local R2)  ──▶  engine :8790
 npm run dev      npm run dev                          runner/local_server.py
                  .dev.vars: LOCAL_RUNNER_URL          (stand-in for GitHub Actions)
```

## Cross-repo contracts

| Contract | Producer | Consumers |
|---|---|---|
| Output JSON (`set`, `bags[].number/pages/parts`) | engine | api (serves), web (renders) |
| R2 layout `jobs/<id>/...` + `status.json` states | api + runner | api, web |
| `process.yml` dispatch inputs | engine | api |
| Brand tokens / SVGs (`docs/brand/`) | workspace | web (copied), engine (PDF layout) |

Dependency order for changes: **engine → api → web**.
