# Deploy

Ship code to a branch's compute — image or source — and verify it actually serves.

## Two modes (pick exactly one)

```bash
insta deploy --image <registry/img> --port <n>    # prebuilt image — ALWAYS pass --port
insta deploy <dir> --port <n>                     # source dir — REQUIRES a Dockerfile
# both: [--branch <b>] targets another branch · [--group <g>] picks a compute service by name
```

Targets the **current branch's** sole compute service by default; the URL prints on success.

Before source deploys, run `insta build <dir> --port <n>`. It is local/offline and catches the
common failures before the remote build: missing Dockerfile/start command, wrong or undetected port,
unexpected `.env.example` keys, and an oversized Docker context. Use `--explain` to see the
Dockerfile that would be used; use `--json` when an agent needs structured output.

Never run a bare `insta deploy <dir>` and assume the port: without `--port` older CLIs default
to 8080 regardless of the Dockerfile (boots "fine", every request refused — see below). Newer
CLIs default from the Dockerfile's `EXPOSE` and print what they picked — read that line and
confirm it matches the server's listen port.

## How source mode builds (what actually happens)

1. The dir must contain a `Dockerfile` (no buildpacks yet — no Dockerfile → use `--image`, or add one from the templates below).
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
- user-defined secrets visible to that compute service (`insta secrets set`, project/branch or
  compute-scoped)
- provider credentials you explicitly bound with `insta secrets bind`

Provider-minted credentials are **not** injected just because the project has a postgres, redis,
mysql, mongodb, or storage service. Bind each credential the app needs, then deploy/redeploy:

```bash
insta secrets sources
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta secrets bind REDIS_URL redis/cache --source-name REDIS_URL --to compute/app
insta deploy . --group app --port 8080
```

If the source has a single credential (`postgres`), `--source-name` is optional. Sources with several
credential names (`storage`, `redis`, `mysql`, `mongodb`) need `--source-name`. Production code reads
`process.env`; **never bake `./.env` into the image** (it's local-dev/user-secrets only). Changing a
secret or binding takes effect on the **next deploy** — no hot reload.

Provider credential **values** stay out of the general bundle (`insta secrets` / `insta run` carry
only user-defined secrets). The one direct read is the postgres DSN — `insta db url` /
`insta db connect` (gated `secrets.read`) — for psql, migrations, and tools outside compute.
Everything else runs where the credentials are bound: the deployed app itself, or a one-shot
`insta compute exec app -- <cmd>` (≤180s, no stdin) — migrations run either way (never as a
startup gate; see the gotchas below).

## Verify before reporting (non-negotiable)

The deploy command exiting ≠ the app serving. After every deploy:

```bash
curl -s -o /dev/null -w '%{http_code}' <printed-url>   # poll ~every 3s, up to ~60s
```

Scale-to-zero branches (the default) cold-start on the first request — allow a slow first hit; always-on services (`insta compute always-on on`) skip this. `200` (or the
app's expected status) → report deployed **with the URL**. Anything else → triage per
[operate.md](operate.md); never claim success you didn't observe.

## Deploy gotchas (each has burned real deploys)

- **Never gate container startup on migrations.** `CMD migrate && server` + a hung migration =
  a "successful" deploy that serves nothing, with empty logs. Run migrations non-blocking:
  `timeout 30 <migrate> || echo skipped; <start-server>`.
- **Cold start ≠ down.** Non-default branches suspend when idle; first request wakes them.
- **Redeploy replaces.** Compute is stateless — anything written to the container filesystem is
  gone on the next deploy. State belongs in the branch's postgres/storage.

## Custom domains (bring your own)

```bash
insta compute set-domain app.example.com [--branch --group]   # prints the DNS records to add
insta compute check-domain app.example.com                    # status once DNS propagates
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
