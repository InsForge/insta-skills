# Setup

From zero to a linked project — CLI install, target selection, auth, project + services.

## Install / upgrade the CLI

```bash
# agent one-liner — CLI + the insta skill for every coding agent + MCP (preferred; any OS/shell,
# Node 18+; self-installs the CLI globally). ALWAYS targets prod (CLI >= 0.0.38 — switches a
# staging-leftover machine back, announced):
npx -y insta@latest setup agent
# staging is its own explicit one-liner (persists the env switch itself):
npx -y insta@latest setup agent --env staging
# no Node? macOS/Linux ONLY — never on native Windows (PowerShell's curl alias + WSL bash shim break it):
curl -fsSL agents.instacloud.com | sh
# staging curl route NOT LIVE YET — until its DNS ships, use the raw URL below:
curl -fsSL agents.staging.instacloud.com | sh
curl -fsSL https://raw.githubusercontent.com/InsForge/insta-cli/main/install.sh | sh -s -- --agents --staging -y
# CLI only:
curl -fsSL https://raw.githubusercontent.com/InsForge/insta-cli/main/install.sh | sh  # native binary, no Node; macOS/Linux
npm install -g insta            # npm alternative · one-shot: npx insta@latest <cmd>
insta setup agent               # add the agent skills later (user-global, all agents)
insta upgrade                   # self-update (auto-update is on by default pre-1.0)
insta autoupdate off            # opt out of auto-update
```

Misbehaving or unrecognized command → update first (re-run the installer — it's idempotent — or
`npm update -g insta`), then retry.

## Pick the target

| | InstaCloud (managed) | insta-oss (self-hosted) |
| --- | --- | --- |
| endpoint | platform API (login persists it) | `INSTA_API_URL=http://127.0.0.1:8080` (its default) |
| auth | required (below) | none — localhost trust, builtin `local` user |
| daemon | n/a | `git clone InsForge/insta-oss && npm i && npx tsx src/main.ts` (needs Docker) |

Managed InstaCloud has two environments — `prod` (default) and `staging` — which are **separate
deployments** with separate project lists and separate logins. `insta env` shows which one you're
on; `insta env use <name>` switches (dropping the session, since a token from one is not valid at
the other). Full table in [cli-reference.md](../cli-reference.md#environments).

`insta status --json` shows which target you're on (`env` + `apiUrl`) + login + linked project.

## Auth (cloud only)

- **Agents:** `insta login --email <e> --password <p>` (or `$INSTA_PASSWORD`), or an API token.
- **Humans:** bare `insta login` — opens the console approval page in the browser (any account
  type: email, GitHub, Google) and polls until approved; the link + code are also printed. If the
  environment can't open a browser, relay the printed link + code to the human **immediately and
  verbatim**; never sit on it silently. `--oauth github|google` still opens the named provider
  directly.
- **Headless machines (VM, SSH, CI — no browser on THIS machine):** `insta login --device` —
  prints a console link + code the human opens **on any other device** and approves; the CLI
  polls until logged in (~15 min window). Relay the printed link + code to the human immediately
  and verbatim. Don't use `--oauth` here: its loopback callback can never reach this machine.
- There is no login on insta-oss — don't try; `insta login` is a cloud-only command.

## Project

Linking is OPTIONAL (CLI ≥ 0.0.10): the first project-scoped command in an unlinked directory
auto-resolves — one project on the account is picked silently, several give a one-keystroke
picker — and persists to `./.insta/project.json` (commit it: teammates + CI inherit the binding,
and it resolves from any subdirectory, git-style). Resolution order:
`--project/--branch flags > INSTA_PROJECT_ID / INSTA_BRANCH / INSTA_ORG_ID env > link file (walk-up) > auto-resolve`.
Use env for CI/one-offs with no state; use `project link` only to pin a specific project.

```bash
insta project create <name> [--org <id>]   # creates an EMPTY project (cloud) and links this dir
insta project link <id>                    # pin a specific project explicitly (optional)
insta project list --org <id> --json
```

Linking writes `./.insta/project.json` (project + org + current branch, per directory) and
auto-installs the agent skills + the observe credential-audit hook into the repo.

**What to commit vs what the CLI gitignores (CLI ≥ 0.0.59):** commit `./.insta/project.json`
and the hook entries in `.claude/settings.json` / `.codex/hooks.json` (neither contains a
machine-specific path). The CLI adds the machine-local rest to `.gitignore` itself in the same
step that writes it: `.insta/observe/` (the generated hook copy) and `.insta/audit.jsonl` (this
machine's findings), the skill dirs `npx skills add` fills (`.claude/skills/`, `.agents/skills/`;
`.github/skills/` is listed defensively) and `skills-lock.json`. Only the `.insta/*` entries and
`skills-lock.json` are new in 0.0.59; the skill dirs were already ignored. Never ignore `.insta/`
wholesale (that hides the project binding), and don't "clean up" the entries or the ignored
files: a re-link regenerates `.insta/observe/` and the skills, while `.insta/audit.jsonl` is
append-only local findings, so `insta observe sync` before removing it. An ignore entry cannot
un-track a file that was committed earlier; the CLI reports those with a
`git rm -r --cached …` hint.

**Naming:** use the directory/repo name for the project; things like `api` or `worker` are
*service* names, not project names.

## Services (the project's resources)

A cloud project starts **empty**; add what the app needs (insta-oss auto-provisions one
postgres + one storage at create):

```bash
insta services add postgres db        # relational DB (size it with insta db limits)
insta services add storage files     # S3-compatible bucket
insta services add compute api       # your container; add --volume 1Gi now, or attach later
insta compute volume api --size 1Gi  # later attach/grow persistent /data; mounts on next deploy
insta services list --json
```

Compute volumes are **not create-time only**. Use `--volume <Gi>` when adding a compute service if
you already know it needs durable `/data`, or run `insta compute volume <service> --size <Gi>` later
on a volumeless service to attach one. The volume appears on the next deploy/redeploy.

Up to 5 services per type. Provider credentials are minted under the service that owns them with
canonical names (`DATABASE_URL`, `BUCKET_NAME`, `AWS_ACCESS_KEY_ID`, `REDIS_URL`, `MYSQL_URL`,
`MONGODB_URL`, …). They are not exported by `insta secrets` and are not injected into compute until
you bind them to a compute service with `insta secrets bind`. The postgres DSN (only) is also
directly readable — for a local psql, a migration, any tool outside compute — via `insta db url`
(prints it) or `insta db connect` (opens psql); match those client tools to the server's Postgres
major first (`pg_version` on `insta services list --json`, see [operate.md](operate.md)). The other
services' credentials have no direct read and reach code only through binding.

## Ship-from-zero (the whole chain)

```bash
insta status                       # target + auth + link in one look
insta login …                      # cloud only, if needed
insta project create myapp
insta services add postgres db && insta services add compute app   # cloud; oss has db+storage already
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta deploy . --port 8080         # or --image <ref>; then VERIFY the URL (see operate.md)
```
