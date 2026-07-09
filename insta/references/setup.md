# Setup

From zero to a linked project — CLI install, target selection, auth, project + services.

## Install / upgrade the CLI

```bash
curl -fsSL https://raw.githubusercontent.com/InsForge/insta-cli/main/install.sh | sh  # native binary, no Node
npm install -g insta            # npm alternative · one-shot: npx insta@latest <cmd>
insta upgrade                   # self-update (auto-update is on by default pre-1.0)
insta autoupdate off            # opt out of auto-update
```

Misbehaving or unrecognized command → update first (re-run the installer — it's idempotent — or
`npm update -g insta`), then retry.

## Pick the target

| | InstaCloud (managed) | insta-oss (self-hosted) |
| --- | --- | --- |
| endpoint | platform API (login persists it) | `INSTA_API_URL=http://127.0.0.1:8080` (its default) |
| auth | required (below) | none — localhost trust, builtin `local` user |
| daemon | n/a | `git clone InsForge/insta-oss && npm i && npx tsx src/main.ts` (needs Docker) |

`insta status --json` shows which target you're on (`apiUrl`) + login + linked project.

## Auth (cloud only)

- **Agents:** `insta login --email <e> --password <p>` (or `$INSTA_PASSWORD`), or an API token.
- **Humans:** `insta login --oauth github|google` — opens a browser (loopback capture). If the
  environment can't open one, relay any printed sign-in URL to the human **immediately and
  verbatim**; never sit on it silently.
- There is no login on insta-oss — don't try; `insta login` is a cloud-only command.

## Project

```bash
insta project create <name> [--org <id>]   # creates an EMPTY project (cloud) and links this dir
insta project link <id>                    # link an existing project instead
insta project list --org <id> --json
```

Linking writes `./.insta/project.json` (project + org + current branch, per directory) and
auto-installs the agent skills + the observe credential-audit hook into the repo.

**Naming:** use the directory/repo name for the project; things like `api` or `worker` are
*service* names, not project names.

## Services (the project's resources)

A cloud project starts **empty**; add what the app needs (insta-oss auto-provisions one
postgres + one storage at create):

```bash
insta services add postgres db        # relational DB (autoscaling)
insta services add storage files     # S3-compatible bucket
insta services add compute api       # your container; add more groups: worker, jobs, …
insta services list --json
```

Up to 5 services per type. Credentials are minted per service — `DATABASE_URL_<NAME>`,
`BUCKET_NAME_<NAME>`, … (name upper-snaked); the oldest of each type also gets the plain
unsuffixed names, so single-service projects use `DATABASE_URL` as usual.

## Ship-from-zero (the whole chain)

```bash
insta status                       # target + auth + link in one look
insta login …                      # cloud only, if needed
insta project create myapp
insta services add postgres db && insta services add compute app   # cloud; oss has db+storage already
insta secrets                      # writes ./.env for local dev
insta deploy . --port 8080         # or --image <ref>; then VERIFY the URL (see operate.md)
```
