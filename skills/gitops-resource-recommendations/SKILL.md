---
name: gitops-resource-recommendations
description: >-
  Run kedr against a live Kubernetes cluster and propose CPU and memory changes
  through the GitOps repository that manages it. Use for GitOps workload
  right-sizing with Argo CD or Flux, Helm values, Kustomize, or plain manifests.
  Covers installing kedr, choosing a context, matching live workloads to Git
  sources, letting the user select recommendations, preparing changes, creating
  an approved pull request, and optionally verifying delivery after merge.
  Targets Deployments, StatefulSets, and DaemonSets; use a direct-application
  workflow when the user explicitly wants live patches instead of Git changes.
---

# GitOps resource recommendations

Use `kedr simple` to measure live workloads, trace their configuration to Git,
and edit only the recommendations the user selects. Kedr reports recommendations;
it does not apply them. Delivery happens through the repository's existing
reconciliation or deployment pipeline after a pull request merges.

Use the host agent's shell, file editor, and conversation tools. This skill works
with Codex, Claude Code, and other agents with shell access; it requires no
agent-specific API, plugin, or companion skill. Examples use POSIX shell syntax;
adapt them to the host shell. Use Python, jq, or another available parser for JSON.

Keep three decisions separate: selecting recommendations authorizes local edits;
creating a PR requires the user's confirmation; waiting for merge and checking
delivery requires the user's answer after the PR exists. Honor authorization
already given for a decision. Never merge the PR as part of this workflow.
Do not patch live workloads, run `helm upgrade`, force a reconciliation, or change
sync policies to make these changes take effect. Keep cluster access read-only
apart from an optional API server dry-run or temporary metrics port-forward.

## 1. Establish the repository, tools, and cluster

Read applicable repository instructions. Inspect `git status --short`, the root,
remotes, branch, and deployment documentation. Preserve unrelated work. Start
with the current repository; ask for the owning repository only if source tracing
shows it is elsewhere. Keep reports and live inventories outside the Git tree:

```sh
umask 077
KEDR_WORKDIR=$(mktemp -d)
command -v kubectl
command -v kedr
```

Reuse a working kedr executable, including a trusted checkout's `bin/kedr`, and
record its path as `KEDR_BIN`. Check `"$KEDR_BIN" version` and
`"$KEDR_BIN" simple --help`. Use flags supported by that installation.
If kedr is missing, install it in a writable location:

- With Homebrew, try `brew install --cask kedify/tap/kedr`.
- Otherwise use the matching OS/architecture archive from the official
  [kedr releases](https://github.com/kedify/kedr/releases), verifying against that
  release's `checksums.txt` when available. Report if verification is unavailable.
- If no suitable release is available, build a trusted checkout, or clone
  `https://github.com/kedify/kedr.git` into a temporary directory. From its root,
  run `go build -o "$KEDR_BIN" ./cmd/kedr`, using an absolute writable destination
  and the Go version required by `go.mod`.

Check version and help after installation. Check Helm when chart discovery is
needed, and Kustomize or `kubectl kustomize` when overlays are involved. Argo/Flux
CLIs are optional: their Kubernetes custom resources can be read with kubectl.
Explain missing tools or permissions precisely instead of claiming completion.

Honor a context and scope already supplied by the user. Otherwise run
`kubectl config get-contexts` and ask which context to use **before contacting a
cluster**. Offer an optional namespace/workload scope in the same question. Never
silently use the current context or scan all contexts. If no contexts exist, ask
for the kubeconfig's location, without requesting pasted credentials.

Record `KEDR_CONTEXT` as the workload cluster, plus `KEDR_NAMESPACE` if scoped.
Pass `--context "$KEDR_CONTEXT"` to every workload-facing kubectl/kedr call and
`--kube-context "$KEDR_CONTEXT"` to Helm. Carry any explicit kubeconfig and
impersonation settings across tools using their supported flags. Do not change
the kubeconfig's current context. An Argo/Flux management cluster may differ:
establish its context separately with the user, and keep scans and final workload
verification pointed at the original workload cluster.

## 2. Inventory live workloads and identify the delivery system

Save an inventory **before** the scan, including UID, Pod template, images,
containers, resource fields, labels, annotations, and owner references:

```sh
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  get deployments,statefulsets,daemonsets -o json --request-timeout=20s \
  > "$KEDR_WORKDIR/workloads-before.json"
helm --kube-context "$KEDR_CONTEXT" ls -A -o json \
  > "$KEDR_WORKDIR/helm-releases.json"
kubectl --context "$KEDR_CONTEXT" api-resources -o name
```

For cluster-wide scope replace kubectl's namespace flag with `--all-namespaces`;
do not expand the scope of the kedr scan simply because release discovery uses
`-A`. Check command failures. A forbidden list is not an empty inventory.
[Helm lists](https://helm.sh/docs/helm/helm_list/) may be paginated; use the
installed version's `--max`/`--offset` options when needed.

Discover the relevant API groups, then read the installed controller's resources.
Fully qualified names avoid ambiguous short names such as `apps`:

```sh
# Argo CD, when its API is present (equivalent to kubectl get apps -A).
kubectl --context "$KEDR_CONTEXT" get applications.argoproj.io -A -o json

# Flux, when its APIs are present.
kubectl --context "$KEDR_CONTEXT" get helmreleases.helm.toolkit.fluxcd.io -A -o json
kubectl --context "$KEDR_CONTEXT" get kustomizations.kustomize.toolkit.fluxcd.io -A -o json
```

Use the established management context for these reads if controllers run
elsewhere. Treat Argo and Flux as alternative owners for this environment;
follow the one actually managing the target, without introducing the other.
If discovery gives conflicting ownership, ask which source is authoritative.
Distinguish absent APIs, no objects, and missing RBAC. An absent HelmRelease does
not rule out Flux managing plain manifests through a Kustomization.

Search the Git checkout with `rg --files` and targeted `rg` queries for
`Application`, `ApplicationSet`, `HelmRelease`, Flux `Kustomization`, source
objects, `Chart.yaml`, `kustomization.yaml`, Helm values files, and workload names.
Inspect deployment workflows for repositories containing plain manifests or Helm
values without a visible controller. Files existing in Git alone do not prove
that merging will deploy them.

## 3. Run one simple recommendation snapshot

Kedr needs existing Prometheus-compatible CPU/memory and kube-state-metrics
ownership data. Let it discover the endpoint. If multiple endpoints are offered,
ask the user to choose and rerun with `--prometheus-url`. If discovery fails, ask
for an existing endpoint or use a context-bound port-forward. Do not install a
monitoring stack for this task. For centralized metrics, establish the cluster
label/value and use `--prometheus-label` with `--prometheus-cluster-label` to
isolate the selected cluster. Keep authentication material out of reports/PRs.

With namespace scope, run:

```sh
"$KEDR_BIN" simple \
  --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  --resource Deployment --resource StatefulSet --resource DaemonSet \
  --formatter json --logtostderr --no-color \
  > "$KEDR_WORKDIR/recommendations.json"
```

For cluster-wide scope omit `--namespace` from kedr. Add established kubeconfig,
endpoint, and selector options. `--selector` filters labels and `--resource`
filters kinds; neither selects an exact workload name. Filter named targets by
their full identity in the report. Do not include standalone Pods, Jobs, or
CronJobs in this workflow.

Check the exit status, parse JSON, and inspect `errors` even on success for partial
failures. Use `simple` defaults: it retains CPU limits, sizes CPU requests, and
sizes memory requests/limits subject to guards. Do not bypass evidence thresholds
or add `--allow-hpa` merely to obtain actionable rows.

Read `scans[]` using these public fields:

| Fields | Interpretation |
| --- | --- |
| `object.cluster`, `.namespace`, `.kind`, `.name`, `.container` | Target identity; bind it to the selected context |
| `object.allocations.requests` / `.limits` | Current CPU and memory |
| `recommended.requests.cpu.value` / `.memory.value` | Proposed requests |
| `recommended.limits.cpu.value` / `.memory.value` | Proposed limits |
| `suppressedResources`, `object.warnings`, `recommended.info`, `analysis` | Guards, diagnostics, and rationale |

CPU values in allocations/recommendations are **cores**; memory values are
**bytes**. For example, `0.25` CPU is `250m`, and `268435456` bytes is `256Mi`.
Raw analysis can use other units; do not apply its intermediate values.
`"?"` means unknown and `null` means unset; neither authorizes removing a field.
Offer only finite positive, unsuppressed changes. Preserve unchanged, unknown,
suppressed, and unselected fields. Normalize to valid Kubernetes quantities
without concealing rounding, and use those exact quantities in the proposal.

Give rows stable IDs from the saved scan's array index plus one, retaining gaps.
Persist IDs with scan time, context, full target, UID/images/template baseline,
exact old/new quantities, and source mapping. Do not generate IDs with a second
scan. Bind any `kedr explain` call to this saved run rather than a later `latest`.

## 4. Trace each actionable workload to its effective Git source

For each actionable candidate in the saved scan, build an evidence-backed mapping:

`context + namespace + kind + workload + container → owner/release → repository
+ tracked revision + source path → effective resource value or manifest field`.

Use tracking metadata, Helm release annotations, controller inventories, chart
identity/version, and rendered resource identity together. Similar names or
`app.kubernetes.io/instance` alone are insufficient: Argo may also use that label.
Confirm the destination cluster and namespace, accounting for namespace/name
transforms and release-name overrides. Resolve ambiguous mappings with the user;
unresolved rows can be reported but cannot become automatic edits.

### Argo CD

Follow the Application's `spec.source` or `spec.sources`, `repoURL`,
`targetRevision`, `path`/`chart`, destination, and tracking metadata. Check its
resource status and whether an ApplicationSet or parent app generates it; edit
the owning Git template/input rather than a generated Application.

Argo [renders Helm charts](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
without creating Helm releases, so an empty `helm ls` does not exclude Helm.
Resolve `helm.releaseName`, ordered `valueFiles`, inline `values`, `valuesObject`,
`parameters`, and `fileParameters`. Precedence is parameters over valuesObject
over values over valueFiles over chart defaults; later value files win.
Edit the effective environment-specific input so higher-priority settings cannot
mask the recommendation.

For [multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/),
follow `$values/...` or another `$ref/...` to the source with the matching `ref`;
the path is relative to that repository's root. Values may live alongside
manifests in the current repo even when the chart lives elsewhere. Check source
ordering when duplicate rendered resources override one another.

### Flux

Trace the [HelmRelease](https://fluxcd.io/flux/components/helm/helmreleases/)
through `spec.chart.spec.sourceRef` or `spec.chartRef` to its chart source and
version. Resolve `releaseName`, `targetNamespace`, `storageNamespace`, and remote
`kubeConfig`; release storage and workload namespaces may differ.

Inspect `spec.values`, `spec.valuesFrom`, chart `valuesFiles`, and post-renderers.
Later valuesFrom entries normally override earlier ones, followed by inline
values; `targetPath` has special overriding behavior. Follow `valuesKey` and
`targetPath` to the effective input. For ConfigMaps/Secrets generated from files,
edit their Git inputs. Preserve the repo's encryption workflow for secret values;
do not copy decrypted values into Git or the PR.

Trace the owning [Flux Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/)
and its `sourceRef` through the source object's URL/ref, `spec.path`, inventory,
patches, and `postBuild` substitutions. This identifies how a HelmRelease, values
ConfigMap, or plain workload actually reaches the cluster. Account for remote
destinations and generators rather than editing only the visible live object.

### Helm values, Kustomize, and plain manifests

For real Helm releases, match release name, namespace, chart version, annotations,
and rendered workload identity to the repo's release configuration. Inspect a
matched release locally when useful:

```sh
helm --kube-context "$KEDR_CONTEXT" --namespace "$KEDR_RELEASE_NAMESPACE" \
  get values "$KEDR_RELEASE" -o yaml
helm --kube-context "$KEDR_CONTEXT" --namespace "$KEDR_RELEASE_NAMESPACE" \
  get manifest "$KEDR_RELEASE"
```

Save these outputs privately: they may contain secrets.
[Computed values](https://helm.sh/docs/helm/helm_get_values/) from `get values
--all` are diagnostic evidence, not a replacement for the repo's override files.
Inspect the pinned chart's defaults, schema, and templates to locate each
container's resource keys. Charts may use `resources`, `controller.resources`,
subchart keys, or presets; never assume a universal path.

For [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/),
trace the deployed overlay through bases, components, patches, and generators.
Prefer an environment-specific override over a shared base. Match containers by
name; avoid replacing the entire containers list or depending on its order.
For plain YAML, find the exact document by kind, name, effective namespace, and
container; change `spec.template.spec.containers[].resources`.

Render the current source using the deployed chart version, release name,
namespace, values order, and relevant transforms. Compare its target identity,
images, and resources to live state, normalizing Kubernetes quantities. Explain
drift or pending Git changes before basing edits on them. If an operator owns the
workload, trace its supported resource configuration instead; do not guess a CR
field or edit generated controller YAML.

Check that the proposed PR's base branch is the revision actually consumed by
the deployment chain. Identify commit/tag pins, chart packaging or promotion,
disabled auto-sync, suspended reconciliation, ignored resource fields, and manual
CI gates. A change to a chart's source does not update a pinned published chart.
Explain any required follow-up and obtain scope for it; do not silently repoint
sources or promise automatic delivery when merging alone is insufficient.

## 5. Offer the recommendations and collect the user's selection

Show context, namespace scope, scan time, strategy, and a numbered table:

`ID | namespace | kind/name | container | CPU request old → new | CPU limit old →
new | memory request old → new | memory limit old → new | Git source/status`.

Explain relevant evidence limits, autoscaler ownership, LimitRange/ResourceQuota
constraints, and rollout effects. Mark unmapped, stale, suppressed, and otherwise
blocked rows clearly. An empty actionable set is a valid result.

Ask which recommendations to put into Git, accepting IDs, all displayed actionable
rows, a field/container subset such as "memory only for 3", or none. Resolve
ambiguity before editing; "all" covers only the displayed actionable rows.
Selection authorizes local preparation of that exact subset. If no changes are
selected, stop with the report. Do not ask about a PR before there is a selected,
concrete diff to review.

## 6. Edit the selected sources and validate the effective diff

Before editing, re-read each selected live workload and compare UID, images,
template, and resources with the scan baseline; also check the relevant Git
source. Refresh stale recommendations when identity, images, or sizing inputs
changed. A status-only update does not invalidate the scan. Obtain another
selection only if the proposed values or effects materially change.

Use a dedicated branch/worktree following repo conventions. Edit the effective
values, generated-source input, environment overlay, or plain manifest identified
above. Merge only selected CPU/memory leaves. Preserve sidecars, CPU limits
retained by `simple`, other resources, comments, and unrelated configuration.
If a shared value or template affects other workloads/clusters, isolate the
override; if isolation is impossible, show the expanded impact and ask before
including it. Keep requests within limits. Explain coupled changes needed for a
partial selection rather than silently changing an unselected field.

Render before and after using the same inputs and versions as the deployment
system. Use `helm template` plus `helm lint` for charts when applicable,
`kubectl kustomize` or `kustomize build` for overlays, and the repo's validation
commands for plain manifests. Include Argo value precedence/multiple sources,
Flux valuesFrom/post-renderers/substitutions, and any generators actually used.
Do not claim a generic Helm render validates a controller-specific pipeline.

Compare parsed rendered objects by kind, namespace, name, and container name.
Verify every selected target has exactly the offered resource changes, that
unselected fields/targets are preserved, and that no objects are unexpectedly
created or removed. Check other consumers of edited shared inputs. Do not commit
rendered output unless the repo treats it as the maintained deployment artifact.
An optional scoped `kubectl --context "$KEDR_CONTEXT" apply --dry-run=server`
of the rendered target can check API/admission behavior without persistence;
report missing RBAC, admission changes, or incomplete validation accurately.

Run required repo checks and `git diff --check`. Show the selected IDs, exact
files/values changed, rendered impact, validation results, and delivery path.
Keep raw scans, inventories, and decrypted Helm values out of commits and PRs.

## 7. Ask to create the PR, then publish it when confirmed

With the validated diff ready, ask: **"Create a pull request with these changes?"**
Reuse an explicit prior instruction to create it. If declined, leave the local
diff or patch for review and stop. Selection alone is not permission to push.

When confirmed, commit only task files, push the task branch to the intended
remote, and create the PR against the verified deployment branch using the
available forge CLI/API, such as `gh` or `glab`. Follow repository conventions;
do not force-push, include unrelated edits, or merge. For a multiline CLI body,
write a private temporary file and use `--body-file` when supported.

Include the context/scope, scan time and strategy, selected old/new values,
source-to-workload mapping, render/check results, rollout effects, and any delivery
gates. Link separate PRs when the established delivery chain spans repositories;
do not claim a single PR will update sources it does not control. If publication
is blocked by auth or tooling, report the concrete blocker and retain the prepared
diff; never invent a PR URL.

Once the PR exists, provide its URL and ask: **"Should I wait for this PR to merge
and verify that the selected changes reach the cluster?"** If the user declines,
finish with the PR URL and describe verification as pending. Keep the context,
IDs, quantities, mapping, and PR details available for resuming verification.

## 8. If requested, wait for merge and verify delivery

Use bounded polling with regular progress updates and respect any user deadline.
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
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  rollout status "$KEDR_RESOURCE/$KEDR_NAME" --timeout=120s
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
