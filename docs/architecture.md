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
  │                              │        │  cache/<inputs>/<analysis>/... │
  │                              │        │    (saved work, 30 days)       │
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

## Saved work (runner `cache/`)

The engine's first half (up to "aggregate per bag") does not depend on
lang/paper/labels: `--analysis-dir` saves it and `--from-analysis` renders from
it. The runner keys saved work by `desembrick --cache-key` =
`<inputs sha256>/<analysis code>/<engine code>`, so the same manual is not read
again, whoever sends it:

```
 runner: key = --cache-key <pdfs>
     │
     ├─ cache/<inputs>/<analysis>/<engine>/<lang>-<paper>.tar ? ──yes──▶ copy it (nothing runs)
     │
     ├─ cache/<inputs>/<analysis>/analysis.tar ? ──yes──▶ --from-analysis (PDF not read)
     │                                                    └─ refused/broken ──┐
     └─ no, or force_reparse ────────────────────────────────────────────────┴─▶ full run
                                                                   + save analysis.tar
     then save <lang>-<paper>.tar unless it was copied (best effort)
```

A fix in the analysis code changes `<analysis>`: every manual is read again.
A fix only in the PDF or labels changes `<engine>`: rendered again from the
saved analysis. `force_reparse` is a maintainer-only input of `process.yml`.

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
| R2 `cache/...` (saved work), `--cache-key` format | engine | runner only |
| `process.yml` dispatch inputs | engine | api |
| Brand tokens / SVGs (`docs/brand/`) | workspace | web (copied), engine (PDF layout) |

Dependency order for changes: **engine → api → web**.
