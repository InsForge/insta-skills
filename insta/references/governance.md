# Governance & audit

**InstaCloud's gates are enforcement, not etiquette.** On most platforms, agent safety is a
convention the agent is asked to follow; here the control plane refuses the action until a human
approves — an agent that ignores its instructions still can't get past the gate. Work *with* this
system; never around it.

## The gates

Every sensitive action passes a per-project policy check at the credential boundary:

| Action | Default | Guards |
| --- | --- | --- |
| `project.delete` | **approve** | destroying every resource |
| `secrets.read` | allow | the credential bundle leaving the platform |
| `secrets.write` | allow | user-secret changes |
| `deploy` | allow | code reaching compute (and the build-token mint) |
| `branch.delete` | allow | tearing down an environment |
| `service.add/remove/scale/upgrade` | allow | resource mutations (scale/upgrade: paid plans) |

Decisions: `allow` (proceed) · `deny` (hard no) · `approve` (human in the loop).

```bash
insta policy get --json
insta policy set <action> <decision>     # admin decision — propose it, don't assume it
```

## The approval flow (relay procedure — CRITICAL)

A gated action returns **"approval required" + an approval id** (HTTP 202; the action did NOT run):

1. **Relay to the human immediately and verbatim**: the exact line, e.g.
   `insta approvals approve 7c3c9b68-… ` (`--always` also flips the policy to allow permanently).
   Don't summarize it away, don't retry in a loop, don't report failure without surfacing it.
2. Only an **admin** can approve (`insta approvals list --status pending` shows what's waiting).
3. Grants are **single-use**: after approval, **re-run the original command**. The next occurrence
   prompts again unless the policy was set to allow.
4. `deny` policy = a hard no: report it and stop. Working around a gate (editing state, bypassing
   the CLI) is never acceptable — the gate is the product's safety model.

## The audit timeline

```bash
insta events [--branch <b>] [--limit <n>] [--json]
```

One per-project timeline containing: resource side-effects (creates, deploys + URLs, deletes),
every govern decision (pending/approved/denied, policy changes), and ingested agent findings.
Use it to answer "what happened to this project and who allowed it" — e.g. after any incident,
before deleting anything, or when a human asks what an agent did.

## The observe hook (credential audit for YOUR tool calls)

Auto-installed on `project create`/`link` (PostToolUse hook for Claude Code / Codex):

- Scans each tool call for credential exposure — AWS / GitHub / Stripe / LLM / DB URLs / JWTs /
  private keys — and appends **redacted fingerprints** (never raw secrets) to `./.insta/audit.jsonl`.
- `insta observe report [--json]` — review locally. `insta observe sync` — upload findings into
  the project timeline (idempotent, deduped).
- Agent etiquette on top of the hook: treat `./.env` as the only credential source; never print
  secret values into chat, logs, code, or commits; if the report shows a leak finding, surface it
  to the human rather than burying it.

## Patterns for agents

- **Before destructive work** (`project delete`, `branch delete` of someone else's branch): check
  `insta events` for recent activity and say what will be destroyed when relaying the approval.
- **Batch your gates:** if a workflow will hit the same gate repeatedly (e.g. many deploys under
  `deploy: approve`), tell the human once and suggest `approve --always` or a policy change,
  instead of interrupting N times.
- **After approval, verify:** the grant being consumed shows up in `insta events` — confirm the
  re-run actually happened before reporting the task complete.
