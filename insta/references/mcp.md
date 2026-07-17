# The remote MCP server (`insta-cloud`)

InstaCloud's control plane is also exposed as a **remote MCP server** — Streamable HTTP at
`https://mcp.instacloud.com/mcp` — so MCP-capable agents (Claude Code, Claude.ai / ChatGPT
connectors, Cursor) can drive projects with native tool calls instead of shelling out to the CLI.
It is a stateless bridge over the same platform API the CLI uses: same auth, same governance
gates, same audit trail.

## When to use which

**Prefer the MCP tools when they're connected** — structured JSON results, no PATH/install
concerns, and hosted agents (Claude.ai / ChatGPT) that have no shell can still operate InstaCloud.
**The CLI remains required** for exactly four things a remote server cannot do:

| Capability | Why CLI-only |
|---|---|
| `insta login` / auth / API-token CRUD | credential minting is deliberately not a remote tool |
| `insta secrets` (pull values → `.env`) / `insta run` | secret **values** never flow out of MCP — names only, values in |
| `insta deploy <dir>` (source builds) | needs a local build context; `insta_deploy` takes prebuilt image URLs only |
| `insta observe` hook / `insta setup` | local-machine operations |

## Connecting

`insta setup agent` registers the server with Claude Code automatically (user scope). The default
is **OAuth — no credential is written**: registration is just

```bash
claude mcp add --transport http --scope user insta-cloud https://mcp.instacloud.com/mcp
```

and on first `/mcp` use Claude discovers the platform's authorization server (RFC 9728 → Better
Auth MCP plugin, dynamic client registration) and runs the browser flow — managed, expiring,
revocable tokens, nothing static on disk.

**Headless machines / CI** (no browser): `insta setup agent --mcp-token` instead mints a durable
`insta_` API token named `mcp-<hostname>` (needs `insta login` first) and registers with an
`Authorization: Bearer` header. Manual setup for other clients works the same way: OAuth if the
client supports MCP OAuth discovery, else a Bearer header with any `insta_` API token.
`INSTA_MCP_URL` points setup at a different deployment (beta/self-host).

New/renamed tools need a **fresh agent session** to appear — reconnecting an existing session
won't pick them up.

## Tool ↔ CLI mapping

Naming is `insta_<noun>_<verb>`; every tool takes **explicit `projectId` / `branch` args** — the
server is stateless, there is no "current project" like `./.insta/project.json`. Get the
`projectId` from `insta_project_list` (or `.insta/project.json` if you're in a linked repo).

| CLI | MCP tool |
|---|---|
| `insta status` (am I connected?) | `insta_whoami` |
| `insta org list` / `create` | `insta_org_list` / `insta_org_create` |
| `insta project list/create/delete` | `insta_project_list` / `insta_project_create` / `insta_project_get` / `insta_project_delete` |
| `insta services add/list/remove` [`--branch`] | `insta_service_add` / `insta_service_list` / `insta_service_remove` (all take `branch?`; add takes `public?` for storage) |
| services public/private toggle | `insta_service_access` |
| `insta services scale/upgrade` | `insta_service_scale` / `insta_service_upgrade` |
| `insta compute start\|stop\|suspend` / `status` | `insta_compute_control` / `insta_compute_status` |
| `insta compute set-domain/check-domain/remove-domain` | `insta_domain_set` / `insta_domain_check` / `insta_domain_remove` |
| `insta branch create/list/merge/delete` | `insta_branch_create` / `insta_branch_list` / `insta_branch_merge` / `insta_branch_delete` |
| `insta manifest` | `insta_manifest` (env view — **no secret values**) |
| `insta secrets list/set/unset` | `insta_secrets_list` (names only) / `insta_secrets_set` / `insta_secrets_unset` |
| `insta deploy --image <url>` | `insta_deploy` (image-only) |
| `insta metrics/logs/events` | `insta_metrics` / `insta_logs` / `insta_events` |
| `insta usage` / `billing` | `insta_usage` / `insta_billing_summary` |
| `insta billing upgrade/portal` | `insta_billing_checkout` / `insta_billing_portal` — return a Stripe **URL for the human**; relay it, never claim payment happened |
| `insta govern …` (policy/approvals) | `insta_policy_get` / `insta_approvals_list` / `insta_approvals_approve` / `insta_approvals_deny` |

## Behavior that carries over from the CLI

- **Governance is identical.** Gated tools return `approval_required` + an `approvalId` instead of
  an error — run the same approval relay you'd run for the CLI (tell the human, wait, retry).
  Never treat `approval_required` as failure.
- **Custom-domain results include DNS records** for the developer's own registrar — relay them
  verbatim, like the CLI's printed table.
- Paid-tier gates (`scale`, `upgrade`) and branch caps (≤10) are enforced by the platform and
  surface as structured errors, same as the CLI.
