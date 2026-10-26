# Verify delivery after merge

Use bounded polling with regular progress updates and respect any user deadline.
This phase requires read access to the workload cluster and delivery system;
if unavailable, report the unverified stages and what access is missing.
Inspect PR state through the forge. Stop if it closes without merging; never
enable auto-merge or approve/merge it yourself. On timeout, inaccessible services,
or user cancellation, report the pending stage and enough detail to resume.

After merge, resolve the merge commit and inspect the merged diff. If reviewers
changed selected values, report that difference and verify the actual merged
intent. Check that the tracked deployment revision contains the change, then
follow the existing delivery chain:

- **Argo:** check the Application's observed source revision(s), sync status,
  operation/conditions, and health for the target destination.
- **Flux:** check source artifact revision, owning Kustomization's
  `lastAppliedRevision`/Ready state, and any HelmRelease's observed generation,
  Ready state, history, and effective configuration. A values-only change can
  retain the same chart version; chart version alone is not delivery evidence.
- **CI/plain manifests or Helm:** check the deployment job/artifact and its
  target cluster. A merge or green build alone does not prove deployment.

Treat manual sync, suspension, revision pins, and failed deployment jobs as
delivery blockers. Explain the next action without forcing sync, resuming a
controller, moving a pin, or manually deploying as an automatic fallback.

On the **original workload context**, read each Deployment/StatefulSet/DaemonSet
and compare selected container resources using quantity equivalence. Check
controller convergence with an appropriate timeout, for example:

```sh
kubectl --context "$RECOMMENDATIONS_CONTEXT" --namespace "$RECOMMENDATIONS_NAMESPACE" \
  rollout status "$RECOMMENDATIONS_RESOURCE/$RECOMMENDATIONS_NAME" --timeout=120s
```

Set the resource/name/namespace from each selected row. Inspect running Pods,
their owner/revision, resource settings, and readiness as well as controller
templates. Paused Deployments, StatefulSet partitions, `OnDelete` strategies,
zero replicas, or admission mutations can leave templates updated without all
running Pods using the intended resources. Report those outcomes; do not delete
Pods or restart workloads to force convergence.

Finish with the PR URL/merge revision, original context, selected IDs, effective
values, reconciliation and rollout evidence, and anything blocked or pending.
Distinguish Git merged, controller reconciled, template updated, and running Pods
verified. For failures, inspect relevant conditions/events and propose a focused
Git revert PR; do not automatically roll back live objects or unrelated changes.
