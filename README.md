<p align="center"><img src="docs/brand/svg/logo.svg" alt="Desembrick" width="320"></p>

# Desembrick Workspace

A lightweight, Bash-only orchestration workspace for the **Desembrick** project
(short alias: `dbrick`). It is not an application repository: it clones,
inspects, builds and plans work across the Desembrick repositories, which live
as plain checkouts under `repos/`.

Desembrick turns an official Lego instruction manual (PDF) into a "reverse
manual": the parts of each numbered bag, so a mixed set can be sorted back into
its original bags.

See [docs/architecture.md](docs/architecture.md) for an ASCII architecture
diagram of the online flow, the engine pipeline and local development.

## Repositories

| Key | Folder | What it is | Status |
|---|---|---|---|
| `engine` | `desembrick-engine` | Python processing engine (CLI): PDF manual in, parts-per-bag PDF + JSON out; GitHub Actions runner | configured |
| `api` | `desembrick-api` | Cloudflare Worker (TypeScript, Hono): jobs, uploads to R2, dispatches the runner | configured |
| `web` | `desembrick-web` | Vue 3 + Vite SPA on Cloudflare Pages | configured |

## Setup

1. Review `repos.conf`.
2. Run `bin/desembrick list`.
3. Optionally source `desembrick.env` to get the `desembrick` and `dbrick`
   commands from any directory.
4. Clone the repositories: `bin/desembrick clone`.

### Shell setup

Zsh (`~/.zshrc`):

```zsh
source /path/to/desembrick-workspace/desembrick.env
```

Bash (`~/.bashrc` or `~/.bash_profile`):

```bash
source /path/to/desembrick-workspace/desembrick.env
```

After that, `desembrick <command>` and `dbrick <command>` are equivalent to
`bin/desembrick <command>`.

## Common local workflows

```bash
dbrick clone                          # clone every configured repo
dbrick status --all                   # where every repo stands
dbrick status && dbrick pull          # sync everything before working
dbrick checkout -b my-feature engine  # new branch carrying local changes
dbrick foreach 'uv run pytest' engine # run a command inside a repo
```

## Run everything locally

No Cloudflare or GitHub account needed: the Worker uses a local R2, and the
engine's local runner plays the part of GitHub Actions:

```bash
dbrick start          # engine runner :8790, api :8787, web :5173
dbrick ps             # what is running
dbrick logs api -f    # follow one app's log
dbrick stop
```

The first `start` creates `repos/desembrick-api/.dev.vars` from the example, which
points the api at the local runner.

Open http://localhost:5173, upload a manual, wait, download.

## Deploy (all free tiers)

1. **Cloudflare R2:** create bucket `desembrick-jobs` with two lifecycle rules,
   prefix `jobs/` deleted after 1 day and prefix `audit/` after 90 days (not a
   whole-bucket rule), and an R2 API token (Object Read & Write, this bucket only). Note the S3 endpoint `https://<account-id>.r2.cloudflarestorage.com`.
2. **GitHub, `desembrick-engine`:** add repo secrets `R2_ACCESS_KEY_ID`,
   `R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT`, `R2_BUCKET`, and push `main` (the
   workflow must be on the default branch to be dispatched). Test it with
   *Actions → process → Run workflow* after uploading a PDF to `jobs/<id>/input/1.pdf`.
3. **GitHub token:** fine-grained token, repository `desembrick-engine` only,
   permission *Actions: Read and write*, with an expiry date (rotate it).
   **Turnstile:** create a widget in **Managed** mode for the Pages domain; note
   the site key and secret.
4. **api:** `cd repos/desembrick-api && npx wrangler login`,
   `npx wrangler secret put` for `GITHUB_TOKEN`, `TURNSTILE_SECRET` and
   `AUDIT_SALT` (a long random string), then `npm run deploy`. Note the
   `*.workers.dev` URL.
5. **web:** Cloudflare Pages → connect `desembrick-web` (build `npm run build`,
   output `dist`, env `VITE_API_BASE=<worker URL>` and
   `VITE_TURNSTILE_SITE_KEY=<site key>`). Note the `*.pages.dev` URL.
6. **Close the loop:** set `ALLOWED_ORIGIN` in `repos/desembrick-api/wrangler.toml`
   to the Pages URL (comma-separate to keep localhost) and `npm run deploy` again.

### Abuse protection and audit

- `POST /api/jobs` needs a Turnstile token and is limited to 5 per minute per IP.
- Every api event is one JSON line in Workers Logs (dashboard → Workers → Logs).
- Per job, `audit/<id>/api.json` (when, hashed IP, country, user agent, upload
  sizes) and `audit/<id>/runner.json` (Actions run id and URL, input sha256,
  set, outcome) are kept for 90 days. IPs are only stored as salted hashes (LGPD).
- The engine repo is private, so Actions logs are visible to collaborators only.

## Commands

### Repositories

```bash
dbrick list                           # key, folder, branch (or "missing")
dbrick clone                          # HTTPS (default), all repos
dbrick clone --ssh engine             # SSH, one repo

dbrick status                         # missing and dirty repos only
dbrick status --all --details         # every repo, with git status --short
dbrick status --porcelain             # stable "<key>\t<state>\t<branch>" for scripts
dbrick diff                           # tracked diff vs HEAD + untracked files
dbrick diff --stat engine
dbrick branch

dbrick fetch
dbrick pull                           # fast-forward only; refuses dirty/detached
dbrick checkout feature/x             # explicit branch, all repos
dbrick checkout feature/x engine      # explicit branch, one repo
dbrick checkout -b feature/y engine   # create, keeping local changes
dbrick checkout -b feature/y --from origin/main engine

dbrick foreach 'git log --oneline -3'
dbrick foreach 'uv run pytest' engine
```

### Saved defaults (`current`)

```bash
dbrick current                        # show saved targets and runtime defaults
dbrick current set feature/x          # save a branch target for all repos
dbrick current set feature/x engine   # ...or for selected repos
dbrick checkout                       # switch each repo to its saved target
dbrick current clear                  # forget the saved targets

dbrick current applications api web   # default apps for start/stop/restart/ps
dbrick current applications clear
dbrick current log api                # default app for logs
dbrick current log clear
```

`current sprint` is intentionally not generated: it relies on a
library/application repo split that this project does not have.

### Build and publish

```bash
dbrick build                          # run each repo's configured build command
dbrick build engine
dbrick publish
```

Both skip repos with no configured command. `build engine` runs the engine's
test suite (`uv run pytest`, which needs [uv](https://docs.astral.sh/uv/)).
There are no library repos, so `publish` has nothing to do.

### Runnable apps

```bash
dbrick start                          # saved current applications, else all apps
dbrick start -f api                   # start one app and follow its log
dbrick start api env=local            # needs DESEMBRICK_APP_ENV_ARG_api
dbrick start api port=8081            # needs DESEMBRICK_APP_PORT_ARG_api
dbrick ps
dbrick logs api
dbrick logs -f --lines 100 api
dbrick logs api port=8081
dbrick stop api
dbrick restart api
```

Apps: `engine` (8790) runs `runner/local_server.py`, the local stand-in for
GitHub Actions; `api` (8787) is `wrangler dev`; `web` (5173) is Vite. The engine
CLI itself still runs directly:
`cd repos/desembrick-engine && uv run python -m desembrick input.pdf --out ./out/`.

No `reset-db` / `fresh` commands are generated: nothing in this workspace owns
a local database yet.

## Agent workflow

- Start from `desembrick-workspace`.
- Run `dbrick status` before cross-repo edits.
- Create a repo-by-repo plan before editing (save longer plans under `plans/`).
- Read the root `CLAUDE.md` and the affected repo's own `CLAUDE.md`.
- Split subagents by repo ownership when a change spans several repos.
- Do not revert unrelated changes.
- Validate in dependency order: `engine`, then `api`, then `web`.
- Finish with `dbrick status` and optionally `dbrick diff --stat`.

## Notes

- **Repo keys and aliases.** Commands take the repo key (`engine`), a short
  alias from `DESEMBRICK_ALIASES` (`eng`), the folder name
  (`desembrick-engine`), or the folder without its `desembrick-` prefix.
- **Clone modes.** `clone` defaults to HTTPS; pass `--ssh` for SSH URLs.
- **Dirty-worktree guards.** `pull` and `checkout` refuse repos with
  uncommitted changes (`checkout -b` is allowed, since creating a branch keeps
  them). `pull` also refuses a detached HEAD and is fast-forward only. The
  script never runs `reset --hard` or `clean`.
- **`-f` is shorthand for `--follow`** on `start` and `logs`, and supports
  exactly one app.
- **Runtime files.** Pid files go to `.desembrick/run/`, logs to
  `.desembrick/logs/`. A `port=<n>` run gets `<app>-<n>.pid` / `<app>-<n>.log`.
- **Saved `current` state** lives in `.desembrick/current*.conf`. It is local,
  git-ignored state — one developer's working set, not configuration.
- **`build` / `publish`** skip repos without a configured command.
- **No submodules.** Repositories are ordinary clones under `repos/`, ignored
  by this workspace's git.
- The script targets the stock macOS Bash 3.2 and needs only Bash, Git and
  standard Unix tools (`ps`, `lsof`, `tail`, `sed`, `awk`).
