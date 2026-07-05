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
insta metrics db · insta logs db      # provider-limited: returns a note, not series
```

insta-oss: metrics/logs return a clear "cloud-only / coming" 501 today — use `docker logs`/`docker
stats` on the branch's containers directly if you must, and don't retry the CLI command.

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
   env, bad image arch.
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
