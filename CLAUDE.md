# Desembrick Workspace — Claude Guidance

This repository coordinates the **Desembrick** repositories. It is not an
application repo: it holds `bin/desembrick` (alias `dbrick`), `repos.conf`,
plans and docs. Application code lives in the checkouts under `repos/`.

## What Desembrick is

Given an official Lego instruction manual (PDF), produce a "reverse manual":
for each numbered bag, the images and summed quantities of the parts used in
that bag's steps — so a mixed set can be sorted back into its original bags.

## Repositories

| Key | Folder | Role | Status |
|---|---|---|---|
| `engine` | `repos/desembrick-engine` | Python 3.12 CLI: detects bag pages and per-step parts boxes, aggregates per bag, writes PDF + JSON. Also holds the GitHub Actions runner (`process.yml`) that processes web jobs | active |
| `api` | `repos/desembrick-api` | Cloudflare Worker (TypeScript, Hono): job HTTP API, uploads to R2, dispatches the engine workflow | active |
| `web` | `repos/desembrick-web` | Vue 3 + Vite SPA on Cloudflare Pages: upload, wait, download | active |

## Context Dependencies

None are currently required. Add them here when enterprise or domain contracts
exist.

The one contract already known is the engine's output JSON, which
`api` serves and `web` renders:

```json
{ "set": "...", "bags": [ { "number": 1, "pages": [1, 2], "parts": [ { "image": "path.png", "quantity": 4 } ] } ] }
```

The engine must keep working fully standalone: command line only, inputs via
arguments or JSON, outputs under `--out`, no database or external service.

## Online flow (all free tiers)

Full ASCII diagrams: `docs/architecture.md` (keep it in sync with changes
to the flow, R2 layout or endpoints).

```
web (Cloudflare Pages) → api (Cloudflare Worker) → R2 bucket desembrick-jobs (1-day lifecycle)
                                  └─ workflow_dispatch → engine `process.yml` on GitHub Actions
```

Jobs are anonymous. State lives in `jobs/<id>/status.json` in R2 (no database):
`uploading → queued → running → done | error`. Engine outputs are copied as is
under `jobs/<id>/out/`. The engine never needs a server: the runner only wraps
the CLI. Changing the R2 layout or `status.json` touches all three repos.
Job creation needs a Turnstile token (managed mode) and is rate limited per IP.
Audit records live in `audit/<id>/api.json` (api) and `audit/<id>/runner.json`
(runner), kept 90 days; IPs only as salted hashes.

## Brand

The visual identity lives in `docs/brand/`: `README.md` (rules summary, pt-BR),
`tokens.css` (colors light/dark, typography, radii, bag color cycle `.db-bag`),
`svg/` (logo, symbol, wordmark, app icon, favicon) and the full guide PDF.
Treat it as the source of truth:

- `web` copies `tokens.css` and the SVGs it needs (favicon, logo) into its own
  tree; do not reference the workspace path at runtime.
- `engine` follows the "Página do manual reverso" rules for its output PDF
  (bag number block in the bag color, six-column parts grid).
- UI copy follows the voice section: one language at a time (pt-BR or en-US,
  picked by the user or the browser). The manufacturer's name only describes
  compatibility ("LEGO® instruction manuals"), with ®, in a few places (hero
  lead, FAQ, meta description) plus the footer trademark notice; never in the
  product name, logo, headings or as if the site were official.

## Dependency Order

1. `engine`
2. `api`
3. `web`

## Change Routing

| Change type | Primary repo |
|---|---|
| PDF reading, bag-page detection, step parts-box parsing, OCR of quantities | `engine` |
| Aggregation per bag, output PDF layout, output JSON shape | `engine` |
| Engine CLI flags, `--debug` artifacts | `engine` |
| HTTP endpoints, persistence, job orchestration | `api` |
| User interface | `web` |
| Brand guide, logos, design tokens | this workspace (`docs/brand/`) |
| Repo list, commands, ports, workspace tooling | this workspace (`repos.conf`, `bin/desembrick`) |

A change to the output JSON shape starts in `engine` and must be followed
through to `api` and `web`.

## Before Editing

- Run `bin/desembrick status` before cross-repo edits.
- Edit application code only under `repos/<repo-folder>`.
- Do not edit sibling checkouts outside this workspace unless explicitly asked.
- Use `repos.conf` and `bin/desembrick status` to resolve repo paths.
- Create a repo-by-repo plan before changing files.
- For changes affecting more than one application repository, prefer one
  subagent per repository once the repo-by-repo plan is clear.
- Keep each subagent limited to its assigned `repos/<repo-folder>` checkout.
  The orchestrating agent owns dependency ordering, contract compatibility,
  cross-repo integration checks and the final summary.
- A single agent may handle very small or tightly coupled multi-repo changes
  when that is simpler, as long as repo ownership boundaries stay explicit.
- Read the affected repo's own `CLAUDE.md` before editing it.
- Do not run `pull` or `checkout` over dirty repos unless explicitly requested.
- Do not commit, push or deploy in any repository without the user's explicit
  approval.

## Change Guidance

- Prefer backward-compatible contract changes (above all the engine's JSON).
- Keep orchestration changes simple and Bash-only; `bin/desembrick` must keep
  running on the stock macOS Bash 3.2.
- Do not introduce Git submodules or runtime dependencies in this workspace.
- After each change, summarize what was done, risks or limitations, next
  steps, and a suggested commit message.

## Suggested Validation Order

1. `engine` — its test suite (`pytest`) and a run against a sample manual.
2. `api` — `npm test`, `npm run typecheck`.
3. `web` — `npm run build`.
4. `bin/desembrick status`, optionally `bin/desembrick diff --stat`.

For workspace changes: `bash -n bin/desembrick repos.conf desembrick.env`, then
`bin/desembrick help`, `list`, `current` and `status --all`.
