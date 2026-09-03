# InstaCloud CLI reference

Command catalog with flags and gates. Task guidance lives in `references/` — setup, deploy,
branching, governance, operate, mcp.

## Commands

| Command | Purpose |
|---|---|
| `insta login` [`--api-url <url>`] [`--env <prod\|staging>`] · `insta login --email <e> --password <p>` · `insta login --oauth <github\|google>` · `insta login --device` · `insta login --api-key <insta_…>` · `insta logout` | auth (api-url + tokens persist; tokens auto-refresh). **Bare `insta login` signs in from the browser** (any account type — email, GitHub, Google): it opens the console approval link locally, prints it as fallback, and polls until the human approves (~15 min window). `--device` is the same grant, print-only, for a machine with no usable browser (VMs, SSH, CI): the human approves **from a browser on any other machine**. `--oauth` opens a browser straight into the named provider (loopback capture). Agents use email/password or an API token; a password (`--password`/`$INSTA_PASSWORD`) needs `--email`. An `insta_` token can also come from ID-JAG registration at `POST https://api.instacloud.com/agent/auth` when the agent runs inside a participating provider (preview, see https://instacloud.com/auth.md), then `insta login --api-key insta_…`. `--env` targets a named deployment (see [Environments](#environments)); `--api-url` wins if both are given |
| `insta env` [`--json`] · `insta env use <prod\|staging>` [`--json`] | show or switch the deployment environment. **Switching drops the stored session** — prod and staging are separate deployments, so the old token cannot authenticate. See [Environments](#environments) |
| `insta status` [`--json`] | environment + login + linked project + current branch |
| `insta org list` [`--json`] · `insta org create <name>` [`--json`] | organizations (**one free org per user** — upgrade an existing org before creating another) |
| `insta project create <name>` [`--org <id>`] [`--json`] | create an **empty** project (no services), link this dir. With `--json`, a missing/unresolvable name is a hard error even on a terminal (never prompts) |
| `insta project list` [`--org`] [`--json`] · `insta project link <id>` [`--json`] | list / link existing |
| `insta project delete` [`--json`] | tear down ALL resources + unlink (gated: `project.delete`, approval by default) |
| `insta services add <postgres\|storage\|compute\|redis\|mysql\|mongodb> <name>` [`--branch <b>`] [`--region <r>`] [`--image <url>`] [`--port <n>`] [`--always-on`] [`--volume <gi>`] [`--json`] | provision a service **on a branch** (default: current/linked branch) — services are **branch-scoped**: adding one on a branch does not add it to any other branch; postgres/compute get a default access domain (gated: `service.add`). **compute only:** `--image` runs that image immediately at creation (otherwise compute starts as an empty, unreachable app until `insta deploy`); `--port` sets the listening port (default `8080`); `--always-on` creates it pinned-warm (never scales to zero — all plans, billing is actual usage either way); `--volume <gi>` attaches a persistent `/data` volume (also attachable later via `insta compute volume --size` — see [Volumes](#volumes)). The image is **persisted** on the service — shown in `services list`, re-run when the branch is forked, and updated later via `insta deploy --image`. Both positional arguments are optional in the parser: a **terminal** given **no type** picks from the same four kinds the dashboard's Add Service menu offers — **Docker Image** (asks for the ref, suggests a name from it, then the port), **Postgres** (`main-db`), **Storage** (`assets`), **Empty Service** (`compute`) — and is asked for a name after; given a **type but no name**, only the name is asked, for that type's plain kind. **Anything without a TTY errors and creates nothing** — with no type, all four kinds, each printed as the command to run instead (Docker Image as `insta services add compute <name> --image <ref> --port <n>`); with a type but no name, that kind's naming usage — so always pass both here. Docker Image is a *kind*, not a follow-up question: non-interactively it is just `--image` on a compute service. `--json` prints the created service object (id, type, name, domain, region, …) instead of the human line, and opts out of the prompts, so a missing positional errors the same way even on a terminal |
| `insta regions` [`--json`] | list regions available for postgres/compute services |
| `insta services list` [`--json`] [`--branch <b>`] · `insta services rename <type> <name> <new-name>` [`--json`] [`--branch <b>`] · `insta services remove <type> <name>` [`--branch <b>`] [`--json`] | list / rename / remove a branch's services (default: current branch; bindings keep pointing at renamed services; gated: `service.rename` / `service.remove`). `list --json` rows carry `pg_version` on postgres services — the Postgres **major** the instance runs (e.g. `16`), known without waking it — so pick `pg_dump`/`pg_restore`/`psql` of the **same major** before you connect (a newer client's dump emits statements the server cannot restore); the human line shows it as `pg 16`. `null` = a legacy row that never recorded one — read `insta db stats` (exact `server_version`, running instance only) instead |
| `insta services secrets <type> <name>` [`--branch <b>`] [`--json`] | secret **names** bound to one service (e.g. `insta services secrets postgres db`) — default: current branch |
| `insta db url` [`--branch <b>`] [`--group <g>`] [`--json`] | print a postgres service's **connection string** (DSN). Default output is the bare URL alone on stdout, pipe-friendly (`psql "$(insta db url)"`); `--json` replaces it with a `{service, branch, url}` envelope. This is **the** command that yields the DSN: provider credentials are not in `insta secrets` (gated: `secrets.read`). `--group` picks one when the branch has several postgres services |
| `insta db connect` [`--branch <b>`] [`--group <g>`] | open an **interactive psql session** on a postgres service (needs `psql` on PATH; gated: `secrets.read`). A suspended instance wakes on connect — the first prompt can take a few seconds. Exits with psql's own exit code |
| `insta db stats` [`--branch <b>`] [`--group <g>`] [`--json`] | point-in-time **postgres stats snapshot**: connections vs the server's max (active count), cache hit rate, database size. Read-only; insta-db-backed services answer from the control plane without waking a suspended instance. Same read as the `insta_db_stats` MCP tool's `metrics` kind (its `insight`/`activity`/`query-stats` kinds are MCP-only) |
| `insta services scale compute <name> <number>` [`region`] | set the compute service's same-region replica count (**1–10**) — **paid plans only** (free → 403); gated: `service.scale`. `region` is an InstaCloud slug (e.g. `us-east`; see `insta regions`), **not** a raw Fly code |
| `insta compute limits [service]` [`--memory <size>`] [`--cpu <n>`] [`--branch <b>`] [`--json`] | show or set a compute service's **resource ceiling** — **paid plans**. Bare = read (prints the ceiling **and** the plan max). Setting **requires `--memory`** (`512mb`, `1gb` — decimal `mb/gb` and binary `Mi/Gi` suffixes both accepted); cpu derives from it, and `--cpu` is only an optional override for parallel workloads (never valid alone). Moves **both directions**: billing is actual usage, so the ceiling caps what the app may burn — it is not a price |
| `insta db limits` [`--cpu <n>`] [`--memory <size>`] [`--branch <b>`] [`--group <g>`] [`--json`] | same ceiling control for a postgres service — **paid plans**; both directions. Takes provider quantities (`--cpu 2` or `2500m`; `--memory 4Gi`); either flag alone works. Bare = read the current ceiling |
| `insta compute volume [service]` [`--size <Gi>`] [`--delete`] [`--branch <b>`] [`--json`] | show, **attach**, grow, or **delete** a compute service's persistent `/data` volume. Bare = read (size, mount path, plan cap — any plan). `--size` on a volumeless service **attaches** one (any plan at the default 1Gi; mounts on the next deploy); on a volume-bearing one it grows — **paid plans, grow-only** (a provisioned disk cannot shrink). `--delete` **destroys the disk and ALL its data immediately** (any plan; irreversible; no detach exists — billing stops now, and suspend fast-wake + scale-out return; gated `service.remove`). See [Volumes](#volumes) |
| `insta db volume` [`--size <Gi>`] [`--branch <b>`] [`--group <g>`] [`--json`] | show or grow a postgres service's provisioned volume (block disk). Bare = read (size + plan cap — any plan); `--size` grows it — **paid plans, grow-only**. Postgres has its volume by default — there is nothing to attach |
| `insta storage list` [`--prefix <p>`] [`--cursor <c>`] [`--limit <n>`] [`--service <name>`] [`--branch <b>`] [`--json`] | list the objects in a storage service's bucket — one `size  modified  key` row each, in key order (S3 lists lexicographically). **Prefix filter only**: S3 has no substring search, so `--prefix` is applied **server-side** and there is nothing to match mid-key. Pagination is by cursor — the `nextCursor` a page prints is what `--cursor` takes; `--limit` is 1..1000 (default 100). `--service` picks one when the branch has several storage services (gated: `storage.read`) |
| `insta storage get <key>` [`-o <file>`] [`--service <name>`] [`--branch <b>`] [`--json`] | download one object. The platform returns a **short-lived presigned URL** (~60s) and the bytes come **straight from the provider** — nothing streams through the control plane. Writes to the key's **last segment** by default (never a path, so no key can write outside cwd); `-o` names the file. `--json` prints `{url, expiresAt}` and downloads nothing (gated: `storage.read`) |
| `insta storage delete <key>` [`--service <name>`] [`--branch <b>`] [`--json`] | delete one object — **irreversible**, and it runs immediately with no prompt (like every other destructive command here, the governance gate is the guard; a data operation stages nothing for a deploy). Deleting a key that is already gone still succeeds, so success is no proof it existed (gated: `storage.delete`, `allow` by default) |
| `insta services upgrade <compute\|postgres> <name> <spec>` | **legacy** (pre-usage-billing): raise a named spec, up-only — **paid plans only**; gated: `service.upgrade`. Prefer `insta compute limits` (and, for postgres, `insta db limits`), which also lower. **Compute-only in practice**: a postgres `upgrade` is rejected outright (400) — resize a postgres service with `insta db limits` (cpu/memory) and grow its disk with `insta db volume` |
| `insta branch create <name>` [`--from <parent>`] [`--json`] | isolated env: **forks the parent branch's current services** — a CoW database branch per postgres, a CoW-forked bucket per storage (snapshot-enabled projects), a clone of every compute service (re-running the parent's persisted image, if any) — then the two branches' service catalogs diverge independently (services are **branch-owned, not project-wide**). **≤10 branches/project.** Does NOT switch |
| `insta branch switch <name>` [`--json`] · `insta branch list` [`--json`] | set current branch / list |
| `insta branch merge <source>` [`--into <target>`] [`--json`] | **structural** merge: creates on the target branch (default: current) every service present on `<source>` but missing there — fresh & **empty, no data copied**. Services the target already has are skipped (reason: `exists`\|`cap`\|`secret-collision`). Additive only — never deletes target services; idempotent |
| `insta branch delete <name>` [`--json`] | tear down the branch's resources (gated: `branch.delete`) |
| `insta secrets` [`--branch <name>`] [`-o <file>`] [`--print`] [`--json`] | secret seam → write user-defined project/branch secrets to `./.env`; provider-minted service credentials are **not** exported here (gated: `secrets.read`) |
| `insta secrets list` [`--branch <b>`] [`--json`] | secret names for the branch, **grouped by service** — each service's bound secrets, plus a branch-level "unbound" group and a project-wide group |
| `insta secrets tree` [`--json`] | the whole project as `project → branch → service → secrets` (names only) |
| `insta secrets set <NAME> [value] [--branch <b>] [--service <compute/name>] [--json]` | Set a user secret (project-wide by default; value from stdin if omitted). `--service` scopes it to that branch's compute service (e.g. `compute/api`) — binding **requires a branch** (defaults to the current branch when `--service` is given); omit `--service` for an unbound secret (as before) |
| `insta secrets unset <NAME> [--branch <b>] [--json]` | Remove a user secret |
| `insta secrets sources` [`--branch <b>`] [`--json`] | List provider credential sources available for explicit compute binding, e.g. `postgres/db: DATABASE_URL` or `redis/cache: REDIS_URL, ...` (names only; gated: `secrets.read`) |
| `insta secrets bind <ENV_NAME> <source>` [`--source-name <name>`] `--to <compute/name>` [`--branch <b>`] [`--json`] | Bind one provider credential from `<source>` (`postgres/db`, `redis/cache`, `mysql/orders`, `mongodb/catalog`, `storage/assets`, …) into a compute service's runtime env var. `--source-name` is required when the source exposes multiple credential names. Takes effect on the next deploy — or immediately on a running service with `insta compute restart` (CLI ≥ 0.0.51) (gated: `secrets.write`) |
| `insta secrets bindings --target <compute/name>` [`--branch <b>`] [`--json`] | List provider credential bindings for one compute service (names only; gated: `secrets.read`) |
| `insta secrets unbind <ENV_NAME> --from <compute/name>` [`--branch <b>`] [`--json`] | Remove one provider credential binding from a compute service; takes effect on the next deploy — or immediately on a running service with `insta compute restart` (CLI ≥ 0.0.51) (gated: `secrets.write`) |
| `insta build [dir]` [`--explain`] [`--port <n>`] [`--json`] | **verify before you deploy** — local, offline, deploys nothing, needs no login: prints the detection plan (builder, install/build/start commands, port **with the reason it was chosen**, `.env.example` keys), the Dockerfile (yours, or — **if nixpacks is installed**, never auto-installed — the one nixpacks would generate; `--explain` includes its content), and static checks each with a next action (missing Dockerfile/start command, port mismatch, `node_modules` shipping in the build context). **Verdict semantics (CLI ≥ 0.0.48): only a Dockerfile IN the directory can make a dir `deployable`.** A dir with no Dockerfile where nixpacks detects the app gets `builder: nixpacks` but its Dockerfile check is a ⚠ warning and the verdict stops at `needs-attention` (exit 0) — because `insta deploy <dir>` builds the directory's own Dockerfile and refuses without one; the nixpacks lane is server-side, for GitHub-connected repos only. The nixpacks Dockerfile shown by `--explain` is **for inspection, not standalone** (it `COPY`s `.nixpacks/` support files the dir does not have) — do NOT save it as `Dockerfile`; use the detected install/start commands as the starting point for your own. Verdict `failed` (exit 1) = no Dockerfile and nixpacks missing/undetected, or no start command. Run it before `insta deploy <dir>` instead of finding out from a burned remote build |
| `insta deploy <dir>` / `--image <url>` [`--branch <b>`] [`--group <g>`] [`--port <n>`] [`--json`] | deploy to a compute service — a **source dir** (**requires a `Dockerfile` in the dir — there is no no-Dockerfile/nixpacks lane on this path**; without one it exits 1 naming the options: write a Dockerfile, `--image <url>`, or connect the repo on GitHub, whose server-side lane builds Dockerfile-less repos with nixpacks. On InstaCloud it builds the Dockerfile remotely on Fly — no local Docker; against a local insta-oss daemon the CLI builds with your local docker instead, same command) or a **prebuilt image**. Defaults to the branch's sole compute service; `--group` picks by name (gated: `deploy`). `--json` prints one `{image, machineId, url, branch, group, nextActions}` document on stdout — build progress moves to stderr so stdout stays parseable |
| `insta template list` [`--json`] · `insta template info <code>` [`--json`] | browse the platform **template registry**: one row per template (code / version / category / required-var count / deploy count / name — tagline), and the detail view — version, maintainer, source, upstream pin, a services summary (types, ports, volumes), and every required/optional variable with its description, generator or default |
| `insta template deploy <code\|./dir>` [`--branch <b>`] [`--set <NAME=value>`] [`-y`, `--yes`] [`--json`] | deploy a template's whole service set onto a branch (default: current) — a **registry code**, or a **local directory** carrying `insta.template.yaml`. A bare word is **always** a registry code; local mode needs a path-looking target (`./dir`, `/abs/dir`, `~/dir`, `sub/dir`), so a same-named directory in the working dir can never shadow a registry template. Missing required variables are prompted for on a terminal; `--yes` or no TTY fails with the exact `--set NAME=value` list instead. `secret:N`-generated and defaulted variables are resolved **by the platform** — generated secrets never transit. Renders the 4-step pipeline (create services → write variables → deploy → health check), then the per-service URLs; `--json` replaces all of that with one document. May come back `approval_required` (hint on stderr, envelope on stdout with `--json`, **exit 2** — as every gated command). See [Templates](#templates) |
| `insta compute set-domain <host>` / `check-domain <host>` / `remove-domain <host>` [`--branch --group --json`] | attach / check / detach a **developer-owned custom domain** on a compute service — Fly issues the cert + routes; prints the DNS records to set in **your own** registrar (set/remove gated: `deploy`) |
| `insta compute start\|stop\|suspend [service]` · `insta compute status [service]` [`--json`] | control a compute service's lifecycle — **persistent override** of auto scale-to-zero: `stop`/`suspend` take it offline and traffic will **not** wake it until `start`; `status` shows desired vs. live state. All plans; ungated. `[service]` defaults to the project's sole compute service |
| `insta compute restart [service]` [`--branch <b>`] [`--json`] | **(CLI ≥ 0.0.51)** **re-run the image reference the service already runs**, against a freshly resolved env bundle — it asks for no new version and no new spec, though it does **not pin a digest**: a service recorded against a moving tag (`app:latest`) gets whatever that tag resolves to now (source deploys record a unique label and are unaffected). The two reasons to use it: a secret/binding changed and the running app hasn't picked it up (env is baked into the machine at deploy time), or the machine is up but **wedged** (`start` no-ops on a machine that is already `started`). The service must be **running**: a deliberately stopped/suspended one 400s and points at `insta compute start`. A never-deployed service 400s like `exec` does. **A running machine is health-gated coming back up** — if the app doesn't answer on its port the machines are rolled back (best-effort) to the config they were serving and the failure is reported (that verdict means the app is broken, not the platform). Whether an **idle machine (the default) is woken and gated at all depends on the compute plane** (`insta manifest --json` names it per compute row — `fly`, `microvm`, or a neutral `compute` when the platform reported none) — the Fly-backed one hands it the new config without waking, so the command returns fast, bills no uptime, and proves nothing about whether the app boots. Send it a request if you need that proof; see [operate.md](references/operate.md). All plans; **gated: `deploy`** — it lands configuration the way a deploy does, so a policy denying deploys denies this too; `start`/`stop` stay ungated and cycle a wedged machine without one. Refused while the org is billing-suspended; any machine it wakes bills as ordinary uptime, an idle one it leaves asleep costs nothing. WebSocket concurrency **is** re-asserted (it is recorded on the service), so a socket app does not need a redeploy to stay one |
| `insta compute exec [service] -- <command> [args...]` [`--branch <b>`] [`--timeout <sec>`] [`--json`] | run a **one-shot** command on the service's live machine — no interactive shell, no stdin. Wakes a scaled-to-zero machine first (the wake counts as billed uptime). `--timeout` bounds the run, **1–180s** (default 30). The CLI's **exit code is the remote command's exit code** — safe for scripts/agents to branch on. stdout/stderr stream to their own local streams verbatim, each capped at **1 MiB** (truncation noted on stderr); `--json` returns the raw response instead of split streams. Gated on **both** `deploy` and `secrets.read` — a deny on either is a 403. A service with no image ever deployed 400s: "this service has no machines yet — deploy an image first, then retry" |
| `insta compute always-on on\|off [service]` [`--branch --json`] | idle-mode dial: `on` = machines never scale to zero (no cold starts; idle RAM bills at actual usage), `off` = default scale-to-zero (idle costs ~nothing, first request cold-starts). All plans; billing is actual usage either way |
| `insta db always-on on\|off` [`--branch --group --json`] | same dial for a postgres service: `off` (default) suspends the idle instance — first connection after idle cold-starts; `on` keeps it warm |
| `insta manifest` [`--json`] | agent-legible env view: each branch's db / storage / compute + URLs |
| `insta metrics <db\|compute\|redis\|mysql\|mongodb>` [`group`] [`--branch --from --to --step --json`] | service metrics (compute + managed DBs=Fly, full; db=provider-limited) |
| `insta logs <db\|compute\|redis\|mysql\|mongodb>` [`group`] [`--branch --limit --region --instance --deploy --json`] [`--from <t>`] [`--to <t>`] [`--since <dur>`] | logs (compute + managed DBs=Fly; db=provider-limited). **Without a window, one recent provider page (~100 lines) comes back regardless of `--limit`** (the provider cursor only pages forward) — the answer says so in a `note` when cut. `--from`/`--to` (unix seconds or ISO-8601) or `--since` (`90s`/`30m`/`2h`/`1d`) page a time window (~7-day retention, up to 5000 lines). `--deploy` shows **deploy events** (Fly machine lifecycle: created/started/…) instead of runtime logs — any Fly-backed target, not db; window flags don't apply to it |
| `insta usage` [`--from --to --json`] | usage aggregated by meter, with `costUsd` (snapshotted at collection) |
| `insta billing` [`--org <id>`] [`--json`] | current cycle summary: tier / included credit / used / overage / status |
| `insta billing upgrade <pro\|team>` · `insta billing portal` [`--org`] [`--no-open`] [`--json`] | Stripe Checkout to subscribe / Customer Portal to manage (opens a browser; `--no-open` prints the URL) |
| `insta events` [`--branch <b>`] [`--limit <n>`] [`--json`] | audit + agent-event timeline |
| `insta policy get` [`--json`] · `insta policy set <action> <decision>` [`--json`] | view / set governance policy (actions include `service.add/remove/rename/scale/upgrade/setAccess` and `storage.read` / `storage.delete`) |
| `insta approvals list` [`--status`] [`--json`] · `insta approvals approve <id>` [`--always`] [`--json`] · `insta approvals deny <id>` [`--json`] | manage gated actions |
| `insta observe install` · `report` [`--json`] · `sync` | local credential-audit hook (see below) |
| `insta feedback --type <bug\|feature-request\|friction\|other> --component <cli\|mcp\|platform\|skills\|docs\|other> --title <t> --detail <d>` [`--file <path>`] [`--area <a>`] [`--command <c>`] [`--error <e>`] [`--expected <x>`] [`--workaround <w>`] [`--doc <ref>`] [`--severity <blocker\|major\|minor>`] [`--json`] | report an **InstaCloud-side** hurdle to the team (see [Feedback](#feedback)) — never for the user's own app. Works logged-out/unlinked/oss; ungated. Non-TTY with missing flags errors (never prompts); transport failures **exit 0** — continue the task, don't retry |
| `insta upgrade` · `insta autoupdate [on\|off]` | self-update the CLI (binary re-runs the installer; npm uses `npm i -g`). Auto-update is **on by default** pre-1.0; `autoupdate off` / `INSTA_NO_AUTOUPDATE=1` disables. (CLI ≥ 0.0.5) |
| `insta setup agent` [`--env <prod\|staging>`] [`--mcp-token`] [`--project <id>`] [`--create [name]`] | one-step agent onboarding: **self-installs the CLI globally first** when running from the npx cache with no durable `insta` on PATH (`npm i -g insta@<running version>`; best-effort — a failed install prints the manual fallback and setup continues; CLI ≥ 0.0.37), making `npx -y insta@latest setup agent` a complete cross-platform one-liner (bash / zsh / PowerShell / cmd); then installs the insta skill user-globally for every coding agent, then registers the **remote MCP server** — Claude Code via `claude mcp add` (user scope) plus a config-file entry for every other detected MCP-capable agent. Default = **OAuth**, no credential written (browser auth on first `/mcp` use); `--mcp-token` = headless fallback that mints a durable `insta_` token named `mcp-<hostname>` (needs `insta login`; Claude Code only). Idempotent; `INSTA_MCP_URL` / `INSTA_SKILLS_REPO` override the URL / skill source. **Environment (CLI ≥ 0.0.38): bare `setup agent` always targets prod** — a machine previously switched to staging is switched back (persisted via the `env use` path, foreign session dropped, announced). Staging is the explicit form `--env staging` (also persists the switch); `$INSTA_ENV` counts as explicit; a deliberate custom host (`$INSTA_API_URL`, or a persisted custom apiUrl) is left alone unless `--env` is given. The skill source, the MCP host and its registration name all resolve from that one environment, so a staging setup installs `InsForge/insta-skills#devel` and registers `insta-cloud-staging` → `mcp.staging.instacloud.com` (both environments can coexist on one machine). Ends with a status-aware `next:` hint (login / project create / the prompt.md fetch prompt). **`--project <id>` (CLI ≥ 0.0.48) also links the working directory to that project** after setup — same code path as `insta project link <id>` (writes `./.insta/project.json`, installs per-project stack skills + the observe hook), flowing through the interactive login offer first when there's no session. With no session available (declined login, `-y`, non-TTY) the link is skipped with the manual `insta login` + `insta project link` hint (exit 0); a failed link (bad id / no access) exits 1. This is the flag behind the console Connect panel's single-line CLI setup. **`--create [name]` (CLI ≥ 0.0.52) instead creates a NEW project** and links it — same code path as `insta project create`, with the same login flow, skip-with-hint and exit-1-on-failure behaviour as `--project`; the name is optional and resolves exactly as that command's does — a directory with no usable name (`~`, `~/projects`, `/tmp`) gets that command's guidance and no project, not an error. `--create` and `--project` are mutually exclusive, rejected before anything is installed. See [mcp.md](references/mcp.md) and [Environments](#environments) |
| `insta mcp install` [`--agent <claude-code\|cursor\|codex\|opencode\|copilot\|factory-droid>`] [`--mcp-token`] | register the remote MCP server only (no skill install) — default: Claude Code + all detected agents; `--agent` targets one. Config merges never clobber existing entries; restart the tool afterwards |

**`--json` contract (CLI ≥ 0.0.37):** every mutating command an agent chains from takes `--json` —
one JSON document on stdout, progress/diagnostics on stderr, so `$(insta … --json)` always parses.
Deliberate exceptions: `insta run` has no `--json` (its stdout belongs to the child command — its
"injected secrets" banner is on stderr); `insta setup agent` and `insta upgrade` don't have it yet
(their stdout is a live installer stream).

**Approval gate exit code (CLI ≥ 0.0.37):** when a gated command returns `approval_required`, the
hint prints to **stderr** and the CLI **exits 2** — distinct from 1 (error), so scripts/agents can
branch on "approvable: have an admin `insta approvals approve <id>`, then re-run". Previously this
printed to stdout and exited 0, which read as success in pipelines.

Provider-minted credentials are **per-branch and per-service**. They live under the service that
created them with canonical names (`DATABASE_URL`, `REDIS_URL`, `MYSQL_URL`, `MONGODB_URL`,
`AWS_ACCESS_KEY_ID`, `BUCKET_NAME`, …), and **do not automatically appear** in `insta secrets`,
`insta run`, or a compute deployment. Decide what each compute service should receive with explicit
bindings:

```bash
insta secrets sources
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta secrets bind REDIS_URL redis/cache --source-name REDIS_URL --to compute/app
insta secrets bindings --target compute/app
insta deploy . --group app --port 8080
```

If a source exposes exactly one credential (`postgres` → `DATABASE_URL`), `--source-name` is
optional. If it exposes several (`storage`, `redis`, `mysql`, `mongodb`), pass the source credential
name to bind. Binding overwrites the target env var's previous binding; an env name that collides
with a user secret visible to the same compute service is rejected (409). Binding itself does not
expose plaintext — the one CLI read that does is `insta db url` / `insta db connect` (the postgres
DSN, gated `secrets.read`); every other credential value only runs where it is bound (the deployed
app, or `insta compute exec`).
Changes apply on the next deploy — **or on `insta compute restart`** (CLI ≥ 0.0.51), which re-runs
the image reference the service already runs against a freshly resolved bundle. There is still no hot reload:
either way the machine takes a new config and restarts on it, in place (the machine id survives). An
idle machine may take the config without waking — see the `insta compute restart` row and
[operate.md](references/operate.md). A project may have **multiple services of every type**, up to
`INSTA_MAX_SERVICES_PER_TYPE` (default 5) per type.

`insta secrets set <NAME>` / `unset <NAME>` manage **user-defined** secrets. A user secret cannot
collide with a provider credential binding visible to the same compute service. Gated:
`secrets.write`. Changes apply on the next
`insta secrets` fetch, the next deploy, or an `insta compute restart` (CLI ≥ 0.0.51) — no hot reload either way. `--service` on `secrets set` scopes a
user-defined secret to a branch compute service; it is separate from provider credential binding
(`secrets bind`). `secrets list`, `secrets tree`, `services secrets`, `secrets sources`, and
`secrets bindings` are all **names only**.

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

Install one-liners — each installs a complete stack for its environment (control plane, MCP
registration, and skill text all match). The npx form works on **every OS and shell** (Node 18+;
CLI ≥ 0.0.37 self-installs globally); the curl form is the no-Node native-binary path for
macOS/Linux only — **never run it on native Windows**, where PowerShell's `curl` alias and the WSL
`bash` shim break it:

```bash
npx -y insta@latest setup agent                         # production (any OS — ALWAYS prod, CLI >= 0.0.38)
npx -y insta@latest setup agent --env staging           # staging (any OS; persists the env switch)
npx -y insta@latest setup agent --project <id>          # + link this directory to a project (CLI >= 0.0.48; the console Connect panel's one-liner)
npx -y insta@latest setup agent --create [name]         # + create a NEW project (CLI >= 0.0.52; name defaults to this directory)
curl -fsSL agents.instacloud.com | sh            # production (macOS/Linux, no Node needed)
curl -fsSL agents.staging.instacloud.com | sh    # staging
```

Staging via npx is the `--env staging` one-liner above (CLI ≥ 0.0.38) — it persists the env switch
itself, so no separate `insta env use staging` is needed (and a bare `setup agent` afterwards would
switch the machine back to prod, by design). The npm route always installs the **stable** CLI
build; staging's prerelease channel is the curl installer's concern (npm's `next` tag can lag
behind `latest`, so `insta@next` is only for deliberately testing a prerelease newer than stable).

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
insta login --env staging      # or --email/--oauth as usual
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
insta build ./app --port <n>               # first: verify — `deployable` only with a Dockerfile IN ./app
insta deploy ./app --port <n>              # build ./app's Dockerfile remotely on Fly, push, deploy
#   no Dockerfile → exits 1 (no nixpacks lane here); write one, use --image, or connect the repo on GitHub
insta deploy --image <url> --port <n>      # or deploy a pre-built / already-pushed image
# targets the CURRENT branch's sole compute service (or --group <name> when there are several);
# --branch targets another branch; the URL prints on success.
# Source mode needs the `fly` CLI (auto-installed via Homebrew on macOS) but NO Fly login — the
# platform mints a short-lived, app-scoped deploy token for the build.
```

`--port` must match the port the image actually listens on (`ENV PORT` / `EXPOSE` / server bind) — a
mismatch boots fine but every request fails with `instance refused connection on 0.0.0.0:<port>`.
At deploy, compute receives `PORT`, user-defined secrets visible to that compute service, and
provider credentials you explicitly bound with `insta secrets bind`. It does **not** receive every
platform credential by default. Read env vars from `process.env` in production; **never bake
`./.env` into the image**. A compute service serves one app on one port at `https://<app>.fly.dev`.

**Custom domain (bring your own):** `insta compute set-domain app.example.com` attaches your own
domain to a branch's compute service and prints the DNS records to add **at your registrar** (a
`CNAME → <app>.fly.dev` for a subdomain, `A`/`AAAA` for an apex, plus a validation `CNAME`). The cert
(Let's Encrypt) + routing are handled for you; `insta compute check-domain <host>` shows status once
DNS propagates. The domain's DNS lives in your zone — you set it, not InstaCloud.

> Multiple services of every type (postgres/storage/compute, up to 5 each), `insta services
> scale`/`upgrade`, and **source-directory deploy** (`insta deploy <dir>` → Fly remote builder of the dir's own Dockerfile, no
> local Docker) are all implemented.

## Dockerfile templates

Moved to [references/deploy.md](references/deploy.md) (backend / full-stack / SPA patterns).

## Templates

A **template** is a whole service set (images, ports, volumes, env) published as one unit —
`insta template deploy <code>` creates those services on a branch, writes their variables, deploys
and health-checks them, instead of a hand-rolled `services add` + `secrets set` + `deploy` sequence.

- **Two modes, chosen by the target.** A bare word (`plausible`) is **always** a registry code. A
  path-looking target (`./plausible`, `/srv/tpl`, `~/tpl`, `sub/dir`) is **always** a local directory and must
  contain `insta.template.yaml` (validated locally, then sent inline). So a local directory never
  shadows a registry template — `./` is how you opt into the local one.
- **Variables.** `--set NAME=value` (repeatable) answers them up front. Required variables with no
  answer are prompted for on a terminal; with `--yes` or no TTY the command fails listing exactly
  what to pass. Variables carrying a `secret:N` generator or a default are resolved **by the
  platform** — generated secrets never leave it.
- **Outcomes.** `succeeded` prints each service's URL — then run `insta secrets` to refresh `./.env`.
  `partial` is **terminal**: the healthy services stay up and the created resources are kept, so read
  the log tail, then re-run the deploy to retry or `insta services remove <type> <name>` to clean up.

## Feedback

`insta feedback` reports a hurdle in the **InstaCloud toolkit itself** to the InstaCloud team.
File it when InstaCloud got in *your* way, then **continue the user's task with a workaround** —
never block on the report, and never file feedback for problems in the app the user is building.

```bash
insta feedback --json --type bug --component cli --area deploy \
  --title "deploy --branch deploys to main" \
  --detail "insta deploy --branch feat accepted the flag but the release landed on main" \
  --command "insta deploy . --branch feat --port 8080" \
  --error "deployed ... (branch main)" \
  --expected "release lands on branch feat, per this reference" \
  --doc "skills/insta/cli-reference.md" --workaround "insta branch switch feat, then deploy"
```

Situation → `--type`:

- **"This should work (per docs / the stated contract), but doesn't"** → `bug`.
- **"I was instructed to do X, but reality required Y"** → also `bug`, **with `--doc` +
  `--expected` + `--workaround`** — you can't know whether the instructions are stale or the
  product regressed, and those three fields let the team disambiguate.
- **"What I need is not supported"** → `feature-request`.
- **"Works, but confusing or awkward"** → `friction`.

`--component` is which piece of the toolkit (`cli|mcp|platform|skills|docs|other`); `--area` is
the product domain (deploy, branch, secrets, db, storage, compute, governance, billing, …). Free
text is **redacted locally** (tokens, emails, home paths) before sending and re-scrubbed
server-side; repeat reports of the same title within a week **fold into one record**
(`status: duplicate`). Project/org/branch context, CLI version, and OS attach automatically when
available. Via MCP: the `insta_feedback` tool takes the same fields (plus explicit
`projectId`/`branch`).

## Govern & observe

- **Policy** gates `secrets.read`, `secrets.write`, `deploy`, `branch.delete`, `project.delete`,
  `storage.read` (`storage list` + `storage get`), `storage.delete`, and
  `service.add` / `service.remove` / `service.rename` / `service.scale` / `service.upgrade` /
  `service.setAccess`. `approve` = require a
  human: the action returns `approval_required` — the hint prints to **stderr** and the command
  **exits 2** (CLI ≥ 0.0.37; distinct from exit 1 = error, so treat exit 2 as "pending, not
  failed"); an admin runs `insta approvals approve <id>`, then
  you **re-run** it (single-use grant). `project.delete` and `service.remove` are gated by default. `--always` on approve
  flips the policy to `allow`.
- `insta approvals list` / `approve <id>` / `deny <id>` — manage pending gates.
- `insta events [--branch] [--limit]` — timeline of resource side-effects (project/branch creates,
  deploys + URLs, govern decisions) plus ingested agent events.
- **`insta observe`** — the local credential-audit hook (a PostToolUse hook for Claude Code / Codex,
  auto-installed on `project create`/`link`). It scans each tool-use for credential exposure
  (AWS / GitHub / Stripe / LLM / DB / JWT / private keys) and appends **redacted fingerprints** (never
  raw secrets) to `./.insta/audit.jsonl`. `insta observe report` renders it; `insta observe sync`
  uploads findings into the project timeline (idempotent).
