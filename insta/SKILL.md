---
name: insta
description: >
  Operate InstaCloud infrastructure with the `insta` CLI: create projects, add
  postgres/storage/compute services, deploy apps, create disposable branch
  environments (isolated DB + storage + compute per branch), bind service
  credentials into compute env, wire user secrets into `.env`, run multiple
  agents each in their own branch, handle governance
  approvals, check metrics/logs/usage, and promote branches to main. Use this
  skill when working in an InstaCloud-managed project (a `.insta/` dir or the
  `insta` CLI), when the user mentions InstaCloud or insta, AND when they ask to
  deploy an app, need a database/backend/object storage, want preview or
  per-agent sandbox environments, want branchable infrastructure, or mention
  agent setup or MCP — even if they don't say "InstaCloud" explicitly. Also
  covers the insta-cloud remote MCP server (insta_* tools) and the self-hosted
  insta-oss runtime (same CLI, local daemon).
allowed-tools: Bash(insta:*), Bash(npx:*), Bash(curl:*), Bash(command:*), Bash(git:*), Bash(npm:*)
---

# InstaCloud

InstaCloud provisions and governs a project's cloud services behind one CLI and one credential
seam. The `insta` CLI talks **only** to the InstaCloud control plane — you never configure a cloud
backend directly. A project can have any number of **services**, added on demand. The common service
types you build directly against are:

- **postgres** — relational DB born at its plan's resource ceiling (move it within the free cap on
  any plan with `insta db limits`; above the free cap needs a paid plan). Plain Postgres: connect any driver/ORM directly with the `DATABASE_URL`
  you bind into compute env (below) — no vendor SDK or vendor skill. The DB is also publicly
  dialable from outside compute: `insta db url` prints the connection string and
  `insta db connect` opens a psql session — that's how you (or a human) reach it from a laptop,
  a migration script, or any external tool. It scales to zero when
  idle, so keep your pool's `idleTimeoutMillis` under the suspend window (see
  [frameworks.md](references/frameworks.md)).
- **storage** — S3-compatible object/blob storage. Point any S3 library at the bound `AWS_*` /
  `BUCKET_NAME` env — no vendor SDK. Set the endpoint explicitly or the client talks to real AWS;
  each branch normally gets its own forked bucket (legacy pre-snapshot projects share one — see
  below). See [storage.md](references/storage.md).
- **compute** — your container(s) at a public URL. A project can have several compute services
  (e.g. `api`, `worker`).
- **redis/mysql/mongodb** — managed Fly-backed data services. They expose connection env names such
  as `REDIS_URL`, `MYSQL_URL`, and `MONGODB_URL`.

**A new project starts empty** — no services are created automatically. Add what you need:
`insta services add postgres <name>`, `insta services add compute <name>`,
`insta services add storage <name>`, `insta services add redis <name>`, etc. A project may have
**multiple services of every type** (up to 5 per type). Provider credentials are scoped to the
service that minted them and use canonical names inside that scope (`DATABASE_URL`, `REDIS_URL`,
`MYSQL_URL`, `MONGODB_URL`, `AWS_ACCESS_KEY_ID`, `BUCKET_NAME`, …). They do **not** automatically
appear in `insta secrets`, `insta run`, or compute env. Bind the credentials a compute service needs,
then deploy — or, if the service is already running, `insta compute restart` (CLI ≥ 0.0.51) to pick
the binding up without deploying a new one. It re-runs the image *reference* already recorded, so a
service on a moving tag (`app:latest`) still gets whatever that tag resolves to now — see
[operate.md](references/operate.md) before using it on production:

```bash
insta secrets sources                    # what's available to bind (--branch <b> targets another branch)
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta secrets bind REDIS_URL redis/cache --source-name REDIS_URL --to compute/app
insta secrets bind MYSQL_URL mysql/orders --source-name MYSQL_URL --to compute/app
insta secrets bind MONGODB_URL mongodb/catalog --source-name MONGODB_URL --to compute/app
insta deploy . --group app --port 8080
```

Binding is for **compute env** only. To use a credential yourself — run migrations, inspect data,
point a local tool at the DB — read the value directly: `insta db url` (postgres connection
string; `insta db connect` for a psql shell).

Use `insta services rename <type> <name> <new-name>` to rename a service; existing bindings keep
pointing at that service.

## Install & upgrade the CLI

If `command -v insta` finds nothing, install it (never assume it's present):

```bash
curl -fsSL https://raw.githubusercontent.com/InsForge/insta-cli/main/install.sh | sh  # native binary, no Node
npm install -g insta                                    # npm alternative
npx insta@latest <cmd>                                  # one-shot, always newest (slow per call)
```

The CLI is pre-1.0 and ships often. If a command misbehaves or is unrecognized, **update first**:
`insta upgrade` (CLIs that have it; auto-update is on by default pre-1.0 — `insta autoupdate off`
to disable), else re-run the installer (idempotent) or `npm update -g insta`.

## Two targets, one CLI

The same commands drive both. Resolve which one you're on from `insta status` (`api:` line):

- **InstaCloud (managed cloud)** — requires `insta login` (agents: `--email/--password` or an API
  token; humans: bare `insta login` opens the console sign-in/approval page in the browser — any
  account type; headless machine with a human reachable elsewhere: `--device` prints a link + code
  they approve from any other browser).
- **insta-oss (self-hosted local daemon)** — `INSTA_API_URL=http://127.0.0.1:8080` (its default).
  **No login exists or is needed** (localhost trust, builtin `local` user); billing/usage/metrics
  return clear "cloud-only" errors — don't retry them.

## Tool routing: CLI vs MCP

InstaCloud has two agent-facing operation paths. **This skill + the CLI is the default for
everything** — the CLI covers the full command surface (bar a few MCP-only read-only
diagnostics, listed in mcp.md), carries linked-repo context
(`.insta/project.json`), and is the only path for local machine state: auth (`insta login`),
pulling secret **values** (`insta secrets` / `insta run`, and the postgres DSN via
`insta db url` / `insta db connect`), source-directory deploys (`insta deploy <dir>`), and the
observe hook.

Fall back to the **remote MCP tools** (`insta_*`) only when the CLI can't be invoked: no shell
(hosted agents like Claude.ai / ChatGPT connectors), or the CLI isn't installed and can't be
(the common case is fixable with no CLI on PATH: `npx -y insta@latest setup agent -y`
self-installs it — see self-heal below). Same platform API, same governance gates, same audit
trail — but MCP tools take **explicit `projectId`/`branch` args**: never assume the CLI's linked
context carries over; resolve IDs first (`insta_project_list`) and pass them explicitly. Full
mapping + connection guide: **[mcp.md](references/mcp.md)**.

**Self-heal:** if the insta skill or the `insta_*` MCP tools are expected but missing, run
`insta setup agent -y` (installs the skill + registers MCP for Claude Code and every detected
agent), then tell the user to **restart their coding tool** — a running session never picks up
newly registered MCP servers or tools. One specific agent: `insta mcp install --agent <slug>`.

## Intent-based routing

Route by intent before running preflight ceremony:

**"Ship / deploy this app" (from zero):** don't interrogate state first — run the chain and
announce it: `insta status` (logged in? linked?) → if unauthenticated on cloud, `insta login` → if
unlinked, `insta project create <dir-name>` → `insta services add postgres db` (if the app needs a
DB) + `insta services add compute app` → bind needed service credentials into compute
(`insta secrets sources`, then `insta secrets bind DATABASE_URL postgres/db --to compute/app`) →
`insta deploy . --port <the port the app listens on>` → **verify the printed URL serves** (below).
The app reads `process.env` creds.

**"Set up / onboard / sign up":** cloud → `insta login` (browser sign-in; relay the printed link
if no browser opens) or `--email/--password`; then `insta project create`. Local/oss → nothing to set up beyond the daemon.

**A unit of work on an existing project (feature, fix, experiment, agent task):** one branch per
unit of work — see the core principle below and **[branching.md](references/branching.md)**.
Never develop on `main`.

**Anything else (configure, debug, inspect):** light preflight, then the matching reference below.

## Preflight & context (before mutations)

```bash
command -v insta                 # installed? (else: Install section)
insta status --json              # target api, login, linked project, current branch
```

Skip this ceremony for the ship-from-zero chain above — `status` is its first step already.

**Context rules (multi-agent safety):**

- The link (`./.insta/project.json`) is **per directory** and includes the current branch.
- **Prefer explicit `--branch <name>`** on commands that accept it (`secrets`, `deploy`, `metrics`,
  `logs`, `events`, `db url` / `db connect` — a wrong-branch DSN means querying the wrong
  database) over `insta branch switch` when acting on a branch you don't own — `switch`
  mutates the shared per-directory link and races parallel agents in the same checkout.
- For parallel agents, the rule is **1:1:1 — task ↔ git worktree ↔ insta branch** (each worktree has
  its own link, so `switch` is safe there). See [branching.md](references/branching.md).

## Core principle

**One unit of work = one branch = one isolated environment.** `insta branch create <name>`
materializes the **parent branch's** current services onto the new branch — a CoW database branch
(copy of the parent's data), a CoW-forked storage bucket, and a clone of every compute service (own
URL each), created **at branch-create**, so a branch is a complete runnable environment from the
start.
Branches run fully in parallel; nothing one does touches another. **≤10 branches per project (hard
limit).** Don't develop on `main`; don't pile multiple features on one branch.

**Multiple independent features (or agent tasks) at once?** Give each its own branch **and its own
subagent** — isolated DB + storage + compute + URLs mean zero collision. See
**[branching.md](references/branching.md) → Parallel agents**.

## Verify before reporting (deploys)

**Never report a deploy as successful from the command exiting alone.** `insta deploy` prints the
branch URL on success — that means the platform accepted and rolled the machine, not that the app
serves:

1. Poll the printed URL (`curl -s -o /dev/null -w '%{http_code}'`) every ~3s for up to ~60s.
   Scale-to-zero branches (the default) cold-start on the first request — allow a slow first hit
   (always-on services skip this; see references/operate.md).
2. `200` (or the app's expected status) → deployed; report the URL.
3. Still failing → the ordered triage list in [operate.md](references/operate.md) (port mismatch
   and migration-gated startup account for most failures).
4. Report the exact failing state — never claim success you didn't observe.

## Approval relay (CRITICAL — gated actions)

Sensitive actions are gated at the credential boundary (`secrets.read`, `secrets.write`, `deploy`,
`project.delete`, `branch.delete`, `service.add/remove/scale/upgrade`; policy per action:
allow/deny/approve — `project.delete` and `service.remove` require approval by default). When a command returns
**"approval required" with an approval id**:

- **Relay it to the human immediately and verbatim** — the exact line to run:
  `insta approvals approve <id>` (add `--always` to also stop future prompts for that action).
  Don't summarize it away, don't retry the command, and don't report the task as failed without
  surfacing the approval first. Only an **admin** can approve.
- Grants are **single-use**: after approval, **re-run the original command**; the next occurrence
  prompts again unless policy was set to allow (`--always` / `insta policy set <action> allow`).
- **Never work around a gate** (e.g. by hand-editing state or bypassing the CLI) — the gate is the
  product's safety model. A `deny` policy is a hard no: report it, don't circumvent it.

## Common quick operations

```bash
insta status --json                          # target, login, link, current branch
insta manifest --json                        # agent-legible env view: every branch's services + URLs
insta services list --json                   # what exists on this project
insta run -- <cmd>                           # run with user-defined secrets injected (NOTHING on disk; --branch <b>)
insta secrets --print                        # user-defined secrets for the current branch (--branch <b>)
insta secrets sources --json                 # provider credential sources available to bind
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta secrets bindings --target compute/app --json
insta secrets set NAME value                 # user config (project-wide; --branch for overrides)
insta build . --port 8080                    # local pre-deploy build/readiness check
insta deploy . --port 8080                   # build (Dockerfile) + deploy to the current branch
insta deploy --image <ref> --port 8080       # prebuilt image instead
insta compute exec app -- printenv PORT      # one-shot command on live compute (no shell/stdin)
insta compute volume app --size 1Gi          # attach/grow persistent /data; mounts on next deploy
insta branch create feat && insta branch list --json
insta logs compute --limit 100 --json        # runtime logs (--branch <b>; also redis|mysql|mongodb; db is provider-limited)
insta logs compute --since 2h --json         # time window (--from/--to too) — a windowless read is ONE page (~100 lines)
insta metrics compute --json                 # service metrics (also redis|mysql|mongodb)
insta events --limit 50 --json               # audit + agent-event timeline
insta usage --json                           # cloud only (insta billing --json likewise)
insta approvals list --status pending        # outstanding gates
```

Use `--json` wherever you parse output.

## Routing

For anything beyond the quick operations, load the reference that matches the intent — one is
usually enough, two at most:

| Intent | Reference | Covers |
| --- | --- | --- |
| Create or connect things ("set up", "new project", "add a database/compute") | [setup.md](references/setup.md) | CLI install/upgrade, cloud vs oss target, auth, project, services, ship-from-zero |
| Ship code or manage releases | [deploy.md](references/deploy.md) · framework recipes: [frameworks.md](references/frameworks.md) | image vs source (remote build), `--port` semantics, explicit service credential binding, secrets at runtime, verify procedure, Dockerfile templates, custom domains |
| Branch environments, parallel agents, promotion ("preview env", "sandbox per task", "merge to main") | [branching.md](references/branching.md) | **the data-forking env model** (what actually clones), branch loop, 1:1:1 worktree pattern + dispatch brief, promotion, migration discipline |
| Approvals, policy, audit, credential scanning | [governance.md](references/governance.md) | gates catalog, the approval relay, events timeline, observe hook, agent audit patterns |
| Check health or debug failures | [operate.md](references/operate.md) | status/manifest triage, ordered deploy-failure list, metrics/logs, cloud-vs-oss differences |
| Command lookup | [cli-reference.md](cli-reference.md) | the full CLI catalog with flags and gates |
| Remote MCP tools ("connect a connector", `insta_*` tools available) | [mcp.md](references/mcp.md) | connecting clients, tool ↔ CLI mapping, what stays CLI-only |
| InstaCloud itself got in your way (bug, stale doc, missing feature, friction) | [cli-reference.md → Feedback](cli-reference.md#feedback) | `insta feedback` / `insta_feedback`: when to file, situation → type mapping |

If a request spans two areas ("deploy and check it's healthy"), load both and answer once.

## Two non-negotiables (wherever you are)

- **Prefer `insta run -- <cmd>`** for user-defined project/branch secrets — the bundle is fetched per
  invocation and injected into the child environment only; nothing is written to disk, so nothing can
  leak or be committed. Provider-minted service credentials are not in this bundle; bind them to a
  compute service with `insta secrets bind`, then deploy (or `insta compute restart` an already-running
  service, CLI ≥ 0.0.51 — a binding change never reaches a live machine on its own).
- When a file is genuinely needed, treat `./.env` (from `insta secrets`; auto-gitignored in git
  repos) as the **only** file-based source for user-defined secrets — never hardcode or print secret
  values. `DATABASE_URL`, `AWS_*` / `BUCKET_NAME`, `REDIS_*`, `MYSQL_*`, and `MONGODB_*` are service
  credentials that reach production compute only through explicit `insta secrets bind` rules. For
  direct use **outside** compute the sanctioned read is `insta db url` / `insta db connect`
  (postgres; gated `secrets.read`) — pipe it (`psql "$(insta db url)"`), never paste the DSN into
  files or code. Everything else runs where the credentials are bound (the app itself, or a
  one-shot `insta compute exec <svc> -- <cmd>`).
  User-set config belongs in `insta secrets set <NAME>` (project-wide) / `--branch` for branch
  overrides — never hand-edit `.env` values you want to persist.
- Track **every** schema change as a file under `migrations/` so it replays on a branch DB and again
  on `main` after a merge. **InstaCloud never merges databases — only migration files carry schema forward.**
  Migrations run where the DB credentials are bound: on the compute service, via
  `insta compute exec app -- <migrate-cmd>` (never as a startup gate — see
  [deploy.md](references/deploy.md)); or directly, with no compute involved:
  `psql "$(insta db url --branch <b>)" -f migrations/<file>.sql` (explicit `--branch` — the bare
  form reads the linked branch's DB). Match `psql` / `pg_dump` / `pg_restore` to the server's
  Postgres major first — `pg_version` on `insta services list --json --branch <b>` (same branch as
  the DSN); if the row has none, read the exact version instead (see
  [operate.md](references/operate.md)).

## Governance & audit (this is the platform's core differentiator)

The gate mechanics and the relay procedure are above; the observe credential-audit hook, the events
timeline, and agent audit patterns are in [governance.md](references/governance.md).

**Billing is by actual app usage** (vCPU·min / RAM GB·min actually consumed + storage + egress —
not machine size × hours). Scale-to-zero is the default, so idle services cost nearly nothing;
`always-on` (all plans: `insta compute always-on`, `insta db always-on`, or `--always-on` at
create) trades a small continuous RAM cost for zero cold starts — see
[operate.md](references/operate.md). **The paid levers are the resource CEILING** (`insta compute limits`,
`insta db limits` — per-machine size, see [operate.md](references/operate.md))
**and machine COUNT** (`insta services scale` — horizontal): a new service is born at its plan's
ceiling and free plans may move within the free cap but not above it, and stay at one machine —
beyond either is a 403 — `insta billing upgrade` first; `insta usage` /
`insta billing` show cycle usage and cost. One free org per user. Full flags in
[cli-reference.md](cli-reference.md).

## When InstaCloud itself gets in your way (feedback)

If you hit a hurdle that is **InstaCloud's fault** — a command that violates its documented
contract, skill/doc text that doesn't match reality, a missing capability, confusing UX — report
it with `insta feedback` (or the `insta_feedback` MCP tool), **then continue the user's task with
a workaround**. Never block on the report, and **never file feedback for problems in the app the
user is building** — this channel is only for the InstaCloud toolkit
(`--component cli|mcp|platform|skills|docs`). Full flags and the situation → type mapping:
[cli-reference.md → Feedback](cli-reference.md#feedback).

## Response format

For operational work, report: **what was done** (action + scope: project/branch/service), **the
result** (URLs, IDs, observed status — not assumed), and **what's next** (or that it's complete).
Include command output only where it helps.
