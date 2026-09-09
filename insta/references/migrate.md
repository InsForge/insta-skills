# Migrate an app in from another platform

Move a running app (Heroku, Railway, Fly, Render) onto InstaCloud: provision, move env and data,
cut over. The reader-facing walkthroughs are `docs.instacloud.com/migrate/render` and
`/migrate/railway`, and they are deliberately thin: they hand the user a prompt and point at this
runbook, so **this file is what actually gets followed.** Those pages deliberately do NOT list these
steps, so do not add detail there when it belongs here. The one thing they do promise the reader is
that the source stops taking writes before the target starts, which is the rollback boundary below.
Everything else lives here: the ordering and the pass conditions that keep a cutover from silently
losing writes.

## The ordered cutover

Each step has a condition that must hold before the next one runs. **The ordering is the point:**
once the target accepts writes, "roll back to the source" silently discards them.

**0. Link a project.** The cutover assumes one exists.

```bash
insta project create <name>        # or: insta project link <project-id>
```

**It rewrites `~/.insta/project.json`, the GLOBAL default link**, so any directory without its own
link now points here. Capture that file's contents before you run this, and restore them after.
Do not trust the command's own output: it prints `linked ./.insta/project.json`, which reads as
local, but **no local `.insta/` directory is created at all** (verified on prod, 2026-09-09). Note
this is a *different* file from `~/.insta/config.json`, which holds the env and session and carries
no project link.

**1. Provision, bind, deploy.**

**Pre-flight, before you deploy anything: find out how the app learns its own hostname.** This is
pure code reading, it needs no platform access, and doing it now is the difference between a planned
step and a mystery 400 after the cutover. Open the app's settings and answer two questions:

- **Does it gate anything on its public host?** Django `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS`,
  Rails `config.hosts`, Phoenix `check_origin`, and any OAuth callback, cookie domain or absolute
  link builder.
- **Where does it read the host from?** A variable you can set (Render's
  `RENDER_EXTERNAL_HOSTNAME`), or a literal you must edit (Fly's `.fly.dev`, Heroku's fallback
  list)? See the per-source table in the Render section for what each platform's apps actually do.

**Write down the variable name, or the file and line to change.** You cannot set the value yet —
on insta-compute the host is minted by the plane at first deploy, so it does not exist until after
the deploy below (`adapters/insta-compute.ts`: routeKey is "learned at first deploy", and
`access_host` "is the only source of truth"). **That is why this is two steps: decide here, apply in
step 5.** If the answer was "a literal I must edit", make that edit NOW, before the deploy, so the
image is already right.

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

**Postgres exposes exactly one credential: `DATABASE_URL`.** If the source app reads the discrete
components instead — `PGHOST` / `PGUSER` / `PGPASSWORD` / `PGDATABASE` / `PGPORT`, which is what
Railway injects by default and what `railwayapp-templates/django` reads via `os.environ[...]` —
**point the app at the single DSN rather than trying to reproduce the five.** In Django that is
`dj-database-url`; most stacks accept a DSN directly. Do this as part of the migration, not after.

The reason it must be a code change is that the alternative fails *silently*. `insta secrets bind`
validates the env name only against `^[A-Z][A-Z0-9_]{0,63}$`, and for a postgres source
`--source-name` defaults to the only allowed key, so

```bash
insta secrets bind PGHOST postgres/db --to compute/app    # accepted, and WRONG
```

is accepted and sets `PGHOST` to the **whole connection string**. Nothing complains at bind time;
the app fails later trying to resolve a hostname that is actually a URL. (From
`insta-platform/src/provisioning/userSecrets.ts:78,88` and `secretNames.ts:5`, read at `a79b067`;
not executed.) Splitting the DSN into five plain secrets with `insta secrets set` does work, but
they are then static copies that no longer follow a rotation, which is the whole point of a
binding. Note this asymmetry is postgres-only: `redis`, `mysql` and `mongodb` each expose their
components alongside the URL, so binding `REDIS_HOST` or `MYSQL_USERNAME` is fine.

Deploy an image that carries a **psql client** if you intend to verify from inside the app in step 5
— `nginx:alpine` and friends cannot.
*Pass:* **curl the URL and read the status** — not `insta compute status`, which reports a service
healthy whenever the port accepts TCP, so an app that refuses every request looks identical to one
that works.

```bash
insta services list                    # read the compute row's host column
curl -s -o /dev/null -w '%{http_code}\n' "https://<that host>"
```

Read the host rather than parsing it out of the row: the column position shifts on a service that
has no image yet, so a clever one-liner can hand you the wrong string silently.

Any 2xx/3xx, or a 5xx from the app's own code, means it is serving and step 5 can proceed. **A 400
here is the hostname problem from the pre-flight**, not a database or build fault, and it is fixed
in step 5 rather than by redeploying. Working against an empty database is expected at this point.

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

**Read both majors first.** They decide which client you need, and whether the schema has hard
blockers.

```bash
insta services list                       # target major, e.g. postgres/db [pg16], NOT selectable
psql "$SOURCE_URL" -c 'show server_version'
```

The one hard rule is `client_major >= source_major`. A newer server cannot be read by an older
client and there is no escape hatch: `pg_dump` 16 against an 18 server aborts with
`pg_dump: error: aborting because of server version mismatch`, and `--format=custom` makes it worse,
not better, because `pg_restore` 16 rejects an 18 archive at the header
(`unsupported version (1.16) in file header`). So always dump with a client at or above the source
major, and use plain format when the target is older, because plain text is the only form you can
filter.

**Upgrade or equal (source <= target).** Nothing special.

```bash
set -o pipefail
pg_dump --no-owner --no-privileges "$SOURCE_URL" \
  | psql -v ON_ERROR_STOP=1 "$(insta db url --group db)" 2>&1 | tee restore.log
grep -c '^ERROR' restore.log              # must print 0
```

**Downgrade (source > target).** Today this is every documented source: Render pg18 and Railway
pg18 into InstaCloud pg16. **This works, at full fidelity, and it is a tested procedure**, not a
workaround. `pg_dump` from 18 emits exactly one statement pg16 does not know.

```bash
set -o pipefail
pg_dump --format=plain --no-owner --no-privileges "$SOURCE_URL" \
  | grep -v -E '^(SET transaction_timeout|\\restrict |\\unrestrict )' \
  | psql -v ON_ERROR_STOP=1 "$(insta db url --group db)" 2>&1 | tee restore.log
grep -c '^ERROR' restore.log              # must print 0
```

Why each piece is there:

- `SET transaction_timeout = 0;` is a PG17 GUC. Of the 12 `SET`s a PG18 `pg_dump` emits, this is the
  **only** one pg16 rejects. In a 1,400 line realistic dump it is the single offending line.
- `\restrict` / `\unrestrict` are psql meta-commands added by the CVE-2025-8714 fix. They fail only
  on psql older than 15.14 / 16.10 / 17.6 (`invalid command \restrict`, exit 3). Filtering them
  makes the command work on any psql, at the cost of that guard. Acceptable when the source is the
  user's own database, not acceptable for a dump from a third party.
- `--no-owner --no-privileges` is **not optional** against Render or Railway. Without it the restore
  dies on `ERROR: role "render_app" does not exist`.

Custom-format archives have no filter hook, so route them through text:

```bash
pg_restore --no-owner --no-privileges -f - source.dump \
  | grep -v -E '^(SET transaction_timeout|\\restrict |\\unrestrict )' \
  | psql -v ON_ERROR_STOP=1 "$(insta db url --group db)"
```

**Never** use `pg_restore` without `--exit-on-error` as the procedure. It reaches full fidelity on a
clean schema, but "ignore all errors" equally swallows every blocker below.

**Hard blockers: PG17/18 constructs that cannot be filtered.** If any appears, the restore stops
there and the schema needs reworking by hand. Escalate to the user with the specific construct
rather than improvising a rewrite.

| Construct | Introduced | Error |
|---|---|---|
| `CREATE COLLATION … provider = builtin` | 17 | `unrecognized collation provider: builtin`, then cascading "collation does not exist" |
| Virtual generated column | 18 | dumped without `STORED`/`VIRTUAL`, so `syntax error at or near ")"` |
| `NOT NULL … NO INHERIT` | 18 | `syntax error at or near "NO"` |
| `ADD CONSTRAINT … NOT NULL … NOT VALID` | 18 | `syntax error at or near "NOT"` |
| `PRIMARY KEY (id, valid_at WITHOUT OVERLAPS)` | 18 | `syntax error at or near "WITHOUT"` |
| `FOREIGN KEY (…, PERIOD valid_at)` | 18 | `syntax error at or near "valid_at"` |
| `CHECK (…) NOT ENFORCED` | 18 | `syntax error at or near "ENFORCED"` |
| `DEFAULT uuidv7()` | 18 | `function uuidv7() does not exist` |
| `JSON_TABLE(…)` in a view | 17 | `syntax error at or near "AS"` |
| `now() AT LOCAL` in a view | 17 | `syntax error at or near "LOCAL"` |
| `random(1, 10)` | 17 | `function random(integer, integer) does not exist` |
| `xmltext(…)` | 17 | `function xmltext(text) does not exist` |
| An extension the target lacks | any | `extension "…" is not available` |

**Two failures that restore with exit 0 and break later.** These are the dangerous ones, because
every guard above passes.

1. **Named `NOT NULL` constraints (PG18).** `c text CONSTRAINT c_must_exist NOT NULL` restores
   clean, and `attnotnull` is set so enforcement survives, but pg16 records **no `pg_constraint`
   row**, so the constraint name is silently gone. A later migration doing
   `ALTER TABLE … DROP CONSTRAINT c_must_exist` will fail on the migrated database only.
2. **PG17/18 SQL inside function bodies.** `pg_dump` emits `SET check_function_bodies = false`, so
   plpgsql bodies are never parsed during a restore. `MERGE … RETURNING` and `RETURNING OLD.*`
   restore silently and fail at call time (`syntax error at or near "RETURNING"`,
   `missing FROM-clause entry for table "old"`). **A clean restore proves nothing about functions.
   Call every one of them once** as part of step 4.

Also expect a catalog difference that is **not** a fidelity loss: PG18 materializes `NOT NULL` as
`contype='n'` rows in `pg_constraint` and pg16 has none, so exclude those rows when diffing
catalogs, after confirming `attnotnull` is set on every column.

**The target must be empty.** A full dump restored into a populated database is not an incremental
sync: it collides on existing objects and primary keys. **Prefer adding a fresh postgres service**
over dropping the database. `DROP DATABASE` needs a DSN retargeted to `/postgres`, is blocked by
insta's own `pg_cron` session until you `pg_terminate_backend` it, and the recreated database
**loses the platform's preinstalled extensions** (`pgcrypto`, `uuid-ossp`, `pgaudit`, `vector`,
`pg_stat_monitor`, `pg_stat_statements`, leaving only `plpgsql`). If you do add a fresh service the
DSN changes, so see step 5.

Which guard catches what: **`ON_ERROR_STOP=1` catches SQL errors** (psql is the last stage, so its
status is the pipeline's), **`pipefail` catches a `pg_dump` failure**. You need both.
*Pass:* `grep -c '^ERROR'` is 0 **and** step 4's fidelity checks match. Exit 0 alone proves nothing.
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

**After a major-version downgrade, add the schema checks**, because that is where a downgrade loses
things quietly:

```bash
psql "$T" -c "select conname, contype, convalidated from pg_constraint
              where connamespace = 'public'::regnamespace and contype <> 'n' order by conname"
psql "$T" -c "select indexname, indexdef from pg_indexes
              where schemaname = 'public' order by indexname"
psql "$T" -c "select oid::regprocedure from pg_proc where pronamespace = 'public'::regnamespace"
```

Exclude `contype = 'n'` rows, since PG18 records `NOT NULL` there and pg16 does not. Diff `indexdef`
as text, and confirm `convalidated` is true rather than merely that the constraint exists.
Then **call every function the third query lists, once.** Their bodies were never parsed during the
restore, so this is the only thing that catches PG17/18 SQL inside them.
*Pass:* counts and latest rows match; sequences at or above the source's; extension sets reconciled;
after a downgrade, constraints validated, `indexdef`s equal, and every function callable.

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

**Now apply the pre-flight finding**, because the host finally exists. Read it off the service row
and set it into the name the app actually reads — its own name, never ours; the app has no idea
`INSTA_*` exists:

```bash
insta services list                                   # the compute row's host column
insta secrets set RENDER_EXTERNAL_HOSTNAME <that host>   # or DJANGO_ALLOWED_HOSTS, or whatever it reads
insta compute restart <service>                       # env is materialized at deploy time
```

Set only what the app needs. Faking a *second* variable to make it believe it is still on the old
platform is how the Render case turns a 400 into a 500 (the ladder in the Render section). Treat
this as an expedient that gets the cutover serving, and open a follow-up to give the app a neutral
way to read its host, since the value you just set is named after a platform it has left.

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

| Need | Heroku | Render | InstaCloud |
|---|---|---|---|
| dump all env | `heroku config -s` | dashboard, or read `render.yaml` | `insta secrets --print` |
| set one env | `heroku config:set K=V` | dashboard | `insta secrets set K` (value on stdin) |
| DB connection string | `heroku config:get DATABASE_URL` | dashboard only — `render postgres get` does NOT expose it | `insta db url` |
| psql session | `heroku pg:psql` | `render psql <id> --command "…" -o json --confirm` (only non-interactive form) | `insta db connect` |
| one-off task | `heroku run <cmd>` | `render jobs create` | `insta compute exec [service] -- <cmd>` (argv, no shell) |
| stop traffic | `heroku maintenance:on` | no switch — scale to zero or suspend, per service | `insta compute stop [service]` |
| scale | `heroku ps:scale web=2` | dashboard only — no CLI command | `insta services scale compute <name> 2` |
| custom domain | `heroku domains:add` | dashboard only — no CLI command | `insta compute set-domain <host> --group <svc>` |
| logs | `heroku logs -t` | `render logs` | `insta logs compute` (target is required) |

## Addon → service

| Source | Provision | Bound as |
|---|---|---|
| Heroku / Railway Postgres | `insta services add postgres <n>` | `DATABASE_URL` |
| Heroku / Railway Redis, **Render Key Value** (`render kv`) | `insta services add redis <n>` | `REDIS_URL` |
| JawsDB, PlanetScale | `insta services add mysql <n>` | `MYSQL_URL` |
| MongoDB Atlas | `insta services add mongodb <n>` | `MONGODB_URL` |
| S3 bucket, Railway bucket | `insta services add storage <n>` | `AWS_*`, `BUCKET_NAME` |
| Heroku Scheduler, Railway cron | none — see the table above | |

Bind every credential the app needs; nothing is auto-injected into compute.

## Per-source deltas

**Render.** Buildpack-built, so almost never a Dockerfile — `insta compute connect-repo` is the
shortest path. **Its Postgres is 18, so step 3 is a downgrade** into insta's pg16. Step 3 has the tested
procedure for that; it is one filtered line, not a blocker.

**Translate `render.yaml` yourself — there is no importer, and you do not need one.** If the repo
has one, read it and provision from this table rather than interviewing the user. Every row is a
command you already have:

| In `render.yaml` | Do this |
|---|---|
| `databases: [{name: X}]` | `insta services add postgres X` |
| `services: [{type: web, name: X}]` | `insta services add compute X --port <n>` |
| `type: worker` | a second compute service; give it a port and leave always-on (step 6 of this file's worker notes) |
| `type: cron` | no equivalent: `pg_cron`, or a scheduler inside an always-on compute service |
| `type: pserv` (private service) | a compute service, but **flag it to the user**: `insta services add` assigns a default domain to every compute service, so a Render private service stops being unreachable from the internet |
| `runtime: python` / `node` / `ruby` / `go` (any non-`image`) | `insta compute connect-repo <owner/repo> X` — nixpacks does what the buildpack did |
| `buildCommand:` | **nixpacks does not run the script**, but do not assume nothing in it happens: its Django provider runs `manage.py migrate` itself at start (measured — a full `admin, auth, contenttypes, sessions` migrate ran against the bound insta pg16 with no instruction from us). The **asset** half is what it skips, so read the script and re-home anything else: `collectstatic` or an `npm run build` needs a `Dockerfile` or nixpacks' own detected build step. A migration you want under your control rather than run at every boot belongs in `insta compute exec` |
| `startCommand:` | nixpacks picks its own, which is often not this one. If the app needs a specific server invocation (`gunicorn mysite.asgi:application -k uvicorn.workers.UvicornWorker`, a `-w` count, an ASGI vs WSGI entrypoint), that is a `Dockerfile` `CMD`, so this row can turn the whole service into the Dockerfile lane |
| `runtime: image`, `image.url` | `insta deploy --image <url> --port <n>` instead; do NOT reach for connect-repo |
| `envVars: [{fromDatabase: {...}}]` | `insta secrets bind DATABASE_URL postgres/X --to compute/Y` |
| `envVars: [{fromService: {...}}]` | usually a plain secret: only credential-minting services can be bound |
| `envVars: [{value: V}]` | `insta secrets set KEY V` |
| `envVars: [{generateValue: true}]` | Render invented it. **Carry the existing value over, do not regenerate** — for a Django `SECRET_KEY` a new one logs out every session, and for an app's own signing keys it invalidates issued tokens |
| `envVars: [{sync: false}]` | never in the file. Read it from the API below, or ask the user |
| `envVarGroups:` | **not returned by the env-vars API** (see below); resolve these from the dashboard |
| `disk: {mountPath, sizeGB}` | `--volume <gi>` on `insta services add`, or `insta compute volume X --size <gi>` later. It mounts at `/data` and only on the deploy *after* it is attached, so a different `mountPath` means a code or config change |
| `healthCheckPath` | not a knob here; insta health-checks the port |
| `numInstances` | `insta services scale compute X <n>` (1 to 10, same region, paid plans) |
| `plan:` | `insta compute limits` / `insta db limits` |
| `region:` | `--region` on `insta services add` (values from `insta regions`) |
| `autoDeploy: false` | nothing to do, and the default runs the other way for a **public** repo: `connect-repo --public` prints `deploys are manual from here: pushes will not redeploy (public repo)`, so nothing auto-deploys until you ask. `builds.auto_deploy` is not implemented on the compute plane either |

**Check for platform-detection env vars before you deploy anything.** Apps routinely branch on
whether the *source platform's own* variable is present, and every one of those branches flips when
the app lands here. The idiom to grep for is a bare presence test on the platform name:

```bash
grep -rnE "RENDER|DYNO|HEROKU|RAILWAY|FLY_APP_NAME|FLY_ALLOC_ID|VERCEL" --include='*.py' \
  --include='*.js' --include='*.ts' --include='*.rb' --include='*.go' .
```

`render-examples/django` is the worked example, and it fails **both** ways:

```python
DEBUG = 'RENDER' not in os.environ            # no RENDER here, so DEBUG becomes True
ALLOWED_HOSTS = []
RENDER_EXTERNAL_HOSTNAME = os.environ.get('RENDER_EXTERNAL_HOSTNAME')
if RENDER_EXTERNAL_HOSTNAME: ALLOWED_HOSTS.append(RENDER_EXTERNAL_HOSTNAME)
```

`ALLOWED_HOSTS` stays empty, so Django answers **HTTP 400 `DisallowedHost` to every request** on the
insta domain. **The platform reports the service as healthy while this happens**, because the check
is TCP on the port (`adapters/fly.ts`: `config.checks = { port: { type: 'tcp' } }`) and the app is
listening — it just refuses every request. `insta compute status` looking fine proves nothing; curl
the URL.

All three outcomes below were **measured end to end** on this repo (prod, insta-compute, 2026-09-09),
after `connect-repo` built it with nixpacks and the `DATABASE_URL` binding worked:

| what you set | result |
|---|---|
| nothing | **400** `DisallowedHost` on every request |
| `RENDER_EXTERNAL_HOSTNAME=<insta domain>` **only** | **200**, the page serves |
| that **plus** `RENDER=1` | **500** |

Read the ladder before copying the middle row. It works because `DEBUG` keys off `RENDER`, which
stays unset, so the host list gets its entry while the manifest static-files backend never switches
on. Adding `RENDER=1` flips `DEBUG=False`, which activates that backend, whose manifest the
`collectstatic` in `buildCommand` was supposed to build — hence the 500. **So the middle row leaves
the app serving with `DEBUG=True`, which leaks tracebacks and is not an end state.** Use it to get a
cutover answering, then fix it properly: give the app its own way to set `ALLOWED_HOSTS` and `DEBUG`
from env instead of impersonating the platform it left, and re-home the static build per the
`buildCommand` row.

**Every source hits this, but the shape differs, and Render is the mildest case.** Read from each
platform's own official Django example, which is what real user code is derived from:

| source | what its example does | what you get here |
|---|---|---|
| **Render** | `ALLOWED_HOSTS` appended from `RENDER_EXTERNAL_HOSTNAME` | 400, and **one env var fixes it** (the ladder above) |
| **Heroku** | `IS_HEROKU_APP = "DYNO" in os.environ`; then `["*"]` if set, else `[".localhost", "127.0.0.1", "[::1]", "0.0.0.0", "[::]"]` | 400, and **no env var can fix it** — both branches are literals, so the code must change. `DEBUG` keys off `ENVIRONMENT`, not the platform, so at least it stays off |
| **Fly** | hardcoded `['localhost', '127.0.0.1', '.fly.dev']` (their guide names no Fly variable) | 400, **code must change**. Do not go looking for `FLY_APP_NAME` in the settings; it is usually not there |
| **Railway** | `ALLOWED_HOSTS = ["*"]`, unconditional | **works as-is** — they bought that by giving up the check entirely |

So the useful expectation is not "grep for the platform variable" but **"assume the app cannot name
its own new hostname, and find out how it learns one."** Sometimes that is a variable you can set,
often it is a literal you have to edit, and occasionally (Railway) there is nothing to do.

**And there is nothing on this side for it to read.** The only variable the platform injects into a
compute service is `PORT` (`provisioning/deploy.ts`: `const env = { PORT: String(port), ...envBundle }`)
— everything else in the machine's env came from a secret or a binding you created. There is no
insta equivalent of `RENDER_EXTERNAL_HOSTNAME`, `RAILWAY_PUBLIC_DOMAIN` or `FLY_APP_NAME`, so an app
cannot discover its own public domain here. **Read the domain off `insta services list` and set it
explicitly** into whatever name the app reads. Do not wait for the app to work it out.

Note this problem belongs to the *pair* of platforms, not to the target: an app leaving Fly for
Railway breaks the same way (`.fly.dev` is a literal in Fly's own example, and Railway serves it at
`*.up.railway.app`), and Railway's own Fly and Render guides do not mention it either. Nobody
documents this, so do not expect the source platform's migration docs to have warned the user.

A quieter cousin: a config helper with a **fallback default** hides a failed binding instead of
reporting it. `dj_database_url.config(default='postgresql://…@localhost:5432/…')` means a missing
`DATABASE_URL` degrades to localhost, so a bind you forgot looks like a network fault. Confirm the
value on the machine (step 5) rather than inferring it from the app's behaviour.

**Env var values come from the API, not the CLI.** The Render CLI has **no** env-var subcommand at
all (`deploys`, `jobs`, `keyvalues`, `logs`, `postgres`, `restart`, `services`, `workflows`,
`workspaces`, `blueprints`, `environments`, `projects`, plus auth and session commands — that is the
whole surface). The REST API does return values:

```bash
curl -s -H "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/services/$SVC/env-vars" | jq -r '.[] | "\(.envVar.key)"'
```

Each item carries `key` **and** `value`, so this is how a `generateValue` or `sync: false` secret is
recovered without the dashboard. Two limits: it returns only vars belonging **directly** to the
service, so an `envVarGroups` member is invisible here, and the user has to mint the API key
(Dashboard → Account Settings → API Keys) because there is no CLI login that yields one. **Ask for
that key at the start**, not after provisioning. Print keys only; never echo a value into the
transcript.

CLI shape, measured rather than read off the docs: `render services -o json --confirm` returns
services **and** databases together; `render psql <id> --command "…" -o json --confirm` is the
**only** non-interactive query path; `render postgres get` does **not** expose a connection string
(dashboard only); `render jobs create` covers one-offs and `render logs` / `render restart` /
`render deploys` exist, but **scaling and custom domains have no CLI command at all**. There is no
maintenance-mode switch, so step 2 means scaling each service to zero or suspending it by hand.
Render Key Value (`render kv`) is the Redis equivalent. A **free** Postgres carries an `expiresAt`
30 days out, is capped at 1 GB, defaults its `ipAllowList` to `0.0.0.0/0`, and has **no backups and
no logical exports** — the connection string is the only way data leaves.

**Heroku.** The richest export surface: `config -s` yields `KEY=value` lines, `pg:backups` and
`maintenance:on` are single commands, and the `Procfile`'s `web:` / `worker:` map straight onto
compute services. No volumes. `app.json`, if present, declares the addons — read it to enumerate
what to provision.

**Railway.** Closest model (services + variables + IaC), so the concept mapping is nearly 1:1 — but
the export has three traps, all measured:

- **`railway variable list` always RESOLVES references**, in both the table and `--json`, and no flag
  shows the raw form. You will never see a `${{…}}`. The hazard runs the other way: a resolved
  `DATABASE_URL` is a literal pointing at **Railway's** Postgres, so copying it verbatim leaves the
  migrated app talking to the database you are leaving. Skip every connection string you are
  binding. The raw form exists only via `railway api` with `variables(… unrendered: true)`, and that
  query returns a **smaller** key set — the `RAILWAY_*` built-ins exist only at render time and are
  not stored variables worth migrating.
- **`railway status` reflects only LIVE deployments.** A stopped Postgres whose volume still holds
  data is indistinguishable from one never provisioned (`latestDeployment: null` for both). Check
  `railway deployment list` per service before concluding a database is unused.
- **A volume cannot be read while its service is stopped** — no offline browse; `render`-style file
  listing refuses with "has no active deployment", so auditing one means starting the service.

**Ask for a project token before you start.** `railway link` and `railway service` are interactive
pickers, and you cannot answer a picker. `RAILWAY_TOKEN` is project-scoped (Project Settings →
Tokens) and `RAILWAY_API_TOKEN` is account-scoped; take the **project** one for a single migration.
Also note `railway link` writes the **global** `~/.railway/config.json` keyed by cwd, not a local
file, so "cd somewhere safe" is not isolation.

**Translate the project yourself.** There is no `render.yaml` equivalent declaring the services:
`railway.json` carries only build and deploy config, and the services live in the project, so read
`railway status --json` for the shape and `railway variable list` per service for the env.

| On Railway | Do this |
|---|---|
| a service, `builder: RAILPACK` or `NIXPACKS` | `insta services add compute X --port <n>`, then `insta compute connect-repo <owner/repo> X` |
| a service built from a Dockerfile | same, `connect-repo` builds the Dockerfile when there is one |
| a service deployed from an image | `insta deploy --image <url> --port <n>` |
| the Postgres service | `insta services add postgres X` |
| Redis / MySQL / MongoDB services | `insta services add redis\|mysql\|mongodb X` |
| `deploy.startCommand` running migrations | do NOT carry it over as a startup gate; run migrations with `insta compute exec` (see SKILL.md) |
| `${{Postgres.DATABASE_URL}}` and friends | `insta secrets bind DATABASE_URL postgres/X --to compute/Y` |
| an app reading `PGHOST` / `PGUSER` / `PGPASSWORD` / `PGDATABASE` / `PGPORT` | a code change to read `DATABASE_URL`, per step 1 above. Railway injects these by default, so expect it |
| `RAILWAY_*` built-ins, `PORT` | skip: render-time only, and the platform supplies `PORT` here |
| any other variable | `insta secrets set KEY` |
| a volume | `--volume <gi>` on `insta services add`, or `insta compute volume X --size <gi>`; mounts at `/data` on the **next** deploy, and download the source contents while its service still runs |
| `numReplicas` | `insta services scale compute X <n>` (1 to 10, same region, paid plans) |
| a cron service | no equivalent: `pg_cron`, or a scheduler inside an always-on compute service |
| multi-region replicas | not available; one region per service, chosen with `--region` at add time |

Railway's Postgres template is **18**, so step 3 is a downgrade. Its volumes carry the same caveat
as any: creating a target volume does not copy contents.

**Fly.** The easiest source of the four, and the only one that is not a Postgres downgrade: Fly
Managed Postgres runs **16**, the same major as insta's, so step 3 needs no filter. A Fly app also
already has a `Dockerfile` and a `fly.toml`, so `insta deploy . --port <n>` works directly and
`internal_port` in `fly.toml` is the `--port` value. `[processes]` maps onto compute services, and
volumes carry the same caveat as any. **The one real obstacle is secrets:** `fly secrets list`
returns names and digests only, because "the actual value of the secret is only available to the
application", so there is no export. Read them off a running machine with
`fly ssh console -C env -a <app>` before you stop it, or have the user re-enter them.

> **Verified as of 2026-09-09**, by executing this runbook against a throwaway project with a seeded
> Postgres: the cutover ordering, the guard behaviour in step 3, `start` not re-resolving env, the
> `secrets set` stdin/argument asymmetry, and the refusal messages quoted above. **Not verified:** any
> end-to-end migration from a real source platform, the portless-worker path, and object-storage or
> volume data movement.
>
> **The pg18→pg16 downgrade in step 3 is verified end to end** against a seeded PG 18.6 source and a
> real InstaCloud PG 16.15 target: restore exited 0 with empty stderr, and a catalog and data diff
> came back identical apart from the extensions insta preinstalls. Sequences kept their positions
> (identity sequence at 900001, next insert returned 900002), 2 FKs and 4 CHECKs were `convalidated`
> and actually rejected violating rows, all 8 index `indexdef`s were byte equal including a GIN on
> jsonb and a partial index, view and trigger definitions were md5 equal, and
> `COPY … WITH (FORMAT binary)` from both sides was `cmp` identical at 123,405 bytes, covering
> microsecond `timestamptz` and nested `jsonb`. The blocker table and the two silent failures in
> step 3 were each reproduced individually.
