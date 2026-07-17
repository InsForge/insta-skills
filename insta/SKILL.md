---
name: insta
description: >
  Operate InstaCloud infrastructure with the `insta` CLI: create projects, add
  postgres/storage/compute services, deploy apps, create disposable branch
  environments (isolated DB + storage + compute per branch), wire `insta secrets`
  into `.env`, run multiple agents each in their own branch, handle governance
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
backend directly. A project can have any number of **services**, added on demand — there are three
service types you build **directly** against:

- **postgres** — relational DB (autoscaling compute + read replicas).
- **storage** — S3-compatible object/blob storage.
- **compute** — your container(s) at a public URL. A project can have several compute services
  (e.g. `api`, `worker`).

**A new project starts empty** — no services are created automatically. Add what you need:
`insta services add postgres <name>`, `insta services add compute <name>`,
`insta services add storage <name>`. A project may have **multiple services of every type** (up to
5 per type). Credentials are named per service: `DATABASE_URL_<NAME>`, `BUCKET_NAME_<NAME>`, …
(service name upper-snaked); the **oldest** service of each type also gets the plain names
(`DATABASE_URL`, `BUCKET_NAME`, …), so single-service projects work unchanged.

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
  token; humans: `--oauth github|google`, opens a browser).
- **insta-oss (self-hosted local daemon)** — `INSTA_API_URL=http://127.0.0.1:8080` (its default).
  **No login exists or is needed** (localhost trust, builtin `local` user); billing/usage/metrics
  return clear "cloud-only" errors — don't retry them.

## Tool routing: MCP vs CLI

InstaCloud has two agent-facing operation paths. Choose in this order:

1. **CLI** for anything that needs local machine state: auth (`insta login`), pulling secret
   **values** (`insta secrets` / `insta run`), source-directory deploys (`insta deploy <dir>`),
   the observe hook, and repo-linked context (`.insta/project.json`).
2. **Remote MCP** (`insta_*` tools, when connected) as the default for platform-scoped
   operations that don't need local files: discovery, services, branches, image deploys,
   metrics/logs/usage, governance. Structured JSON, same governance gates, same audit trail.
3. No MCP connected → the CLI covers everything.

MCP tools take **explicit `projectId`/`branch` args** — never assume the CLI's linked context
carries over; resolve IDs first (e.g. `insta_project_list`, or `insta status --json` locally) and
pass them explicitly. `insta setup agent` registers the server with Claude Code automatically.
Full mapping + connection guide: **[mcp.md](references/mcp.md)**.

## Intent-based routing

Route by intent before running preflight ceremony:

**"Ship / deploy this app" (from zero):** don't interrogate state first — run the chain and
announce it: `insta status` (logged in? linked?) → if unauthenticated on cloud, `insta login` → if
unlinked, `insta project create <dir-name>` → `insta services add postgres db` (if the app needs a
DB) + `insta services add compute app` → `insta secrets` → `insta deploy . --port <the port the
app listens on>` → **verify the printed URL serves** (below). The app reads `process.env` creds.

**"Set up / onboard / sign up":** cloud → `insta login --oauth github` (browser) or
`--email/--password`; then `insta project create`. Local/oss → nothing to set up beyond the daemon.

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
  `logs`, `events`) over `insta branch switch` when acting on a branch you don't own — `switch`
  mutates the shared per-directory link and races parallel agents in the same checkout.
- For parallel agents, the rule is **1:1:1 — task ↔ git worktree ↔ insta branch** (each worktree has
  its own link, so `switch` is safe there). See [branching.md](references/branching.md).

## Core principle

**One unit of work = one branch = one isolated environment.** `insta branch create <name>`
materializes the **parent branch's** current services onto the new branch — a Neon DB branch (CoW
copy of the parent's data), a CoW-forked storage bucket, and a clone of every compute service (own
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
   Scale-to-zero branches cold-start on the first request — allow a slow first hit.
2. `200` (or the app's expected status) → deployed; report the URL.
3. Still failing → the ordered triage list in [operate.md](references/operate.md) (port mismatch
   and migration-gated startup account for most failures).
4. Report the exact failing state — never claim success you didn't observe.

## Approval relay (CRITICAL — gated actions)

Sensitive actions are gated at the credential boundary (`secrets.read`, `secrets.write`, `deploy`,
`project.delete`, `branch.delete`, `service.add/remove/scale/upgrade`; policy per action:
allow/deny/approve — `project.delete` requires approval by default). When a command returns
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
insta run -- <cmd>                           # run with the branch bundle injected (NOTHING on disk; --branch <b>)
insta secrets --print                        # credential bundle for the current branch (--branch <b>)
insta secrets set NAME value                 # user config (project-wide; --branch for overrides)
insta deploy . --port 8080                   # build (Dockerfile) + deploy to the current branch
insta deploy --image <ref> --port 8080       # prebuilt image instead
insta branch create feat && insta branch list --json
insta logs compute --limit 100 --json        # runtime logs (--branch <b>; db is provider-limited)
insta metrics compute --json                 # service metrics
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
| Ship code or manage releases | [deploy.md](references/deploy.md) · framework recipes: [frameworks.md](references/frameworks.md) | image vs source (remote build), `--port` semantics, secrets at runtime, verify procedure, Dockerfile templates, custom domains |
| Branch environments, parallel agents, promotion ("preview env", "sandbox per task", "merge to main") | [branching.md](references/branching.md) | **the data-forking env model** (what actually clones), branch loop, 1:1:1 worktree pattern + dispatch brief, promotion, migration discipline |
| Approvals, policy, audit, credential scanning | [governance.md](references/governance.md) | gates catalog, the approval relay, events timeline, observe hook, agent audit patterns |
| Check health or debug failures | [operate.md](references/operate.md) | status/manifest triage, ordered deploy-failure list, metrics/logs, cloud-vs-oss differences |
| Command lookup | [cli-reference.md](cli-reference.md) | the full CLI catalog with flags and gates |
| Remote MCP tools ("connect a connector", `insta_*` tools available) | [mcp.md](references/mcp.md) | connecting clients, tool ↔ CLI mapping, what stays CLI-only |

If a request spans two areas ("deploy and check it's healthy"), load both and answer once.

## Two non-negotiables (wherever you are)

- **Prefer `insta run -- <cmd>`** — the bundle is fetched per invocation and injected into the
  child environment only; nothing is written to disk, so nothing can leak or be committed.
- When a file is genuinely needed, treat `./.env` (from `insta secrets`; auto-gitignored in git
  repos) as the **only** credential source — never hardcode or print
  secret values. `DATABASE_URL` + compute + storage (`AWS_*` / `BUCKET_NAME`) are all **per-branch**
  (each branch copy-on-write-forks its parent's bucket; a legacy bucket created without snapshots
  stays **shared** — no isolation). User-set config belongs in `insta secrets set <NAME>`
  (project-wide) / `--branch` for branch overrides — never hand-edit `.env` values you want to
  persist; a redeploy re-injects the platform's view.
- Track **every** schema change as a file under `migrations/` so it replays on a branch DB and again
  on `main` after a merge. **InstaCloud never merges databases — only migration files carry schema forward.**

## Governance & audit (this is the platform's core differentiator)

The gate mechanics and the relay procedure are above; the observe credential-audit hook, the events
timeline, and agent audit patterns are in [governance.md](references/governance.md).

**Scaling & billing (cloud, paid plans):** `insta services scale/upgrade` (free plans get 403 —
`insta billing upgrade` first); `insta usage` / `insta billing` show cycle usage and cost. One free
org per user. Full flags in [cli-reference.md](cli-reference.md).

## Response format

For operational work, report: **what was done** (action + scope: project/branch/service), **the
result** (URLs, IDs, observed status — not assumed), and **what's next** (or that it's complete).
Include command output only where it helps.
