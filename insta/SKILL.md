---
name: insta
description: Use when working in an InstaCloud-managed project (a `.insta/` dir or the `insta` CLI) — creating/switching branches, deploying to a branch's URL, wiring `insta secrets` into `.env`, running multiple agents each in their own isolated branch/env, handling governance approvals, or promoting a branch to main.
---

# InstaCloud

InstaCloud provisions and governs a project's cloud services behind one CLI and one credential
seam. The `insta` CLI talks **only** to the InstaCloud control plane — you never configure a cloud
backend directly. A project can have any number of **services**, added on demand — there are three
service types you build **directly** against:

- **postgres** (Neon) — relational DB (autoscaling compute + read replicas).
- **storage** (Tigris, S3-compatible) — object/blob storage.
- **compute** (Fly.io) — your container(s) at `https://<app>.fly.dev`. A project can have several
  compute services (e.g. `api`, `worker`).

**A new project starts empty** — no services are created automatically. Add what you need:
`insta services add postgres <name>`, `insta services add compute <name>`,
`insta services add storage <name>` (postgres/compute get a default access domain). `insta services
list` / `insta services remove <type> <name>` manage them.

A project may have **multiple services of every type** (up to 5 per type). Credentials are named
per service: `DATABASE_URL_<NAME>`, `BUCKET_NAME_<NAME>`, … (service name upper-snaked). The
**oldest** service of each type also gets the plain names (`DATABASE_URL`, `BUCKET_NAME`, …), so
single-service projects work unchanged.

**Setup:** `insta login --email <e> --password <p>` → `insta project create <name>` (empty project)
or `insta project link <id>`. You land linked (`./.insta/project.json`) on branch `main`. Then add
services. `insta status` shows login + linked project + current branch. `insta secrets` writes every
service's credentials into `./.env` for local dev.

## Core principle

**One unit of work = one branch = one isolated environment.** `insta branch create <name>`
materializes the project's **current services** onto the new branch — for each service the project
has:

- a **Neon DB branch** (copy-on-write copy of the parent's data, own `DATABASE_URL`) per postgres service,
- a **copy-on-write storage bucket** (forked from the parent) per storage service, and
- a **clone of every compute service** — one isolated Fly app + URL each.

These are created **at branch-create** (InstaCloud clones the compute services immediately — not
"on first deploy"), so a branch is a complete, runnable environment from the start. A project with no
services yields an empty branch; adding a service later materializes it onto every existing branch.
Branches run fully in parallel; nothing one does touches another. **A project may have at most 10
branches (a hard system limit).** **Don't develop on `main`; don't pile multiple features on one
branch.**

**Multiple independent features (or agent tasks) at once?** Give each its own branch **and its own
subagent** — isolated DB + storage + compute + URLs mean zero collision. This is the recommended way
to parallelize agent work — see **workflow.md → Running multiple agents in parallel**.

## Where to go

- **Doing the work** → **workflow.md**: the branch loop, parallel agents (a git worktree + an insta
  branch each), promoting a branch to main (merge → migrate → redeploy → validate), and the deploy
  gotchas that bite.
- **Command lookup** → **cli-reference.md**: the full CLI catalog, deploy, Dockerfile templates, and
  the govern/observe commands.

## Two non-negotiables (wherever you are)

- Treat `./.env` (from `insta secrets`) as the **only** credential source — never hardcode or print
  secret values. `DATABASE_URL` + compute + storage (`AWS_*` / `BUCKET_NAME`) are all **per-branch**
  (each branch copy-on-write-forks its parent's bucket; a legacy bucket created without snapshots
  stays **shared** — no isolation). User-set config belongs in `insta secrets set <NAME>`
  (project-wide) / `--branch` for branch overrides — never hand-edit `.env` values you want to
  persist; a redeploy re-injects the platform's view.
- Track **every** schema change as a file under `migrations/` so it replays on a branch DB and again
  on `main` after a merge. **InstaCloud never merges databases — only migration files carry schema forward.**

## Governance & audit (this is the platform's core differentiator)

Sensitive actions are gated at the credential boundary: `secrets.read`, `secrets.write`, `deploy`,
`project.delete`, `branch.delete`, and the service mutations `service.add` / `service.remove` / `service.scale` /
`service.upgrade`. Each has a per-project policy of `allow` / `deny` / `approve`.

- **`project.delete` requires approval by default.** A gated action returns *"approval required"*
  with an approval id; an **admin** runs `insta approvals approve <id>`, then you **re-run** the
  action (grants are single-use). `insta policy set <action> <decision>` changes the policy.
- The **`insta observe`** hook (auto-installed on `project create`/`link`) scans your agent tool-use
  for credential exposure into `./.insta/audit.jsonl`. `insta observe report` reviews it locally;
  `insta observe sync` uploads findings into the project's audit timeline (`insta events`).

**Scaling (paid plans only):** `insta services scale compute <name> <number> [region]` sets a compute
service's machine count; `insta services upgrade <compute|postgres> <name> <spec>` raises its spec
(compute guest / postgres CU ceiling; up-only). New services start at the provider **minimum** spec.
**Free-plan projects cannot scale or upgrade** (403) — upgrade the org first (`insta billing upgrade`).

**Observability:** `insta metrics <db|compute> [group]` and `insta logs <db|compute> [group]` show
service metrics + runtime logs. Compute (Fly) is fully served; DB (Neon) has no realtime metrics/logs
API, so `db` returns a provider-limitation note.

**Usage & billing:** `insta usage` aggregates service usage by meter (with `costUsd`); `insta billing`
shows the current cycle (tier / included credit / used / overage). `insta billing upgrade <pro|enterprise>`
and `insta billing portal` open Stripe in a browser (interactive). **A user may own only one free org**
— to create another org, upgrade an existing one to a paid tier first. See `cli-reference.md`.
