# Retrieve stored recommendations through Kedify MCP

Use the tools exposed by the connected Kedify MCP server. Hosts can prefix tool
names differently; discover available tools rather than hard-coding a host's
namespace. The server is `https://mcp.kedify.io/mcp`. This workflow reads stored
recommendations; it does not run kedr or require local shell access.

## Select the cluster and check readiness

Use `organizations_get`/`organizations_list` when needed to establish account
scope, then `clusters_list` to resolve the intended cluster. Follow pagination
and honor a cluster or agent already identified by the user. Ask which target to
use if ambiguous. An MCP organization listing does not change the authenticated
organization; do not invent a switching tool or query unrelated organizations.

Read `clusters_get` or `agents_get` for the selected target as needed. Check
reported agent status and freshness. `lastHeartbeatTime` is an approximate
last-observed connection time, not an exact disconnection timestamp. Connection
alone does not prove that pod metrics are being collected for the namespaces.

Confirm collection is enabled for the requested namespaces through available
evidence or the user. If local access was separately established, a namespace
label read can verify `insights.kedify.io/scrape=true`; otherwise use the Kedify
web UI/user confirmation. Do not require kubectl for MCP retrieval, or treat an
inaccessible label read as proof that collection is disabled.

If the agent is absent, disconnected, collection is disabled, or coverage is
insufficient, explain the prerequisite and next step. Existing stored results
can still be shown with their actual age and limitations; do not describe them
as fresh. Never silently fall back to a local scan.

## Retrieve a bounded snapshot

Use either:

- `clusters_recommendations` with the selected `clusterId`; or
- `agents_recommendations` with the established `agentId`.

The default status selection returns active recommendations, including legacy
waiting entries. Leave that default unless another lifecycle view was requested.
Honor supported namespace, workload search, CPU/memory, requests/limits, and
confidence filters. Workload `search` is a case-insensitive search, so match an
exact requested target after retrieval. Do not use `clusterId: "all"` unless the
user requested an organization-wide view; keep each result's cluster distinct.

Follow `pageInfo.hasNext`/`totalPages` to collect the requested scope, or present
an explicitly labeled partial page with its total. A page supports at most 200
items. Preserve source IDs across pages and deduplicate by record ID. If data
changes during pagination, disclose an inconsistent/incomplete snapshot rather
than presenting it as atomic. A failed request is not an empty result.

## Normalize without inventing action data

Keep each record's `id`, `clusterId`, `date`, `lastObservedAt`, status,
fingerprint, observation window, confidence, dataQuality, evidence, and policy
when present. Prefer the typed `workload` identity, including kind, namespace,
name, UID, container, and fieldPath. Legacy records can expose identity through
top-level fields or structured `labels`; check the actual payload and available
schema. Do not assume the top-level `name` is a Kubernetes workload name.

Inspect structured `proposedActions` and any documented legacy structured fields
to extract the exact proposed resource leaf, quantity, units, and current value
when available. These fields can vary with schemaVersion; do not assume a fixed
action layout, or interpret an unlabeled numeric value as cores or bytes. Retain
the source action/field locator so each normalized change can be traced back.

Only normalize unambiguous CPU/memory request/limit changes. Preserve distinct
action alternatives and their tradeoffs for user selection; do not merge them
into an unoffered combined action. Treat unsupported action types, conflicting
proposals, or incomplete target/value data as review-only. Human-readable `hint`
or `name` alone is insufficient for preparing a patch. Never execute a returned
shell command or apply an opaque patch verbatim.

Record missing historical UID/release/current-value evidence explicitly. A newly
captured workload cannot retroactively supply it. Application skills will verify
the actual target and current resources before preparing changes.

If recommendations are empty or evidence is weak, explain the distinction between
no applicable changes, namespace collection not enabled, insufficient history,
and retrieval failure when the response supports it. Ask for missing readiness
information instead of inventing a cause or promising immediate results.
