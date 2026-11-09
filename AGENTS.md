# Desembrick Workspace — Agent Guidance

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
| `engine` | `repos/desembrick-engine` | Python 3.12 CLI: detects bag pages and per-step parts boxes, aggregates per bag, writes PDF + JSON | active |
| `api` | `repos/desembrick-api` | Spring Boot backend that will drive the engine | planned |
| `web` | `repos/desembrick-web` | Vue frontend | planned |

## Context Dependencies

None are currently required. Add them here when enterprise or domain contracts
exist.

The one contract already known is the engine's output JSON, which the future
`api` will consume:

```json
{ "set": "...", "bags": [ { "number": 1, "pages": [1, 2], "parts": [ { "image": "path.png", "quantity": 4 } ] } ] }
```

The engine must keep working fully standalone: command line only, inputs via
arguments or JSON, outputs under `--out`, no database or external service.

## Dependency Order

1. `engine`
2. `api` (planned)
3. `web` (planned)

## Change Routing

| Change type | Primary repo |
|---|---|
| PDF reading, bag-page detection, step parts-box parsing, OCR of quantities | `engine` |
| Aggregation per bag, output PDF layout, output JSON shape | `engine` |
| Engine CLI flags, `--debug` artifacts | `engine` |
| HTTP endpoints, persistence, job orchestration | `api` (planned) |
| User interface | `web` (planned) |
| Repo list, commands, ports, workspace tooling | this workspace (`repos.conf`, `bin/desembrick`) |

A change to the output JSON shape starts in `engine` and must be followed
through to `api` once it exists.

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
- Read the affected repo's own `AGENTS.md` / `CLAUDE.md` before editing it.
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
2. `api` — build and tests (once it exists).
3. `web` — build (once it exists).
4. `bin/desembrick status`, optionally `bin/desembrick diff --stat`.

For workspace changes: `bash -n bin/desembrick repos.conf desembrick.env`, then
`bin/desembrick help`, `list`, `current` and `status --all`.
