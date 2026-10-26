---
name: recommendations-get
description: >-
  Retrieve Kubernetes CPU and memory recommendations through Kedify MCP or a
  local kedr scan. Use when the user wants to inspect or collect recommendations.
---

# Get resource recommendations

Retrieve and explain one recommendation snapshot. Keep retrieval independent of
how the user may later apply it. Read the shared
[recommendation handoff](../../references/recommendation-set.md) before preparing
the result.

## 1. Ask which retrieval source to use

First ask whether the user wants **Kedify MCP** or **local kedr**, unless they
already explicitly chose a source. Explain the prerequisites with the choice:

| Source | Requirements and environment |
| --- | --- |
| Kedify MCP | A connected Kedify account, an installed and running Kedify agent in the target cluster, and pod metrics collection enabled for the relevant namespaces. The agent must have scraped metrics long enough to produce recommendations. Works without access to the user's local kubeconfig, including from a hosted UI with the MCP connection. |
| Local kedr | A terminal with access to the user's kubeconfig and Kubernetes API, plus an existing reachable Prometheus-compatible service with pod CPU/memory usage and kube-state-metrics (KSM) data. A ChatGPT web session without access to the user's machine/cluster cannot run this local workflow. |

Explain that namespace collection for MCP can be enabled in the Kedify web UI or
with the namespace label:

```sh
kubectl --context "$RECOMMENDATIONS_CONTEXT" \
  label ns "$RECOMMENDATIONS_NAMESPACE" insights.kedify.io/scrape=true
```

This is setup guidance, not a command to run during ordinary retrieval. Use an
explicitly chosen context and namespace if the user requests the setup change.
Enabling collection does not immediately create sufficient history. Report the
actual evidence/coverage limitations; do not invent a universal waiting period.

If the chosen method is unavailable, explain what is missing and offer the other
method. Obtain the user's choice before switching. Do not install an agent or
monitoring stack, or alter namespace labels, merely to obtain a result.

## 2. Retrieve through the chosen source

Read only the relevant adapter:

- **MCP:** [Kedify MCP retrieval](references/mcp.md). Select the target from the
  connected cluster inventory and retrieve stored recommendations. Local
  Kubernetes tools are unnecessary for this path.
- **Kedr:** [local kedr retrieval](references/kedr.md). Ask which Kubernetes
  context to use before contacting a cluster, honor an already-specified
  context, and bind the inventory and scan to it.

Honor any namespace, workload, resource, or evidence filters already supplied.
Preserve partial failures and empty results; neither is permission to expand the
scope, weaken evidence requirements, or manufacture values.

## 3. Present the snapshot and hand it off

Show the source, cluster identity, scope, retrieval/observation times, and a
numbered table containing namespace, kind/name, container, and exact current →
proposed CPU requests/limits and memory requests/limits. Include confidence and
evidence limitations when available. Mark unknown, suppressed, stale, unsupported,
or otherwise blocked rows clearly. An empty actionable set is a valid result.

Retain stable selection IDs, source records, exact quantities, and identity
evidence using the shared handoff. Save private artifacts when a filesystem is
available; otherwise keep the structured result in the conversation.

Finish with the recommendations when the user requested retrieval only. If they
also requested application, read and follow the selected application skill:

- [recommendations-apply-gitops](../recommendations-apply-gitops/SKILL.md) for
  repository changes and a pull request.
- [recommendations-apply-kubectl](../recommendations-apply-kubectl/SKILL.md) for
  direct live changes.

Carry the snapshot, selection, and existing authorization into that workflow.
If application was requested without a delivery method, ask which method to use.
