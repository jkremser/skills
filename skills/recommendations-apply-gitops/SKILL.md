---
name: recommendations-apply-gitops
description: >-
  Apply existing Kubernetes CPU and memory recommendations to the owning GitOps
  repository and create a pull request. Use for delivery through Argo CD, Flux,
  Helm values, Kustomize, or plain manifests.
---

# Apply recommendations through GitOps

Turn recommendations obtained from MCP, kedr, or a supplied report into focused
repository changes and a pull request. Read the shared
[recommendation handoff](../../references/recommendation-set.md) to interpret
the input. If recommendations are missing, read and follow
[recommendations-get](../recommendations-get/SKILL.md), preserving the user's
source choice. Do not rerun retrieval merely because the source was MCP rather
than kedr.

Support Deployments, StatefulSets, and DaemonSets. Report standalone Pods, Jobs,
CronJobs, and unsupported actions as outside this application workflow. Do not
patch live workloads, run `helm upgrade`, force reconciliation, or change sync
policies. Delivery follows the repository's established pipeline after merge.

## 1. Ask for the configuration repository and establish access

Ask which Git repository configures the target cluster, and for its URL or local
checkout path, unless already identified. Do not assume the current working
directory is the owning repository. Use the supplied checkout or obtain a working
copy with available repository tools. Read its instructions and deployment docs;
inspect its status, remotes, branch, and deployment paths, preserving unrelated
work. Keep raw reports and inventories in a private directory outside Git.

Use local shell/editor/forge tools or connected repository tools appropriate to
the host. Check Helm/Kustomize only when the actual source needs them. Missing
rendering tools or cluster access must be reported as validation limits, not
invented successful checks. If the host cannot access or edit the repository,
explain the missing capability and retain the recommendation handoff.

When local cluster access is needed, reuse the already-established workload
context. Otherwise list `kubectl config get-contexts` and ask which context maps
to the recommendation's cluster before contacting it. Keep that mapping as
`RECOMMENDATIONS_CONTEXT`, with `RECOMMENDATIONS_NAMESPACE` for scoped calls.
Pass `--context` to kubectl and `--kube-context` to Helm; carry explicit kubeconfig
and impersonation settings using supported flags. Do not change current-context.
Establish a separate management context if Argo/Flux runs in another cluster.

## 2. Identify the effective source and deployment path

Use available live inventory and repository configuration to establish ownership.
With cluster access, capture the target controllers as JSON, including UID,
images, templates, resources, labels/annotations, and owner references. Discover
installed APIs before querying controller resources:

```sh
kubectl --context "$RECOMMENDATIONS_CONTEXT" api-resources -o name
# Use the established management context instead if appropriate.
kubectl --context "$RECOMMENDATIONS_CONTEXT" get applications.argoproj.io -A -o json
kubectl --context "$RECOMMENDATIONS_CONTEXT" get helmreleases.helm.toolkit.fluxcd.io -A -o json
kubectl --context "$RECOMMENDATIONS_CONTEXT" get kustomizations.kustomize.toolkit.fluxcd.io -A -o json
```

Run only queries for installed/relevant APIs. Distinguish missing RBAC from absent
APIs or empty inventories. List real Helm releases when relevant, honoring
pagination; Argo can render Helm without creating a Helm release. Discovery must
not broaden the selected recommendation scope.

Search Git for Applications/ApplicationSets, HelmReleases, Flux Kustomizations and
sources, charts/values, overlays, workload names, and deployment workflows. Read
[source mapping](references/source-mapping.md) for the relevant Argo, Flux,
Helm, Kustomize, or plain-manifest details. Consult only the applicable sections.

Map every candidate to the exact repository/revision/path and effective container
resource input. Resolve cluster destination, namespace/name transforms, value
precedence, generated inputs, and pinned versions. Similar names alone do not
prove ownership. Without live access, use trustworthy snapshot and repository
evidence; mark unresolved identity, historical baseline, or delivery mapping as
blocked rather than guessing. Merely finding YAML in Git does not prove that
merging it deploys to this cluster.

## 3. Collect the selection and prepare the concrete diff

Show the saved IDs, cluster, observation time, current → proposed quantities,
container targets, and mapped source paths. Explain evidence limits, autoscaler
ownership, namespace resource constraints, and rollout effects relevant to these
changes. Ask which recommendations to put into Git unless the user already chose
them. Accept IDs, a field/container subset, all displayed actionable rows, or none.
Do not offer blocked rows as executable changes.

Selection authorizes preparation of that exact subset. Re-read the chosen
workloads/source files and compare known identity, release, template, and current
values with the handoff before editing. Refresh stale sizing evidence through
the selected retrieval source; obtain another selection only for a materially
changed proposal. If live verification is unavailable, state that limitation and
resolve any uncertainty about applicability before preparing executable edits.

Use a dedicated branch/worktree following repository conventions. Edit the
effective values, generated-source input, environment overlay, or plain manifest.
Merge only selected CPU/memory leaves; preserve sidecars, other resources,
comments, and unrelated configuration. If a shared value affects additional
workloads or clusters, isolate the override or ask about the expanded scope.
Keep requests within limits; explain coupled changes required by a partial
selection instead of silently adding an unselected change.

## 4. Validate and show the result

Render before/after with the deployed versions and effective inputs:

- Helm: use `helm template` and relevant `helm lint` checks with the actual
  release name, namespace, ordered values, and controller overrides.
- Kustomize: use `kubectl kustomize` or `kustomize build` on the deployed overlay,
  including the generators/substitutions used in delivery.
- Plain manifests: follow the repository's established validation commands.

Account for Argo multiple sources and value precedence, Flux valuesFrom,
post-renderers/substitutions, and any generated files actually involved. Compare
parsed rendered objects by kind, namespace, name, and container name. Verify the
selected targets have exactly the offered changes, all unselected fields and
targets are preserved, and no objects are unexpectedly created or removed.
Check other consumers of shared inputs. Keep generated output out of Git unless
it is the repository's maintained deployment artifact.

An optional context-bound API server dry-run can check admission without
persistence. Report missing access, admission mutations, and incomplete validation
accurately. Run required repository checks and `git diff --check` when using a
checkout. Show the exact diff, selected IDs, rendered impact, validation results,
rollout effects, and any delivery gates before publication. Keep raw scans,
inventories, decrypted values, and secrets out of commits and PRs.

## 5. Create the pull request

Reuse an explicit instruction to create the PR. Otherwise ask **"Create a pull
request with these changes?"** once the concrete diff and validation are ready.
Selection alone does not authorize pushing. If declined, leave the prepared diff
or patch for review and stop.

With authorization, commit only task files, push the task branch to the intended
remote, and create the PR against the verified deployment branch using available
forge tools such as `gh`, `glab`, or a connected API. Do not force-push or merge.
For multiline CLI bodies, use a private temporary file and `--body-file` where
supported. Include source and observation time, cluster/scope, selected exact
old/new values, source mapping, checks and limitations, rollout effects, and
delivery gates. Link separate PRs if the established deployment chain requires
changes in multiple repositories. Report publication failures without inventing
a PR URL, and retain the prepared work.

Return the PR URL and describe cluster delivery as pending. If the user already
requested post-merge verification, continue with
[delivery verification](references/verify-delivery.md); otherwise ask whether
they want to wait for merge and verify delivery. Never merge or enable auto-merge
as part of this workflow. Preserve the snapshot and mapping for later verification.
