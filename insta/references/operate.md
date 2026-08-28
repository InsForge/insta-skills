# Operate — status, triage, and recovery

## Reading the environment

```bash
insta status --json        # target api · login · linked project · current branch
insta manifest --json      # every branch's db/storage/compute + URLs — the ground truth
insta services list --json # what the project has
insta events --limit 50    # what happened (resources + govern + agent findings)
```

`manifest` is the first stop whenever reality seems to disagree with expectations — it shows what
each branch *actually* has (including a legacy shared bucket, or a compute group that was never
deployed).

## Metrics & logs

```bash
insta metrics compute [group] [--branch --from --to --step --json]
insta logs compute [group] [--branch --limit --region --instance --json]
insta logs compute [group] --since 2h     # time window (--from/--to also accepted) — pages ~7 days of history
insta metrics redis|mysql|mongodb [group]   # managed DBs are Fly apps: same full metrics/logs
insta logs redis|mysql|mongodb [group] [--deploy]
insta metrics db · insta logs db      # postgres: provider-limited — returns a note, not series
```

insta-oss: metrics/logs return a clear "cloud-only / coming" 501 today — use `docker logs`/`docker
stats` on the branch's containers directly if you must, and don't retry the CLI command.

## Idle modes & what an app costs

**Billing is always by actual app usage** — vCPU·min burned, GB·min of RAM resident, storage,
egress — never by machine size × hours. The idle mode only changes what "idle" consumes:

- **Scale-to-zero (default)**: idle machines suspend and auto-wake on the next request. An idle
  service costs **nearly nothing**; the trade is a cold start (typically a few seconds) on the
  first request after idling.
- **Always-on (opt-in, all plans)**: machines never suspend, so there are **no cold starts** — but
  the idle app keeps its RAM resident (plus a trickle of vCPU), and that real usage bills
  continuously (roughly $1–2.50/month for an idle minimum-spec app, mostly RAM).

Flip it any time — it is a latency/cost dial, not a plan feature:

- `insta services add compute <name> --always-on` — create pinned-warm.
- `insta compute always-on on|off [service]` — toggle a live service.
- `insta db always-on on|off [--group <g>]` — the same dial for a postgres service:
  `off` (default) suspends the idle instance and cold-starts the first connection after idle;
  `on` keeps it warm.

## Resource ceilings (limits)

A service's size is **not** a price — billing is actual usage either way. It is a **ceiling**: the
most the app may burn, i.e. its blast radius. So it moves in both directions, and lowering one
costs the customer nothing.

```bash
insta compute limits                     # ceiling 1 vCPU / 256 MB  (plan max 2 vCPU / 2 GB)
insta compute limits --memory 1gb        # set it — cpu derives from memory
insta db limits --memory 8Gi --cpu 4     # same dial for postgres
```

- **Memory is the dial.** It is the ceiling that actually bites (hitting it OOM-kills the app);
  vCPU only throttles, so it is derived unless `--cpu` is passed for a parallel workload. On
  compute, setting always requires `--memory` (`--cpu` is an override, never valid alone); on db,
  either flag alone works. Decimal (`mb`/`gb`) and binary (`Mi`/`Gi`) suffixes are both accepted.
- **Paid plans.** Free services stay at the minimum ceiling — this is the one thing a plan gates,
  precisely because usage billing means the size is no longer what you pay for.
- **Compute ceilings snap to the provider's sizes** (vCPU comes from a fixed ladder; memory in
  256 MB steps within a per-vCPU band). A request that cannot be honored exactly is REJECTED with
  the legal value named — never silently rounded.
- Raising or lowering a compute ceiling **restarts the machine**; a postgres resize restarts the
  instance only if it is awake (a suspended one applies the new ceiling on its next wake).

`insta services upgrade` still exists but is the pre-usage-billing control for **compute**: named
specs, up-only. Prefer `limits`. For postgres it is not a fallback at all — an `upgrade` on a
postgres service is rejected outright; resize it with `insta db limits` and grow its disk with
`insta db volume`.

## Compute volumes

Compute persistent `/data` volumes are **not create-time only**:

```bash
insta services add compute app --volume 1Gi  # attach at creation
insta compute volume app --size 1Gi          # attach later if the service has no volume
insta compute volume app                     # view size, mount path, and plan cap
insta compute volume app --delete            # destroy the disk and all data
```

`--size` on a volumeless service attaches a volume, and it mounts on the **next deploy/redeploy**.
`--size` on an existing volume grows it only; volumes cannot shrink. Deleting is the only way off a
volume and is irreversible. A volume keeps machine count at 1 and changes scale-to-zero from suspend
to stop; those constraints lift after deletion.

## Pausing & resuming compute

To take a service **offline on purpose** — a maintenance window, cost control, or parking a
preview branch — use the lifecycle controls, which are a *persistent* override: a stopped/suspended
service will **not** be re-woken by incoming traffic (unlike scale-to-zero's auto-wake).

- `insta compute stop [service]` — clean shutdown; stays down until `start`.
- `insta compute suspend [service]` — snapshot RAM for a faster resume; stays down until `start`.
- `insta compute start [service]` — bring it back online and re-enable auto-wake.
- `insta compute status [service]` — desired (your intent) vs. live runtime state.

`[service]` is optional when the project has exactly one compute service. These work on all plans and
require no approval. A billing suspension is separate: you can't `start` while an org is billing-
suspended, and a manual `stop` is preserved across a billing pause/resume cycle. A billing
suspension force-stops always-on machines too — pinned-warm does not outlive the org's credit.

## Restarting a compute service

`insta compute restart [service]` re-runs the image the service **already** runs, against a freshly
resolved env bundle. Reach for it in exactly two situations:

1. **Config changed and the running app hasn't picked it up.** `insta secrets set`,
   `insta secrets bind` and `insta secrets unbind` all change what the app *would* receive, not what
   the running machine holds — env is baked into the machine at deploy time. `restart` is how that
   change lands without shipping a new version.
2. **The machine is up but wedged.** A crash-looped or hung process is still `started`, so
   `insta compute start` is a no-op on it — it only flips desired state and wakes a machine that is
   *down*. `restart` cycles it.

What it is **not**: a new deploy. It ships no new image and no new spec — the image, port and
resource ceiling are exactly the ones already recorded on the service.

Rules worth knowing before you call it:

- **The service must be running.** A deliberately stopped or suspended one is refused (400) and
  pointed at `insta compute start`, which is also what re-enables auto-wake. A restart would
  otherwise silently undo the persistent override `stop` gives you.
- **A service that has never been deployed** is refused the same way `exec` refuses it — deploy an
  image first.
- **It is health-gated on the way back up.** If the app doesn't answer on its port, the machines are
  rolled back to the config they were serving and the command reports the failure. That verdict is
  the useful part: a restart that "fails" here is telling you the app itself is broken, not the
  platform. The old config still serving is the safe outcome, not a second failure.
- All plans; ungated (no approval). It boots machines, so it bills as ordinary uptime and is refused
  while the org is billing-suspended — the same door `start` stands behind.
- WebSocket apps: the connections-based concurrency floor is set by `insta deploy --websocket` and is
  not recorded on the service, so a restart comes back without it — same as any `insta deploy --image`
  without the flag. Redeploy with `--websocket` instead of restarting for those.

## Running a one-shot command on a compute machine

`insta compute exec [service] -- <command> [args...]` runs a single command on the service's live
machine and returns — **no interactive shell, no stdin**. Use it for a one-off migration, a debug
`ls`/`cat`, or confirming a process is actually up.

- Targets **a single machine** — the first `started` machine, else the first live one. If a service
  has been scaled out to multiple machines, `exec` runs on exactly one of them, not all.
- A **scaled-to-zero machine wakes first** — adds a few seconds, and that wake time bills as normal
  compute uptime, same as any other request.
- `--timeout <sec>` bounds the run, **1–180s** (default 30); the remote command is killed if it
  doesn't finish in time.
- The CLI's **exit code is the remote command's exit code** — safe to check in a script (`&&`,
  `$?`). stdout/stderr stream to their own local streams verbatim; each is capped at **1 MiB**, with
  a truncation notice on stderr if hit. `--json` returns the raw response instead of split streams.
- Gated on **both** `deploy` and `secrets.read` — a deny on either is a 403; an approve on either
  triggers the usual relay (`insta approvals approve <id>`; see governance.md).
- A compute service with no image ever deployed 400s ("this service has no machines yet — deploy an
  image first, then retry") — `insta deploy` it, then retry.

`[service]` is optional under the same rule as `start`/`stop`/`status` above.

## Deploy triage (URL not serving after deploy)

Work the list in order — these cover ~all real failures seen so far:

1. **Port mismatch** (most common): `--port` ≠ the port the app listens on. Symptom: deploy
   "succeeds", every request refused/000. Fix: redeploy with the app's actual listen port; bind
   `0.0.0.0`.
2. **Cold start**: non-default branches suspend when idle — first request can take seconds. Poll
   up to ~60s before concluding failure.
3. **Migration-gated startup**: `CMD migrate && server` with a hung migration = nothing listening,
   empty logs. Fix the CMD to start the server regardless (see deploy.md).
4. **Read the logs**: `insta logs compute [group] --branch <b> --limit 100` — crash loops, missing
   env, bad image arch. A bare read is ONE provider page (~100 lines); when the failure is older
   than that, window it: `--since 2h`, or `--from <unix|ISO>` / `--to`.
5. **Stale CLI**: unrecognized command / odd 4xx → `insta upgrade` (or re-run the installer), retry.
6. **Gate, not failure**: a 202 "approval required" is not an error — relay it (governance.md).

## Failure-reporting discipline

Report the exact observed state (HTTP code, log line, gate id) — never an assumed one. A deploy
isn't "done" until the URL served; a promotion isn't "done" until main's URL validated; a teardown
isn't "done" until `manifest`/`events` reflect it.

## Cloud vs insta-oss behavior differences

| Surface | Cloud | insta-oss |
| --- | --- | --- |
| login | required | doesn't exist (localhost trust) |
| usage / billing | real (billing dimensions) | 501 — no billing locally |
| metrics / logs | served (compute full, db limited) | 501 today (docker-stats planned) |
| source deploy (`deploy <dir>`) | ✅ remote build | not yet — use `--image` |
| services add postgres/storage | ✅ (≤5 each) | 501 — one of each, auto-provisioned |
| branch compute | fresh empty app — deploy to it | parent's image auto-redeployed |
| branch app URL | own subdomain | host port +1000 |
