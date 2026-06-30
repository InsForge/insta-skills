---
name: insta
description: Use when working in an InstaCloud-managed project (a `.insta/` dir or the `insta` CLI) — creating/switching branches, deploying to a branch's URL, wiring `insta secrets` into `.env`, running multiple agents each in their own isolated branch/env, handling governance approvals, or promoting a branch to main.
---

# InstaCloud

InstaCloud provisions and governs a project's cloud resources behind one CLI and one credential
seam. The `insta` CLI talks **only** to the InstaCloud control plane — you never configure a cloud
backend directly. Every project gets three resources you build **directly** against:

- **Postgres** (Neon) — relational DB (autoscaling compute + read replicas).
- **S3-compatible storage** (Tigris) — object/blob storage.
- **Compute** (Fly.io) — your container(s) at `https://<app>.fly.dev`. A project can have several
  **compute groups** (e.g. `default`, `backend`).

**Setup:** `insta login --email <e> --password <p>` → `insta project create <name>` (provisions all
three) or `insta project link <id>`. You land linked (`./.insta/project.json`) on branch `main`.
`insta status` shows login + linked project + current branch. `insta secrets` writes every
resource's credentials into `./.env` for local dev.

## Core principle

**One unit of work = one branch = one isolated environment.** `insta branch create <name>` gives
each feature, experiment, or agent task:

- its own **Neon DB branch** (copy-on-write copy of the parent's data, own `DATABASE_URL`),
- its own **copy-on-write storage bucket** (forked from the parent), and
- a **clone of every compute group** — one isolated Fly app + URL each.

All three are created **at branch-create** (InstaCloud clones the compute groups immediately — not
"on first deploy"), so a branch is a complete, runnable environment from the start. Branches run
fully in parallel; nothing one does touches another. **Don't develop on `main`; don't pile multiple
features on one branch.**

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
  stays **shared** — no isolation).
- Track **every** schema change as a file under `migrations/` so it replays on a branch DB and again
  on `main` after a merge. **InstaCloud never merges databases — only migration files carry schema forward.**

## Governance & audit (this is the platform's core differentiator)

Sensitive actions are gated at the credential boundary: `secrets.read`, `deploy`, `project.delete`,
`branch.delete`. Each has a per-project policy of `allow` / `deny` / `approve`.

- **`project.delete` requires approval by default.** A gated action returns *"approval required"*
  with an approval id; an **admin** runs `insta approvals approve <id>`, then you **re-run** the
  action (grants are single-use). `insta policy set <action> <decision>` changes the policy.
- The **`insta observe`** hook (auto-installed on `project create`/`link`) scans your agent tool-use
  for credential exposure into `./.insta/audit.jsonl`. `insta observe report` reviews it locally;
  `insta observe sync` uploads findings into the project's audit timeline (`insta events`).

**Observability:** `insta metrics <db|compute> [group]` and `insta logs <db|compute> [group]` show
resource metrics + runtime logs. Compute (Fly) is fully served; DB (Neon) has no realtime
metrics/logs API, so `component=db` returns a provider-limitation note.

**Usage & billing:** `insta usage` aggregates resource usage by meter (with `costUsd`); `insta billing`
shows the current cycle (tier / included credit / used / overage). `insta billing upgrade <pro|enterprise>`
and `insta billing portal` open Stripe in a browser (interactive). See `cli-reference.md`.
