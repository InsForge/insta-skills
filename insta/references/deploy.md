# Deploy

Ship code to a branch's compute — image or source — and verify it actually serves.

## Two modes (pick exactly one)

```bash
insta deploy --image <registry/img> --port <n>    # prebuilt image — ALWAYS pass --port
insta deploy <dir> --port <n>                     # source dir — REQUIRES a Dockerfile
# both: [--branch <b>] targets another branch · [--group <g>] picks a compute service by name
```

Targets the **current branch's** sole compute service by default; the URL prints on success.

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

Secrets are **injected at deploy** as env vars (decrypted for that branch — platform creds + your
`insta secrets set` values). Production code reads `process.env`; **never bake `./.env` into the
image** (it's local-dev only). Changing a secret takes effect on the **next deploy** — no hot reload.

## Verify before reporting (non-negotiable)

The deploy command exiting ≠ the app serving. After every deploy:

```bash
curl -s -o /dev/null -w '%{http_code}' <printed-url>   # poll ~every 3s, up to ~60s
```

Scale-to-zero branches cold-start on the first request — allow a slow first hit. `200` (or the
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

## Dockerfile templates

**Backend** (reads creds from env, listens on `--port`):
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]   # reads process.env.DATABASE_URL etc.
```

**Full-stack:** ship frontend + backend as **one container, one port** — the backend serves the
built frontend. One deploy, one URL, same-origin calls.
**Separate SPA:** build static assets, serve from a tiny static-server container (e.g. caddy) with
unknown paths rewritten to `index.html`; deploy as its own compute service.
