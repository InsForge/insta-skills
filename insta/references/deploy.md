# Deploy

Ship code to a branch's compute — image or source — and verify it actually serves.

## Two modes (pick exactly one)

```bash
insta --agent deploy --image <registry/img> --port <n>    # prebuilt image — ALWAYS pass --port
insta --agent deploy <dir> --port <n>                     # source dir — REQUIRES a Dockerfile
# both: [--branch <b>] targets another branch · [--group <g>] picks a compute service by name
```

Targets the **current branch's** sole compute service by default; the URL prints on success.

Before source deploys, run `insta --agent build <dir> --port <n>`. It is local/offline and catches the
common failures before the remote build: missing Dockerfile/start command, wrong or undetected port,
unexpected `.env.example` keys, and an oversized Docker context. Read the verdict literally: only
`deployable` means `insta --agent deploy <dir>` will build it — a dir with no Dockerfile that nixpacks
detects stops at `needs-attention` (⚠ Dockerfile check), because this path needs the dir's own
Dockerfile (CLI ≥ 0.0.48). `--explain` shows the Dockerfile — yours, or the nixpacks one **for
inspection only** (not standalone; do not save it as `Dockerfile`); use `--json` when an agent needs
structured output.

Never run a bare `insta --agent deploy <dir>` and assume the port: without `--port` older CLIs default
to 8080 regardless of the Dockerfile (boots "fine", every request refused — see below). Newer
CLIs default from the Dockerfile's `EXPOSE` and print what they picked — read that line and
confirm it matches the server's listen port.

## How source mode builds (what actually happens)

1. The dir must contain a `Dockerfile`. There is **no nixpacks/buildpack lane on this path** — the CLI exits 1 without one. Dockerfile-less options: add one from the templates below (run `insta --agent build <dir>` first: it reports the detected install/start commands to base it on, and only a dir with its own Dockerfile verdicts `deployable`), use `--image`, or connect the repo on GitHub — that server-side lane builds Dockerfile-less repos with nixpacks. Do **not** save the nixpacks Dockerfile that `insta --agent build --explain` prints as your `Dockerfile`: it `COPY`s `.nixpacks/` support files the dir does not have.
2. Needs the `fly` CLI locally (auto-installed via Homebrew on macOS) but **NO Fly account/login** —
   the platform mints a **short-lived, app-scoped deploy token** (this mint is govern-gated: it can
   return `approval_required` *before* any build runs).
3. The build runs on **remote builders** (no local Docker); the image is pushed and **pinned by
   digest** (tags race the registry), then deployed like any image.
4. insta-oss: source mode is not implemented yet — use `--image`.

## `--port` — the #1 deploy mistake

**`--port` must equal the port the app LISTENS on inside the container** (`EXPOSE` / server bind).
A mismatch boots "successfully" but every request fails (`instance refused connection`). Bind to
`0.0.0.0`, never `127.0.0.1`. On insta-oss it's also the host port for direct deploys; branch
clones keep the listen port and shift the **host** mapping +1000.

## Secrets at runtime

Compute env is explicit. At deploy, the platform injects:

- `PORT`
- user-defined secrets visible to that compute service (`insta --agent secrets set`, project/branch or
  compute-scoped)
- provider credentials you explicitly bound with `insta --agent secrets bind`

Provider-minted credentials are **not** injected just because the project has a postgres, redis,
mysql, mongodb, or storage service. Bind each credential the app needs, then deploy/redeploy:

```bash
insta --agent secrets sources
insta --agent secrets bind DATABASE_URL postgres/db --to compute/app
insta --agent secrets bind REDIS_URL redis/cache --source-name REDIS_URL --to compute/app
insta --agent deploy . --group app --port 8080
```

If the source has a single credential (`postgres`), `--source-name` is optional. Sources with several
credential names (`storage`, `redis`, `mysql`, `mongodb`) need `--source-name`. Production code reads
`process.env`; **never bake `./.env` into the image** (it's local-dev/user-secrets only). Changing a
secret or binding takes effect on the **next deploy**, or on **`insta --agent compute restart`** (CLI ≥
0.0.51) for a service already running — no hot reload in either case: the machine takes a new config
and restarts on it, in place. Whether an *idle* machine is woken to do so depends on the compute
provider; see [operate.md](operate.md) before treating a restart as proof the app came back.

Provider credential **values** reach two places by different routes. The local seam
(`insta --agent secrets` / `insta --agent run`) carries user-defined secrets **plus** each type's
**primary** service credentials, so `.env` and a local run have a working `DATABASE_URL` as soon as
the branch has a postgres. A **compute container** gets nothing it was not explicitly bound. For a
**specific** (non-primary) postgres there is also a direct read — `insta --agent db url` /
`insta --agent db connect` (gated `secrets.read`) — for psql, migrations, and tools outside compute; pick
client tools of the server's Postgres major first (`pg_version` on `insta --agent services list --json`; a row
without one falls back to the exact-version read in [operate.md](operate.md)).
Everything else runs where the credentials are bound: the deployed app itself, or a one-shot
`insta --agent compute exec app -- <cmd>` (≤180s, no stdin) — migrations run either way (never as a
startup gate; see the gotchas below).

## Verify before reporting (non-negotiable)

The deploy command exiting ≠ the app serving. After every deploy:

```bash
curl -s -o /dev/null -w '%{http_code}' <printed-url>   # poll ~every 3s, up to ~60s
```

A scale-to-zero service (`--no-always-on` at create, or `insta --agent compute always-on off`) cold-starts on the first request — allow a slow first hit; new compute services are born always-on (since 2026-09-07) and skip this. `200` (or the
app's expected status) → report deployed **with the URL**. Anything else → triage per
[operate.md](operate.md); never claim success you didn't observe.

## Deploy gotchas (each has burned real deploys)

- **Never gate container startup on migrations.** `CMD migrate && server` + a hung migration =
  a "successful" deploy that serves nothing, with empty logs. Run migrations non-blocking:
  `timeout 30 <migrate> || echo skipped; <start-server>`.
- **Cold start ≠ down.** A scale-to-zero compute service (`--no-always-on`, or switched off with `insta --agent compute always-on off`) suspends when idle; the first request wakes it. New compute is born always-on and does not.
- **Redeploy replaces.** Compute is stateless — anything written to the container filesystem is
  gone on the next deploy. State belongs in the branch's postgres/storage.

## Custom domains (bring your own)

```bash
insta --agent compute set-domain app.example.com [--branch --group]   # prints the DNS records to add
insta --agent compute check-domain app.example.com                    # status once DNS propagates
```

Cert + routing are handled for you; the DNS records live in **your** registrar (CNAME for a
subdomain, A/AAAA for an apex, + a validation CNAME).

## Dockerfile templates → use the framework recipes

**Before hand-writing a Dockerfile, copy the recipe for your framework: [frameworks.md](frameworks.md).**
Next.js, Node/Express, Vite/SPA, and FastAPI each have a paste-and-deploy recipe with the four
first-deploy traps already solved (bind `::` not IPv4-only; `EXPOSE` == listen port so `--port`
auto-derives; `PORT` env matches; multi-stage build). Skipping this is why a first deploy boots
"fine" yet refuses every request. Full-stack = one container/one port (backend serves the built
frontend); separate SPA = its own tiny static-server compute service.
