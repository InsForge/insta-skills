# Migrate an app in from another platform

Move a running app (Heroku, Railway, Fly, Render) onto InstaCloud: provision, move env and data,
cut over. The human walkthroughs live at `docs.instacloud.com/migrate/<source>`; this is the
runbook — the ordering and the pass conditions that keep a cutover from silently losing writes.

## The ordered cutover

Each step has a condition that must hold before the next one runs. **The ordering is the point:**
once the target accepts writes, "roll back to the source" silently discards them.

**0. Link a project.** The cutover assumes one exists.

```bash
insta project create <name>        # or: insta project link <project-id>
```

Note this rewrites the **global** default link in `~/.insta/config.json`, not only the local
`.insta/project.json` it reports — any directory without its own link now points here.

**1. Provision, bind, deploy.**

```bash
insta services add postgres db                          # + redis/storage/… as the source needs
insta services add compute app --port <n>               # REQUIRED: the bind below targets it
insta secrets bind DATABASE_URL postgres/db --to compute/app
insta deploy --image <registry/img> --port <n>          # works on every compute plane
# or: insta deploy <dir> --port <n>                     # Dockerfile required, Fly-backed compute only
# or: insta compute connect-repo <owner/repo> app       # attaches to THIS service; nixpacks if no Dockerfile
```

Without the `services add compute` line the bind fails with `service not found on branch:
compute/app`. A deploy materializes env into the machine config, so the binding takes effect with
it. **`insta compute restart` is refused while a service has no image** ("this service has no
machines yet — deploy an image first, then retry"), so a first migration is bind → **deploy**, never
bind → restart.

Deploy an image that carries a **psql client** if you intend to verify from inside the app in step 5
— `nginx:alpine` and friends cannot.
*Pass:* the app boots and serves, even against an empty database.

**2. Stop the writers — on BOTH sides.**

Source: maintenance/read-only **and** stop its workers and cron. A read-only web tier with a live
worker is still writing. Heroku: `heroku maintenance:on` plus `heroku ps:scale worker=0`.
Railway / Fly / Render: no single maintenance switch — stop or scale each service by hand.

Target: `insta compute stop <service>` for what step 1 deployed, plus any worker.
**Do not infer target quiescence from "nothing has been rebound yet"** — after step 1 the target
app is live and can write.

**`stop` is a traffic barrier, not an execution barrier.** `insta compute exec` succeeds on a
stopped service and leaves it **live** (`status` then reads `desired=stopped live=running`), so any
`exec` — including a verification query in step 4 — re-animates the machine. Re-`stop` after using
it.
*Pass:* no write traffic at either end.

**3. Copy into a CLEAN target.**

**Match your `pg_dump` major to the TARGET server first.** `insta services list` prints the postgres
major for exactly this reason: a newer client emits statements an older server rejects. A pg18
`pg_dump` against insta's pg16 fails on the first `SET`:
`ERROR: unrecognized configuration parameter "transaction_timeout"`. What matters is the **client's**
version, not the source server's.

```bash
insta services list                       # read the postgres major, e.g. postgres/db [pg16]
PG=16                                     # match it

set -o pipefail
docker run --rm postgres:$PG pg_dump --no-owner --no-privileges "$SOURCE_URL" \
  | psql -v ON_ERROR_STOP=1 "$(insta db url --group db)" 2>&1 | tee restore.log
grep -c '^ERROR' restore.log              # must print 0
```

A full dump restored into a populated database is **not** incremental sync — it collides on existing
objects and primary keys, so the target must be empty. **Prefer adding a fresh postgres service**
over dropping the database: `DROP DATABASE` needs a DSN retargeted to `/postgres`, is blocked by
insta's own `pg_cron` session until you `pg_terminate_backend` it, and the recreated database
**loses the platform's preinstalled extensions** (`pgcrypto`, `uuid-ossp`, `pgaudit`, `vector`,
`pg_stat_monitor`, `pg_stat_statements` → only `plpgsql` survives). But if you add a fresh service,
the DSN changes — see step 5.

Which guard catches what: **`ON_ERROR_STOP=1` catches SQL errors** (psql is the last stage, so its
status is the pipeline's), **`pipefail` catches a `pg_dump` failure**. You need both.
*Pass:* `grep -c '^ERROR'` is 0. Exit 0 alone proves nothing.

**4. Verify the data.**

```bash
T="$(insta db url --group db)"             # scriptable; `insta db connect` is interactive
psql "$T" -c "select relname, n_live_tup from pg_stat_user_tables order by n_live_tup desc limit 5"
psql "$T" -c "select max(id), max(created_at) from <append_only_table>"
psql "$T" -c "select sequencename, last_value from pg_sequences order by sequencename"
psql "$T" -c "select extname from pg_extension order by extname"
```

Run the same four against the source and diff. **"Extensions present" cannot fail** on its own — a
fresh insta postgres already ships `pgcrypto`, `uuid-ossp`, `pgaudit`, `vector`, `pg_stat_monitor`
and `pg_stat_statements`, so the dump's `CREATE EXTENSION IF NOT EXISTS` is a no-op. Compare the
**sets** source-vs-target instead of asserting presence.
*Pass:* counts and latest rows match; sequences at or above the source's; extension sets reconciled.

**5. Bring the app onto the target — and `start` does NOT re-resolve env.**

```bash
# binding unchanged (you restored into the same postgres service):
insta compute start <service>

# binding CHANGED (you restored into a fresh postgres service):
insta compute start <service> && insta compute restart <service>
```

`insta compute start` is a machine-lifecycle operation only. On a stopped-and-rebound service it
brings the machine back **with the env it was deployed with**, so the app keeps writing to the
pre-migration database — while `insta secrets bindings` already reports the new source. `restart`
does re-resolve (`restarted … — env re-resolved from the current secrets`) but is **refused on a
stopped service**, so the changed-binding case is `start` *then* `restart`.

*Pass:* **check the machine, not the intent.**

```bash
insta compute exec <service> -- printenv DATABASE_URL    # must name the NEW postgres service
```

`insta secrets bindings` reports what *should* be bound and will show the new source even while the
machine holds the old DSN — a false pass at the exact moment the rollback boundary is crossed. Then
confirm the app reads **and writes** the new database.

**6. Cut traffic.**

```bash
insta compute set-domain <host> --group <service>       # host is positional; service is --group
insta compute check-domain <host> --group <service>
```

`set-domain <service> <host>` fails with `invalid domain`. It returns the DNS records for you to
publish at your provider — it does not change your DNS.

**7. Decommission the source** — after a soak period, not before.

**Rollback boundary.** Through step 4, returning to the source is a clean revert. **From step 5 the
target may hold writes the source does not** — rollback then needs a reverse copy or an accepted
data loss. "If verification fails, just point back at the source" is wrong once the target is live.

## What no source-platform guide will tell you

| | |
|---|---|
| **A binding is not live until a deploy** | Env is materialized into machine config at deploy time. `insta secrets bind` changes the rules only; the running machine keeps its old env until `insta deploy` (first time) or `insta compute restart` (already running). Until then **the app still writes to the old database.** |
| **`--port` must equal the listen port** | `PORT` is injected as the routed port. An app reading `$PORT` is fine; a hardcoded port boots "successfully" and refuses every request. Source deploys default from the Dockerfile's last `EXPOSE` — read the line the CLI prints and confirm it. |
| **Three routes get code in on every plane** | `insta deploy --image`; `insta compute connect-repo <owner/repo> <service>` (attaches to an EXISTING service and builds its Dockerfile, or detects the runtime with nixpacks when there is none — `--public` needs no GitHub App, `--root-dir` handles a monorepo); or the console's repo binding, which CREATES a service rather than attaching. `insta deploy <dir>` needs a Dockerfile **and** Fly-backed compute — on insta-compute the platform refuses it outright, Dockerfile or not, with `source builds are not supported on the insta-compute provider yet`. Do not plan a migration around it unless you have confirmed the target's plane. |
| **Postgres scales to zero** | Keep the pool's `idleTimeoutMillis` under the suspend window, or the first request after a wake fails on a dead pooled connection. |
| **No bulk env import** | `insta secrets set <name>` takes one variable per call (value as an argument or on stdin). Loop over the source's export, and drop the platform's own vars — `HEROKU_*`, `RAILWAY_*`, `DYNO`, `PORT`. |
| **`insta secrets list` prints names only** | It cannot reveal a truncated or mis-escaped value. To compare values, use `insta secrets --print --json` — **not** bare `--print`, which double-quotes every value and does not escape embedded newlines, so a multi-line value breaks line-oriented parsing and every key then digests differently from the source export. |
| **No app-level scheduler — but the DB has one** | There is no `insta schedule`. Two options. In-process (node-cron, APScheduler, whenever) inside a **web** service: keep it always-on, since a suspended service stops firing, and remember **replicas multiply every tick** (`insta services scale` allows 1–10, so two replicas run each job twice). Or **`pg_cron`**, which insta postgres ships (a `pg_cron launcher` worker is already running; `CREATE EXTENSION pg_cron`) — one schedule, no replica problem, but SQL-only. |
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
