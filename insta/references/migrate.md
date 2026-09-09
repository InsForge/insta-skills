# Migrate an app in from another platform

Move a running app (Heroku, Railway, Fly, Render) onto InstaCloud: provision, move env and data,
cut over. The human walkthroughs live at `docs.instacloud.com/migrate/<source>`; this is the
runbook — the ordering and the pass conditions that keep a cutover from silently losing writes.

## The ordered cutover

Each step has a condition that must hold before the next one runs. **The ordering is the point:**
once the target accepts writes, "roll back to the source" silently discards them.

**1. Provision, bind, deploy.**

```bash
insta services add postgres db                          # + redis/storage/… as the source needs
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta deploy --image <registry/img> --port <n>          # works on every compute plane
# or: insta deploy <dir> --port <n>                     # Dockerfile required, Fly-backed compute only
```

A deploy materializes env into the machine config, so the binding takes effect with it.
**`insta compute restart` is refused while a service has no image** ("this service has no machines
yet — deploy an image first, then retry"), so a first migration is bind → **deploy**, never
bind → restart.
*Pass:* the app boots and serves, even against an empty database.

**2. Stop the writers — on BOTH sides.**

Source: maintenance/read-only **and** stop its workers and cron. A read-only web tier with a live
worker is still writing. Heroku: `heroku maintenance:on` plus `heroku ps:scale worker=0`.
Railway / Fly / Render: no single maintenance switch — stop or scale each service by hand.

Target: `insta compute stop <service>` for what step 1 deployed, plus any worker.
**Do not infer target quiescence from "nothing has been rebound yet"** — after step 1 the target
app is live and can write.
*Pass:* no write traffic at either end.

**3. Copy into a CLEAN target.**

```bash
set -o pipefail
pg_dump --no-owner --no-privileges "$SOURCE_URL" \
  | psql -v ON_ERROR_STOP=1 "$(insta db url --group db)"
```

A full dump restored into a populated database is **not** incremental sync — it collides on existing
objects and primary keys. Drop and recreate the target database (or add a fresh postgres service)
and restore **once**.

**`psql` continues past errors by default, and a shell pipeline reports only its last command's
status.** Without `ON_ERROR_STOP=1` *and* `pipefail`, a broken restore exits 0 and reads as success.
Near-zero-downtime via logical replication is out of scope here.
*Pass:* **no SQL errors** — not merely exit 0.

**4. Verify the data.** Row counts on the largest tables, the newest row in each append-only table,
extensions present, and sequence positions.
*Pass:* counts and latest rows match the source; sequences at or above the source's.

**5. Bring the app onto the target.** Rebind first if the restore went into a *fresh* service. Then:

```bash
insta compute start <service>      # if step 2 stopped it
insta compute restart <service>    # if still running and only the binding changed
```

**`restart` is refused on a stopped service** and tells you to use `start`, which also re-enables
auto-wake.
*Pass:* `insta secrets bindings --target compute/app` shows the intended source, **and** the app
both reads **and writes** the new database.

**6. Cut traffic.** `insta compute set-domain <service> <host>`, confirm with `check-domain`.

**7. Decommission the source** — after a soak period, not before.

**Rollback boundary.** Through step 4, returning to the source is a clean revert. **From step 5 the
target may hold writes the source does not** — rollback then needs a reverse copy or an accepted
data loss. "If verification fails, just point back at the source" is wrong once the target is live.

## What no source-platform guide will tell you

| | |
|---|---|
| **A binding is not live until a deploy** | Env is materialized into machine config at deploy time. `insta secrets bind` changes the rules only; the running machine keeps its old env until `insta deploy` (first time) or `insta compute restart` (already running). Until then **the app still writes to the old database.** |
| **`--port` must equal the listen port** | `PORT` is injected as the routed port. An app reading `$PORT` is fine; a hardcoded port boots "successfully" and refuses every request. Source deploys default from the Dockerfile's last `EXPOSE` — read the line the CLI prints and confirm it. |
| **Only two routes get code in on every plane** | `insta deploy --image`, or a connected GitHub repo (console-only today, and it CREATES the compute service rather than attaching to one you made). `insta deploy <dir>` needs a Dockerfile **and** Fly-backed compute — on insta-compute the platform refuses it outright, Dockerfile or not, with `source builds are not supported on the insta-compute provider yet`. Do not plan a migration around it unless you have confirmed the target's plane. |
| **Postgres scales to zero** | Keep the pool's `idleTimeoutMillis` under the suspend window, or the first request after a wake fails on a dead pooled connection. |
| **No bulk env import** | `insta secrets set <name>` takes one variable per call (value as an argument or on stdin). Loop over the source's export, and drop the platform's own vars — `HEROKU_*`, `RAILWAY_*`, `DYNO`, `PORT`. |
| **`insta secrets list` prints names only** | It cannot reveal a truncated or mis-escaped value. To check values, compare per-key digests of the **raw bytes** between the source export and `insta secrets --print`, so trailing spaces and newlines are caught. |
| **No cron / scheduler** | There is no `insta schedule`. Run an in-process scheduler (node-cron, APScheduler, whenever) inside a **web** service. Keep it always-on — a suspended service stops firing — and remember **replicas multiply every tick** (`insta services scale` allows 1–10, so two replicas run each job twice). |
| **Workers** | `port === 0` is the platform's own worker convention, but `insta services add --port 0` is rejected and `insta template deploy` refuses `type: worker`. **Until that path is verified end to end**, give the worker a port and let it listen — the machine check is **TCP, not HTTP**, so `require('net').createServer().listen(process.env.PORT)` is enough (no framework, no `/health`). Never `--no-always-on`: a suspended worker has no inbound traffic to wake it. |

## Command mapping

| Need | Heroku | InstaCloud |
|---|---|---|
| dump all env | `heroku config -s` | `insta secrets --print` |
| set one env | `heroku config:set K=V` | `insta secrets set K` (value on stdin) |
| DB connection string | `heroku config:get DATABASE_URL` | `insta db url` |
| psql session | `heroku pg:psql` | `insta db connect` |
| one-off task | `heroku run <cmd>` | `insta compute exec [service] -- <cmd>` (argv, no shell) |
| stop traffic | `heroku maintenance:on` | `insta compute stop [service]` |
| scale | `heroku ps:scale web=2` | `insta services scale compute <name> 2` |
| custom domain | `heroku domains:add` | `insta compute set-domain` |
| logs | `heroku logs -t` | `insta logs compute` (target is required) |

## Addon → service

| Source | Provision | Bound as |
|---|---|---|
| Heroku / Railway Postgres | `insta services add postgres <n>` | `DATABASE_URL` |
| Heroku / Railway Redis | `insta services add redis <n>` | `REDIS_URL` |
| JawsDB, PlanetScale | `insta services add mysql <n>` | `MYSQL_URL` |
| MongoDB Atlas | `insta services add mongodb <n>` | `MONGODB_URL` |
| S3 bucket, Railway bucket | `insta services add storage <n>` | `AWS_*`, `BUCKET_NAME` |
| Heroku Scheduler, Railway cron | none — see the table above | |

Bind every credential the app needs; nothing is auto-injected into compute.

## Per-source deltas

**Heroku.** The richest export surface: `config -s` yields `KEY=value` lines, `pg:backups` and
`maintenance:on` are single commands, and the `Procfile`'s `web:` / `worker:` map straight onto
compute services. No volumes. `app.json`, if present, declares the addons — read it to enumerate
what to provision.

**Railway.** Closest model (services + variables + IaC), so the concept mapping is nearly 1:1 — but
the export is more manual. Variables may be **reference variables** (`${{Postgres.DATABASE_URL}}`)
that must be resolved to literals first; services are enumerated one at a time rather than from a
single file; and there is **no maintenance-mode equivalent**, so step 2 means stopping each service
by hand. Railway apps may carry a **volume** — creating a target volume does **not** copy its
contents, so either copy and verify it explicitly or state that it is out of scope.

**Fly / Render.** Same spine. Render declares services and databases in `render.yaml` — read it to
enumerate. Fly's `fly.toml` `[processes]` block maps onto compute services, and Fly apps may carry
volumes with the same caveat as Railway.

> Unverified as of 2026-09-08: no migration has been run end to end against this runbook. The
> portless-worker path, object-storage and volume data movement, and Railway's lack of a maintenance
> mode are all read from code and docs, not from a completed migration.
