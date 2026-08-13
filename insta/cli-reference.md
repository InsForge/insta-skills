# InstaCloud CLI reference

Command catalog with flags and gates. Task guidance lives in `references/` — setup, deploy,
branching, governance, operate, mcp.

## Commands

| Command | Purpose |
|---|---|
| `insta login --email <e> --password <p>` [`--api-url <url>`] [`--env <prod\|staging>`] · `insta login --oauth <github\|google>` · `insta login --device` · `insta logout` | auth (api-url + tokens persist; tokens auto-refresh). `--oauth` opens a browser (loopback capture) — for interactive use on a machine with a browser. `--device` is the headless path (VMs, SSH, CI): it prints a console link + code, the human approves **from a browser on any other machine**, and the CLI polls until logged in (~15 min window). Agents can also use email/password or an API token. `--env` targets a named deployment (see [Environments](#environments)); `--api-url` wins if both are given |
| `insta env` [`--json`] · `insta env use <prod\|staging>` | show or switch the deployment environment. **Switching drops the stored session** — prod and staging are separate deployments, so the old token cannot authenticate. See [Environments](#environments) |
| `insta status` [`--json`] | environment + login + linked project + current branch |
| `insta org list` [`--json`] · `insta org create <name>` | organizations (**one free org per user** — upgrade an existing org before creating another) |
| `insta project create <name>` [`--org <id>`] | create an **empty** project (no services), link this dir |
| `insta project list` [`--org`] [`--json`] · `insta project link <id>` | list / link existing |
| `insta project delete` | tear down ALL resources + unlink (gated: `project.delete`, approval by default) |
| `insta services add <postgres\|storage\|compute> <name>` [`--branch <b>`] [`--region <r>`] [`--image <url>`] [`--port <n>`] [`--always-on`] [`--volume <gi>`] | provision a service **on a branch** (default: current/linked branch) — services are **branch-scoped**: adding one on a branch does not add it to any other branch; postgres/compute get a default access domain (gated: `service.add`). **compute only:** `--image` runs that image immediately at creation (otherwise compute starts as an empty, unreachable app until `insta deploy`); `--port` sets the listening port (default `8080`); `--always-on` creates it pinned-warm (never scales to zero — all plans, billing is actual usage either way); `--volume <gi>` attaches a persistent `/data` volume (also attachable later via `insta compute volume --size` — see [Volumes](#volumes)). The image is **persisted** on the service — shown in `services list`, re-run when the branch is forked, and updated later via `insta deploy --image`. Both positional arguments are optional in the parser: a **terminal** with either missing is walked through the kinds and then a name, while **anything without a TTY errors** with the kind list and creates nothing — so always pass both here |
| `insta regions` [`--json`] | list regions available for postgres/compute services |
| `insta services list` [`--json`] [`--branch <b>`] · `insta services rename <type> <name> <new-name>` [`--json`] [`--branch <b>`] · `insta services remove <type> <name>` [`--branch <b>`] | list / rename / remove a branch's services (default: current branch; rename re-keys managed secret names; gated: `service.rename` / `service.remove`) |
| `insta services secrets <type> <name>` [`--branch <b>`] [`--json`] | secret **names** bound to one service (e.g. `insta services secrets postgres db`) — default: current branch |
| `insta services scale compute <name> <number>` [`region`] | set compute machine count — **paid plans only** (free → 403); gated: `service.scale`. `region` is an InstaCloud slug (e.g. `us-east`; see `insta regions`), **not** a raw Fly code |
| `insta compute limits [service]` [`--memory <size>`] [`--cpu <n>`] [`--branch <b>`] [`--json`] | show or set a compute service's **resource ceiling** — **paid plans**. Bare = read (prints the ceiling **and** the plan max). Setting **requires `--memory`** (`512mb`, `1gb` — decimal `mb/gb` and binary `Mi/Gi` suffixes both accepted); cpu derives from it, and `--cpu` is only an optional override for parallel workloads (never valid alone). Moves **both directions**: billing is actual usage, so the ceiling caps what the app may burn — it is not a price |
| `insta db limits` [`--cpu <n>`] [`--memory <size>`] [`--branch <b>`] [`--group <g>`] [`--json`] | same ceiling control for a postgres service — **paid plans**; both directions. Takes provider quantities (`--cpu 2` or `2500m`; `--memory 4Gi`); either flag alone works. Bare = read the current ceiling |
| `insta compute volume [service]` [`--size <Gi>`] [`--delete`] [`--branch <b>`] [`--json`] | show, **attach**, grow, or **delete** a compute service's persistent `/data` volume. Bare = read (size, mount path, plan cap — any plan). `--size` on a volumeless service **attaches** one (any plan at the default 1Gi; mounts on the next deploy); on a volume-bearing one it grows — **paid plans, grow-only** (a provisioned disk cannot shrink). `--delete` **destroys the disk and ALL its data immediately** (any plan; irreversible; no detach exists — billing stops now, and suspend fast-wake + scale-out return; gated `service.remove`). See [Volumes](#volumes) |
| `insta db volume` [`--size <Gi>`] [`--branch <b>`] [`--group <g>`] [`--json`] | show or grow a postgres service's provisioned volume (block disk). Bare = read (size + plan cap — any plan); `--size` grows it — **paid plans, grow-only**. Postgres has its volume by default — there is nothing to attach |
| `insta services upgrade <compute\|postgres> <name> <spec>` | **legacy** (pre-usage-billing): raise a named spec, up-only — **paid plans only**; gated: `service.upgrade`. Prefer `insta compute limits` (and, for postgres, `insta db limits`), which also lower. **Compute-only in practice**: a postgres `upgrade` is rejected outright (400) — resize a postgres service with `insta db limits` (cpu/memory) and grow its disk with `insta db volume` |
| `insta branch create <name>` [`--from <parent>`] | isolated env: **forks the parent branch's current services** — a CoW database branch per postgres, a CoW-forked bucket per storage (snapshot-enabled projects), a clone of every compute service (re-running the parent's persisted image, if any) — then the two branches' service catalogs diverge independently (services are **branch-owned, not project-wide**). **≤10 branches/project.** Does NOT switch |
| `insta branch switch <name>` · `insta branch list` [`--json`] | set current branch / list |
| `insta branch merge <source>` [`--into <target>`] | **structural** merge: creates on the target branch (default: current) every service present on `<source>` but missing there — fresh & **empty, no data copied**. Services the target already has are skipped (reason: `exists`\|`cap`\|`secret-collision`). Additive only — never deletes target services; idempotent |
| `insta branch delete <name>` | tear down the branch's resources (gated: `branch.delete`) |
| `insta secrets` [`--branch <name>`] [`-o <file>`] [`--print`] [`--json`] | secret seam → write creds to `./.env` (gated: `secrets.read`) |
| `insta secrets list` [`--branch <b>`] [`--json`] | secret names for the branch, **grouped by service** — each service's bound secrets, plus a branch-level "unbound" group and a project-wide group |
| `insta secrets tree` [`--json`] | the whole project as `project → branch → service → secrets` (names only) |
| `insta secrets set <NAME> [value] [--branch <b>] [--service <type/name>]` | Set a user secret (project-wide by default; value from stdin if omitted). `--service` binds it to that branch's service (e.g. `postgres/db`) — binding **requires a branch** (defaults to the current branch when `--service` is given); omit `--service` for an unbound secret (as before) |
| `insta secrets unset <NAME> [--branch <b>]`       | Remove a user secret |
| `insta deploy <dir>` / `--image <url>` [`--branch <b>`] [`--group <g>`] [`--port <n>`] | deploy to a compute service — a **source dir** (needs a `Dockerfile`; built remotely on Fly, no local Docker) or a **prebuilt image**. Defaults to the branch's sole compute service; `--group` picks by name (gated: `deploy`) |
| `insta compute set-domain <host>` / `check-domain <host>` / `remove-domain <host>` [`--branch --group --json`] | attach / check / detach a **developer-owned custom domain** on a compute service — Fly issues the cert + routes; prints the DNS records to set in **your own** registrar (set/remove gated: `deploy`) |
| `insta compute start\|stop\|suspend [service]` · `insta compute status [service]` [`--json`] | control a compute service's lifecycle — **persistent override** of auto scale-to-zero: `stop`/`suspend` take it offline and traffic will **not** wake it until `start`; `status` shows desired vs. live state. All plans; ungated. `[service]` defaults to the project's sole compute service |
| `insta compute always-on on\|off [service]` [`--branch --json`] | idle-mode dial: `on` = machines never scale to zero (no cold starts; idle RAM bills at actual usage), `off` = default scale-to-zero (idle costs ~nothing, first request cold-starts). All plans; billing is actual usage either way |
| `insta db always-on on\|off` [`--branch --group --json`] | same dial for a postgres service: `off` (default) suspends the idle instance — first connection after idle cold-starts; `on` keeps it warm |
| `insta manifest` [`--json`] | agent-legible env view: each branch's db / storage / compute + URLs |
| `insta metrics <db\|compute>` [`group`] [`--branch --from --to --step --json`] | service metrics (compute=Fly; db=provider-limited) |
| `insta logs <db\|compute>` [`group`] [`--branch --limit --region --instance --deploy --json`] | logs (compute=Fly; db=provider-limited); `--deploy` shows compute **deploy events** (Fly machine lifecycle: created/started/…) instead of runtime logs (compute-only) |
| `insta usage` [`--from --to --json`] | usage aggregated by meter, with `costUsd` (snapshotted at collection) |
| `insta billing` [`--org <id>`] [`--json`] | current cycle summary: tier / included credit / used / overage / status |
| `insta billing upgrade <pro\|enterprise>` · `insta billing portal` [`--org`] [`--no-open`] [`--json`] | Stripe Checkout to subscribe / Customer Portal to manage (opens a browser; `--no-open` prints the URL) |
| `insta events` [`--branch <b>`] [`--limit <n>`] [`--json`] | audit + agent-event timeline |
| `insta policy get` [`--json`] · `insta policy set <action> <decision>` | view / set governance policy (actions include `service.add/remove/rename/scale/upgrade/setAccess`) |
| `insta approvals list` [`--status`] · `insta approvals approve <id>` [`--always`] · `insta approvals deny <id>` | manage gated actions |
| `insta observe install` · `report` [`--json`] · `sync` | local credential-audit hook (see below) |
| `insta upgrade` · `insta autoupdate [on\|off]` | self-update the CLI (binary re-runs the installer; npm uses `npm i -g`). Auto-update is **on by default** pre-1.0; `autoupdate off` / `INSTA_NO_AUTOUPDATE=1` disables. (CLI ≥ 0.0.5) |
| `insta setup agent` [`--mcp-token`] | one-step agent onboarding: installs the insta skill user-globally for every coding agent, then registers the **remote MCP server** — Claude Code via `claude mcp add` (user scope) plus a config-file entry for every other detected MCP-capable agent. Default = **OAuth**, no credential written (browser auth on first `/mcp` use); `--mcp-token` = headless fallback that mints a durable `insta_` token named `mcp-<hostname>` (needs `insta login`; Claude Code only). Idempotent; `INSTA_MCP_URL` / `INSTA_SKILLS_REPO` override the URL / skill source. The skill source, the MCP host and its registration name are all resolved from the current **environment**, so a staging machine installs `InsForge/insta-skills#devel` and registers `insta-cloud-staging` → `mcp.staging.instacloud.com` (both environments can coexist on one machine). See [mcp.md](references/mcp.md) and [Environments](#environments) |
| `insta mcp install` [`--agent <claude-code\|cursor\|codex\|opencode\|copilot\|factory-droid>`] [`--mcp-token`] | register the remote MCP server only (no skill install) — default: Claude Code + all detected agents; `--agent` targets one. Config merges never clobber existing entries; restart the tool afterwards |

`DATABASE_URL` + compute + storage (`AWS_*` / `BUCKET_NAME`) are **per-branch** (new projects: each
branch copy-on-write-forks its parent's bucket; a project created before snapshots keeps a **shared** bucket).
A project may have **multiple services of every type**, up to `INSTA_MAX_SERVICES_PER_TYPE` (default
5) per type. Minted credentials are named per service (service name upper-snaked): `DATABASE_URL_<NAME>`,
`BUCKET_NAME_<NAME>`, `AWS_ACCESS_KEY_ID_<NAME>`, etc. The **oldest** service of each type also gets
the plain unsuffixed names, so single-service projects are unaffected.

`insta secrets set <NAME>` / `unset <NAME>` manage **user-defined** secrets — reserved names
(`DATABASE_URL`, `AWS_*`, `BUCKET_NAME`, and any other live minted credential name) are rejected to
avoid clobbering platform-managed credentials. Gated: `secrets.write`. Changes apply on the next
`insta secrets` fetch or the next deploy — no hot reload. `--service` binding is **metadata only** —
it records which service a secret belongs to but never changes the secret's name/value or the
`.env` bundle. `secrets list`, `secrets tree`, and `services secrets` are all **names only**; secret
**values** still come exclusively from `insta secrets` → `.env`.

## Volumes

A **volume** is a service's persistent block disk — always a **service attribute**, never a
standalone resource (nothing to create or list separately; it lives and dies with its service):

- **postgres** has one by default — view/grow only: `insta db volume` [`--size <Gi>`].
- **compute** opts in at creation (`insta services add compute <name> --volume <gi>`) **or any
  time later** (`insta compute volume <name> --size <gi>` on a volumeless service attaches one;
  the disk mounts on the **next deploy**). Fixed mount path **`/data`** (survives deploys and
  restarts). Constraints: machine count stays **1** (scale to 1 before attaching), idle
  scale-to-zero uses **stop** (cold wake) instead of suspend, and a volume **never detaches** —
  but it **can be deleted** (`insta compute volume <name> --delete`): the disk and **all its
  data** are destroyed immediately (irreversible — download anything you need first), billing
  stops, and both constraints lift. View/grow/delete: `insta compute volume` [`--size <Gi>`]
  [`--delete`].
- **Free plans may attach at the default 1Gi**; viewing is every plan. Only **growth is paid** and
  plan-capped — don't pre-check the plan, just run the command: the backend's 403 carries the
  upgrade hint. **Grow-only** — a provisioned disk cannot shrink.
- Billing is **actual data stored**; the size is a cap, not a price.

## Environments

`prod` and `staging` are **separate deployments** — different regions, databases, and auth. A
session minted by one can never authenticate against the other, so `insta env use` drops the stored
session and you log in again. Default is `prod`; nothing changes unless you switch.

| | `prod` (default) | `staging` |
|---|---|---|
| control plane | `api.instacloud.com` (us-east-2) | `api.staging.instacloud.com` (us-west-1) |
| MCP server | `mcp.instacloud.com/mcp` | `mcp.staging.instacloud.com/mcp` |
| registers as | `insta-cloud` | `insta-cloud-staging` |
| agent skills | `InsForge/insta-skills` | `InsForge/insta-skills#devel` |
| CLI channel | latest stable release | newest prerelease (`v*-rc.N`), else stable |

Install one-liners — each installs a complete stack for its environment (CLI build, control plane,
MCP registration, and skill text all match):

```bash
curl -fsSL agents.instacloud.com | sh            # production
curl -fsSL agents.staging.instacloud.com | sh    # staging
```

> **`agents.staging.instacloud.com` is not live yet** (its DNS/CloudFront ships separately). Until
> it resolves, use the raw URL, which is exactly what the short host will serve:
>
> ```bash
> curl -fsSL https://raw.githubusercontent.com/InsForge/insta-cli/main/install.sh | sh -s -- --agents --staging -y
> ```
>
> If the environment cannot be applied (a CLI older than `insta env`), the installer **fails with a
> non-zero exit** rather than silently leaving you on production.

```bash
insta env                      # current environment + its hosts
insta env use staging          # switch; persisted to ~/.insta/config.json
insta login --env staging --oauth github
```

Control plane, MCP host **and** skill source are resolved **together** from one switch, so this
machine's CLI and its agents can never end up on different environments — including the case where
the CLI talks to staging while the agent reads prod's skill text.

Resolution order, most specific first:

1. `INSTA_API_URL` — a literal URL; the only way to reach a host no environment name covers
   (`insta-oss` on localhost, a preview deployment). `INSTA_MCP_URL` and `INSTA_SKILLS_REPO` do the
   same for the MCP host and the skill source.
2. `INSTA_ENV` — `prod` | `staging`. An unrecognised value is an **error**, never a silent fallback
   to prod.
3. the persisted `apiUrl` in `~/.insta/config.json`.
4. `prod`.

Prereleases never take the `latest` GitHub release or npm's `latest` dist-tag (they ship
`--prerelease` and under npm `next`), so a staging CLI build can't reach production installers.

**Agents:** don't switch environments as a debugging step. If a command fails, check `insta env`
first — targeting staging when the user meant prod (or vice versa) produces confusing "project not
found" errors, because the two have entirely separate project lists.

## Deploy

```
insta deploy ./app --port <n>              # build ./app (needs a Dockerfile) remotely on Fly, push, deploy
insta deploy --image <url> --port <n>      # or deploy a pre-built / already-pushed image
# targets the CURRENT branch's sole compute service (or --group <name> when there are several);
# --branch targets another branch; the URL prints on success.
# Source mode needs the `fly` CLI (auto-installed via Homebrew on macOS) but NO Fly login — the
# platform mints a short-lived, app-scoped deploy token for the build.
```

`--port` must match the port the image actually listens on (`ENV PORT` / `EXPOSE` / server bind) — a
mismatch boots fine but every request fails with `instance refused connection on 0.0.0.0:<port>`.
Secrets are **injected at deploy** as env vars (decrypted from the branch). Read creds from
`process.env` in production; **never bake `./.env` into the image**. A compute service serves one app
on one port at `https://<app>.fly.dev`.

**Custom domain (bring your own):** `insta compute set-domain app.example.com` attaches your own
domain to a branch's compute service and prints the DNS records to add **at your registrar** (a
`CNAME → <app>.fly.dev` for a subdomain, `A`/`AAAA` for an apex, plus a validation `CNAME`). The cert
(Let's Encrypt) + routing are handled for you; `insta compute check-domain <host>` shows status once
DNS propagates. The domain's DNS lives in your zone — you set it, not InstaCloud.

> Multiple services of every type (postgres/storage/compute, up to 5 each), `insta services
> scale`/`upgrade`, and **source-directory deploy** (`insta deploy <dir>` → Fly remote builder, no
> local Docker) are all implemented.

## Dockerfile templates

Moved to [references/deploy.md](references/deploy.md) (backend / full-stack / SPA patterns).

## Govern & observe

- **Policy** gates `secrets.read`, `secrets.write`, `deploy`, `branch.delete`, `project.delete`, and
  `service.add` / `service.remove` / `service.rename` / `service.scale` / `service.upgrade` /
  `service.setAccess`. `approve` = require a
  human: the action returns `approval_required`; an admin runs `insta approvals approve <id>`, then
  you **re-run** it (single-use grant). `project.delete` is gated by default. `--always` on approve
  flips the policy to `allow`.
- `insta approvals list` / `approve <id>` / `deny <id>` — manage pending gates.
- `insta events [--branch] [--limit]` — timeline of resource side-effects (project/branch creates,
  deploys + URLs, govern decisions) plus ingested agent events.
- **`insta observe`** — the local credential-audit hook (a PostToolUse hook for Claude Code / Codex,
  auto-installed on `project create`/`link`). It scans each tool-use for credential exposure
  (AWS / GitHub / Stripe / LLM / DB / JWT / private keys) and appends **redacted fingerprints** (never
  raw secrets) to `./.insta/audit.jsonl`. `insta observe report` renders it; `insta observe sync`
  uploads findings into the project timeline (idempotent).
