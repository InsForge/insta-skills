# InstaCloud CLI reference

Command catalog, deploy, Dockerfile templates, and govern/observe. For the development *process*
(branch → deploy → test → promote) and the deploy gotchas, see `workflow.md`.

## Commands

| Command | Purpose |
|---|---|
| `insta login --email <e> --password <p>` [`--api-url <url>`] · `insta logout` | auth (api-url + tokens persist; tokens auto-refresh) |
| `insta status` [`--json`] | login + linked project + current branch |
| `insta org list` [`--json`] · `insta org create <name>` | organizations |
| `insta project create <name>` [`--org <id>`] | provision DB + storage + compute, link this dir |
| `insta project list` [`--org`] [`--json`] · `insta project link <id>` | list / link existing |
| `insta project delete` | tear down ALL resources + unlink (gated: `project.delete`, approval by default) |
| `insta branch create <name>` [`--from <parent>`] | isolated env: Neon branch + forked bucket + a clone of **every** compute group (does NOT switch) |
| `insta branch switch <name>` · `insta branch list` [`--json`] | set current branch / list |
| `insta branch delete <name>` | tear down the branch's resources (gated: `branch.delete`) |
| `insta secrets` [`--branch <name>`] [`-o <file>`] [`--print`] [`--json`] | secret seam → write creds to `./.env` (gated: `secrets.read`) |
| `insta secrets list` [`--branch`] | list secret names only |
| `insta deploy --image <url>` [`--branch <b>`] [`--group <g>`] [`--port <n>`] | deploy an image to a branch's compute group (gated: `deploy`) |
| `insta manifest` [`--json`] | agent-legible env view: each branch's db / storage / compute + URLs |
| `insta events` [`--branch <b>`] [`--limit <n>`] [`--json`] | audit + agent-event timeline |
| `insta policy get` [`--json`] · `insta policy set <action> <decision>` | view / set governance policy |
| `insta approvals list` [`--status`] · `insta approvals approve <id>` [`--always`] · `insta approvals deny <id>` | manage gated actions |
| `insta observe install` · `report` [`--json`] · `sync` | local credential-audit hook (see below) |

`DATABASE_URL` + compute + storage (`AWS_*` / `BUCKET_NAME`) are **per-branch** (new projects: each
branch copy-on-write-forks its parent's bucket; a project created before snapshots keeps a **shared** bucket).

## Deploy

```
insta deploy --image <url> --port <n>      # deploy a pre-built / already-pushed image
# targets the CURRENT branch's `default` compute group; --branch / --group target another; the URL prints on success
```

`--port` must match the port the image actually listens on (`ENV PORT` / `EXPOSE` / server bind) — a
mismatch boots fine but every request fails with `instance refused connection on 0.0.0.0:<port>`.
Secrets are **injected at deploy** as env vars (decrypted from the branch). Read creds from
`process.env` in production; **never bake `./.env` into the image**. A compute group serves one app
on one port at `https://<app>.fly.dev`.

> Building from source (remote builder, no local Docker), compute-group management
> (`add-group`/`scale`/`set-domain`), and `db scale` are planned; today deploy takes a pre-built image.

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
**Full-stack:** ship frontend + backend as **one container, one port** — have the backend serve the
built frontend. One `insta deploy`, one URL, same-origin calls.
**Separate SPA:** build static assets, serve from a tiny static-server container (e.g. caddy) with
unknown paths rewritten to `index.html`; deploy it as its own compute group or project.

## Govern & observe

- **Policy** gates `secrets.read`, `deploy`, `branch.delete`, `project.delete`. `approve` = require a
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
