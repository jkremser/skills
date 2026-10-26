# Trace recommendations to their effective Git source

For each actionable candidate in the supplied recommendation snapshot, build an
evidence-backed mapping:

`cluster + namespace + kind + workload + container → owner/release → repository
+ tracked revision + source path → effective resource value or manifest field`.

Include a verified local workload context when available. Without live access,
use the handoff and authoritative repository/deployment evidence, and identify
any checks that could not be performed.

Use tracking metadata, Helm release annotations, controller inventories, chart
identity/version, and rendered resource identity together. Similar names or
`app.kubernetes.io/instance` alone are insufficient: Argo may also use that label.
Confirm the destination cluster and namespace, accounting for namespace/name
transforms and release-name overrides. Resolve ambiguous mappings with the user;
unresolved rows can be reported but cannot become automatic edits.

## Argo CD

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

## Flux

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

## Helm values, Kustomize, and plain manifests

For real Helm releases, match release name, namespace, chart version, annotations,
and rendered workload identity to the repo's release configuration. Inspect a
matched release locally when useful:

```sh
helm --kube-context "$RECOMMENDATIONS_CONTEXT" --namespace "$RECOMMENDATIONS_RELEASE_NAMESPACE" \
  get values "$RECOMMENDATIONS_RELEASE" -o yaml
helm --kube-context "$RECOMMENDATIONS_CONTEXT" --namespace "$RECOMMENDATIONS_RELEASE_NAMESPACE" \
  get manifest "$RECOMMENDATIONS_RELEASE"
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

