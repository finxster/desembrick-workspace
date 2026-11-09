# Desembrick Workspace

A lightweight, Bash-only orchestration workspace for the **Desembrick** project
(short alias: `dbrick`). It is not an application repository: it clones,
inspects, builds and plans work across the Desembrick repositories, which live
as plain checkouts under `repos/`.

Desembrick turns an official Lego instruction manual (PDF) into a "reverse
manual": the parts of each numbered bag, so a mixed set can be sorted back into
its original bags.

## Repositories

| Key | Folder | What it is | Status |
|---|---|---|---|
| `engine` | `desembrick-engine` | Python processing engine (CLI): PDF manual in, parts-per-bag PDF + JSON out | configured |
| `api` | `desembrick-api` | Spring Boot backend | planned, not created yet |
| `web` | `desembrick-web` | Vue frontend | planned, not created yet |

Only `engine` is active in `repos.conf`. `api` and `web` are commented
placeholders there; uncomment them when the repositories exist.

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

**No runnable apps are configured yet.** The engine is a one-shot CLI, not a
long-lived process, so it is not in `DESEMBRICK_APP_KEYS`; until `api` or `web`
is added there, the commands above exit with
`no runnable apps configured`. Run the engine directly from its checkout
(`cd repos/desembrick-engine && uv run python -m desembrick input.pdf --out ./out/`)
or through `dbrick foreach '<command>' engine`.

No `reset-db` / `fresh` commands are generated: nothing in this workspace owns
a local database yet.

## Agent workflow

- Start from `desembrick-workspace`.
- Run `dbrick status` before cross-repo edits.
- Create a repo-by-repo plan before editing (save longer plans under `plans/`).
- Read the root `AGENTS.md` and the affected repo's own `AGENTS.md` / `CLAUDE.md`.
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
