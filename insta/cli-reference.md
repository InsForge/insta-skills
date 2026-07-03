# InstaCloud CLI reference

Command catalog, deploy, Dockerfile templates, and govern/observe. For the development *process*
(branch → deploy → test → promote) and the deploy gotchas, see `workflow.md`.

## Commands

| Command | Purpose |
|---|---|
| `insta login --email <e> --password <p>` [`--api-url <url>`] · `insta login --oauth <github\|google>` · `insta logout` | auth (api-url + tokens persist; tokens auto-refresh). `--oauth` opens a browser (loopback capture) — for interactive use; agents use email/password or an API token |
| `insta status` [`--json`] | login + linked project + current branch |
| `insta org list` [`--json`] · `insta org create <name>` | organizations (**one free org per user** — upgrade an existing org before creating another) |
| `insta project create <name>` [`--org <id>`] | create an **empty** project (no services), link this dir |
| `insta project list` [`--org`] [`--json`] · `insta project link <id>` | list / link existing |
| `insta project delete` | tear down ALL resources + unlink (gated: `project.delete`, approval by default) |
| `insta services add <postgres\|storage\|compute> <name>` | provision a service on demand; postgres/compute get a default access domain (gated: `service.add`) |
| `insta services list` [`--json`] · `insta services remove <type> <name>` | list / remove services (remove gated: `service.remove`) |
| `insta services scale compute <name> <number>` [`region`] | set compute machine count — **paid plans only** (free → 403); gated: `service.scale` |
| `insta services upgrade <compute\|postgres> <name> <spec>` | raise spec (up-only) — **paid plans only**; gated: `service.upgrade`. compute: `1vcpu-256mb`→`2vcpu-2gb`; postgres: `pg-0.25cu`→`pg-4cu` |
| `insta branch create <name>` [`--from <parent>`] | isolated env: materializes the project's **current** services (Neon branch + forked bucket + a clone of every compute service). **≤10 branches/project.** Does NOT switch |
| `insta branch switch <name>` · `insta branch list` [`--json`] | set current branch / list |
| `insta branch delete <name>` | tear down the branch's resources (gated: `branch.delete`) |
| `insta secrets` [`--branch <name>`] [`-o <file>`] [`--print`] [`--json`] | secret seam → write creds to `./.env` (gated: `secrets.read`) |
| `insta secrets list` [`--branch`] | list secret names only |
| `insta deploy <dir>` / `--image <url>` [`--branch <b>`] [`--group <g>`] [`--port <n>`] | deploy to a compute service — a **source dir** (needs a `Dockerfile`; built remotely on Fly, no local Docker) or a **prebuilt image**. Defaults to the branch's sole compute service; `--group` picks by name (gated: `deploy`) |
| `insta compute set-domain <host>` / `check-domain <host>` / `remove-domain <host>` [`--branch --group --json`] | attach / check / detach a **developer-owned custom domain** on a compute service — Fly issues the cert + routes; prints the DNS records to set in **your own** registrar (set/remove gated: `deploy`) |
| `insta manifest` [`--json`] | agent-legible env view: each branch's db / storage / compute + URLs |
| `insta metrics <db\|compute>` [`group`] [`--branch --from --to --step --json`] | service metrics (compute=Fly; db=provider-limited) |
| `insta logs <db\|compute>` [`group`] [`--branch --limit --region --instance --json`] | runtime logs (compute=Fly; db=provider-limited) |
| `insta usage` [`--from --to --json`] | usage aggregated by meter, with `costUsd` (snapshotted at collection) |
| `insta billing` [`--org <id>`] [`--json`] | current cycle summary: tier / included credit / used / overage / status |
| `insta billing upgrade <pro\|enterprise>` · `insta billing portal` [`--org`] [`--no-open`] [`--json`] | Stripe Checkout to subscribe / Customer Portal to manage (opens a browser; `--no-open` prints the URL) |
| `insta events` [`--branch <b>`] [`--limit <n>`] [`--json`] | audit + agent-event timeline |
| `insta policy get` [`--json`] · `insta policy set <action> <decision>` | view / set governance policy (actions include `service.add/remove/scale/upgrade`) |
| `insta approvals list` [`--status`] · `insta approvals approve <id>` [`--always`] · `insta approvals deny <id>` | manage gated actions |
| `insta observe install` · `report` [`--json`] · `sync` | local credential-audit hook (see below) |

`DATABASE_URL` + compute + storage (`AWS_*` / `BUCKET_NAME`) are **per-branch** (new projects: each
branch copy-on-write-forks its parent's bucket; a project created before snapshots keeps a **shared** bucket).

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

> Multiple compute services, `insta services scale`/`upgrade`, and **source-directory deploy**
> (`insta deploy <dir>` → Fly remote builder, no local Docker) are all implemented.
> More than one postgres/storage service per project is not yet supported (credential-seam limit).

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
unknown paths rewritten to `index.html`; deploy it as its own compute service or project.

## Govern & observe

- **Policy** gates `secrets.read`, `deploy`, `branch.delete`, `project.delete`, and `service.add` /
  `service.remove` / `service.scale` / `service.upgrade`. `approve` = require a
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
