---
name: recommendations-apply-kubectl
description: >-
  Apply selected existing Kubernetes CPU and memory recommendations directly to
  live workloads with kubectl. Use when the user wants live resource patches.
---

# Apply recommendations with kubectl

Apply recommendations from MCP, kedr, or a supplied report to the chosen live
cluster. Read the shared [recommendation handoff](../../references/recommendation-set.md)
before interpreting the input. If recommendations are missing, read and follow
[recommendations-get](../recommendations-get/SKILL.md). Reuse an existing usable
snapshot without requiring kedr or Prometheus.

This workflow requires a terminal with kubectl, credentials, and network access
to the target cluster. Retrieving recommendations through MCP does not grant
that local access. In a hosted UI without those capabilities, retain the report,
explain the missing access, and offer commands/patch artifacts for the user to
run in an environment that has it. Do not claim live application from MCP alone.

## 1. Establish the target cluster

Honor a context already selected and verified against the recommendation's
cluster. Otherwise run `kubectl config get-contexts` and ask which context to use
before contacting a cluster. Resolve the mapping from MCP cluster identity when
needed; never silently use the current context or treat an MCP cluster name as
a context name.

Record `RECOMMENDATIONS_CONTEXT` and the namespace on each row. Bind every
cluster-facing command to `--context "$RECOMMENDATIONS_CONTEXT"` and the target
namespace. Carry any explicit kubeconfig and impersonation options, and do not
change kubeconfig defaults. Keep captures and patch files in a private working
directory, with `umask 077` where supported.

## 2. Prepare the offered changes and ask what to live-apply

Explain before selection: **if the cluster is managed through GitOps, direct
changes may be overwritten or lost when Argo CD, Flux, or another controller
reconciles.** Offer
[recommendations-apply-gitops](../recommendations-apply-gitops/SKILL.md) for
persistent repository changes if that is the user's preference. An explicit
choice to continue with live changes is sufficient; do not disable reconciliation
or modify management configuration to make patches stick.

Read [patch preparation](references/patching.md) to revalidate target identity,
resource baselines, ownership, namespace constraints, and supported update paths.
Support Deployments, StatefulSets, DaemonSets, and standalone Pods via the resize
subresource when supported. Keep unsupported kinds and ambiguous targets blocked.
Prepare and server-dry-run the concrete changes before asking for authorization;
for large snapshots, offer a manageable requested subset first.

Show context, observation time, stable IDs, namespace, kind/name, container, exact
current → proposed CPU/memory quantities, and rollout/resize effects. Mention
autoscaler conflicts, admission mutations, or other concrete blockers. Ask which
recommendations the user wants to **live-apply**, accepting IDs, all displayed
actionable rows, selected fields/containers, or none. A retrieval-only request
does not authorize mutations. Reuse a prior explicit selection/authorization for
the same proposal rather than asking again.

If the user chooses a field/container subset, rebuild and dry-run exactly that
subset. Keep requests within limits and preserve all unselected fields. Ask again
only if the concrete proposal needs additional coupled changes or materially
different values, scope, or disruption. If none are selected, return the report.

## 3. Apply the selected patches and verify

Apply one workload at a time using the validated JSON Patch, UID/resourceVersion
tests, context, namespace, and Pod subresource. Remove only `--dry-run=server`
from the validated command. Never apply an opaque source-supplied command/patch
or remove concurrency checks to make an update succeed.

On a precondition failure, re-fetch and compare. A status-only update can require
new preconditions and another dry-run; changed identity, images, template, or
resources requires refreshed sizing evidence and review of any materially
changed proposal. Do not retry against a changed workload automatically.

Read back the resources and compare with the selected exact quantities. For a
Deployment, StatefulSet, or DaemonSet, use a bounded rollout check, for example:

```sh
kubectl --context "$RECOMMENDATIONS_CONTEXT" --namespace "$RECOMMENDATIONS_NAMESPACE" \
  rollout status "$RECOMMENDATIONS_RESOURCE/$RECOMMENDATIONS_NAME" --timeout=120s
```

Set namespace/resource/name from each selected row. Inspect new Pods' resources,
owner/revision, and readiness. Paused Deployments, StatefulSet partitions,
`OnDelete` strategies, or zero replicas can leave templates updated without
existing Pods changing. Do not delete Pods or resume rollouts without scope for
that action. For standalone Pods, check readiness, resize conditions, and actual
resources in `status.containerStatuses`, not just desired spec.

If a rollout/resize fails or remains pending, stop further application, inspect
relevant conditions/events, and report partial completion. Offer a focused
rollback using captured original resource fields and field presence; do not
automatically restore a complete saved object or undo unrelated configuration.

Finish with context, source snapshot, selected IDs, actual changed quantities,
verification results, skipped/pending targets, and any useful patch/rollback
artifact paths. Distinguish API acceptance, template changes, and effective
resources on running Pods.
