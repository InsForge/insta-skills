# InstaCloud workflow: branch → build → deploy → test → promote

Read this when developing in an InstaCloud project. For command syntax/flags and Dockerfile
templates, see `cli-reference.md`.

## Branching is the default unit of work

`insta branch create <name>` provisions a complete **isolated environment** at once:

- its own **Neon DB branch** (copy-on-write copy of the parent's data, own `DATABASE_URL`),
- its own **copy-on-write storage bucket** (forked from the parent at branch-create), and
- a **clone of every compute group** in the project — each is its own Fly app + URL.

Branches run **fully in parallel** — nothing one does touches another. (A project whose root bucket
was created before snapshots were enabled keeps one **shared** bucket — no per-branch storage isolation.)

Per-branch loop:

1. `insta branch create feat-x` — isolated DB + storage + compute. **Does NOT auto-switch you.**
2. `insta branch switch feat-x` — sets the current branch (per-directory, in `./.insta/project.json`).
3. `insta secrets` — writes feat-x's `DATABASE_URL`, `AWS_*` storage creds, etc. into `./.env`.
4. Build your image, then `insta deploy --image <registry/img>` — deploys to feat-x's compute; **the
   command prints feat-x's URL**.
5. **Test against that URL directly** — public HTTPS, no tunnel.

Non-default branches' compute **scales to zero when idle** (so parallel preview envs cost nothing
while unused); the default branch (`main`) stays always-on. Scale-to-zero is a cost lever, **not**
load-based horizontal scaling.

## Running multiple agents in parallel (encouraged)

Per-branch isolation makes parallel agent development the natural mode: each agent gets its **own git
worktree + its own insta branch**, so they build, deploy, and test concurrently with zero collision.

Bind each worktree to a branch **before** dispatching the agent:

```bash
git worktree add -b feat-x ../proj-feat-x main      # isolated CODE copy on its own git branch
cd ../proj-feat-x
insta branch create feat-x && insta branch switch feat-x && insta secrets   # isolated DB + storage + compute + creds
npm install                                                                 # fresh worktree has NO node_modules — install before the first build
```

> **Fresh-worktree gotcha:** `git worktree add` creates a clean checkout with **no `node_modules`**.
> The first build fails with `Module not found` until you `npm install` in the new worktree.

Then dispatch one subagent per worktree: work only in its dir, build, `insta deploy`, **capture the
printed URL and test against it**, open a PR. Mapping = **1:1:1 — task ↔ git worktree (code) ↔ insta
branch (data + compute)**. Create both layers.

*(Optional: if your agent has the **superpowers** skills, `superpowers:using-git-worktrees` and
`superpowers:dispatching-parallel-agents` go deeper on the worktree/subagent mechanics.)*

## Promote a branch to main (merge → migrate → redeploy → validate)

InstaCloud does **not** merge databases (diverged Postgres can't be safely 3-way merged). Promotion
happens in **git + migration replay**:

1. **Merge the code in git.** Expect **conflicts** when parallel branches edited the same files →
   resolve by **combining** (parallel features are usually additive). Migration *files* merge cleanly
   if uniquely named.
2. **Verify the merged code builds** before the slow remote deploy.
3. **Switch + redeploy main:** `insta branch switch main` → `insta secrets` → re-run the new
   migration files against **main's** DB → `insta deploy`. Branch DBs are never merged — the
   migration **files** carry the schema forward.
4. **Validate on main's URL** — promotion isn't done until the live result checks out.
5. `insta branch delete <name>` — tear down the promoted branch's env (Neon branch + forked bucket +
   the branch's Fly apps). If `branch.delete` is gated, this returns an approval request first.

## Gotchas (read before deploying)

- **Never gate container startup on migrations.** A `CMD` of `migrate && server` lets a hung/failed
  migration stop the server from ever starting — the deploy "succeeds" but the URL times out with
  nothing in the logs. Run migrations **non-blocking**: `timeout 30 <migrate> || echo skipped; <start-server>`.
- **`--port` MUST equal the port your container listens on.** Fly routes external traffic to that
  internal port; a mismatch boots fine but every request fails with `instance refused connection`.
  Bind to `0.0.0.0`, never `127.0.0.1`.
- **Secrets are injected at deploy** as env vars (decrypted from the branch scope). Read creds from
  `process.env` in production; **never bake `./.env` into the image** (it's local-dev only).
- **A gated action returns "approval required".** Have an admin run `insta approvals approve <id>`,
  then **re-run** the action — grants are single-use.
