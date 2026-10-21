---
name: live-resource-recommendations
description: >-
  Use kedr to recommend CPU and memory requests and limits for live Kubernetes
  workloads, then apply the recommendations the user selects. Use for live
  workload right-sizing, resource optimization by running kedr against a Kubernetes cluster.
  Covers installing kedr, choosing a Kubernetes context, and updating Deployments,
  StatefulSets, DaemonSets, and standalone Pods.
---

# Live resource recommendations

Run `kedr simple`, explain the results, and apply only the changes the user chooses.
Kedr reads Kubernetes and Prometheus-compatible metrics; it does **not** modify
workloads. Use `kubectl` for application.

Use the host agent's terminal and conversation tools. These instructions work with
Codex, Claude Code, and other agents with shell access; no agent-specific API or
plugin is required. Shell examples use POSIX syntax; adapt them to the host shell.
Use an available JSON parser, such as Python or jq, for reports and patches.

## 1. Find or install kedr

Check for `kubectl`, kubeconfig access, and a working kedr executable:

```sh
command -v kubectl
command -v kedr
```

Reuse an existing kedr binary, including a trusted checkout's `bin/kedr`. Store its
path as `KEDR_BIN`. Run `"$KEDR_BIN" version` and `"$KEDR_BIN" simple --help` to check
the installed interface; use its supported flags rather than inventing commands
such as `kedr apply` or flags such as `--output json`.

If kedr is missing, install it locally using an appropriate available method:

- With Homebrew: `brew install --cask kedify/tap/kedr`.
- Otherwise, download the matching OS/architecture archive from the official
  [kedify/kedr releases](https://github.com/kedify/kedr/releases), verify it against
  that release's `checksums.txt`, and put the executable in a user-writable bin
  directory. Use its full path if that directory is outside `PATH`.
- If binaries are unavailable, build a trusted existing checkout or clone
  `https://github.com/kedify/kedr.git` into a temporary directory. From the checkout
  root, run `go build -o "$KEDR_BIN" ./cmd/kedr`, with `KEDR_BIN` set to an absolute,
  writable destination. Use the Go version and dependencies required by `go.mod`.

Do not replace a working installation unnecessarily. Verify version and help after
installation. If a required tool or download is unavailable, explain the specific
blocker; do not pretend to have run a recommendation.

## 2. Choose the cluster and scope

Honor any context, kubeconfig, namespace, or workload already specified by the
user. Otherwise, list the available contexts and mark the current one:

```sh
kubectl config get-contexts
```

Ask which context to use before contacting a cluster if none was specified. Bundle
an optional namespace/workload question with that choice. Do not silently choose
the current context or scan every context. If none exist, ask for a kubeconfig;
never request pasted credentials.

Keep the chosen values as `KEDR_CONTEXT` and, when scoped to one namespace,
`KEDR_NAMESPACE`. Pass `--context "$KEDR_CONTEXT"` to **every** cluster-facing
kubectl and kedr invocation. Honor the same explicit `--kubeconfig` and
impersonation options in both tools when provided. Do not run `kubectl config
use-context` or change kubeconfig defaults.

Verify access with a bounded read, for example:

```sh
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  get deployments,statefulsets,daemonsets,pods --request-timeout=15s
```

Use the supplied namespace. If only a workload name was supplied, resolve its
namespace and kind through scoped reads. For an unrestricted cluster scan, omit
`--namespace` from kedr and explicitly label the scope as cluster-wide. Match a
named target exactly when processing results: kedr's `--selector` filters workload
labels, and `--resource` filters kinds, not workload names.

Kedr needs an existing Prometheus-compatible source with CPU/memory usage and
kube-state-metrics ownership data. Let it discover endpoints first. If it finds
multiple endpoints, show the candidates and ask which to use, then rerun with
`--prometheus-url "$KEDR_PROMETHEUS_URL"`. This also avoids a terminal selection
prompt when the agent's shell is noninteractive. If discovery fails, ask for an
existing endpoint; a context-bound port-forward to an existing service is an
option. Do not install a monitoring stack as part of a recommendation request.

For centralized metrics, confirm the cluster label and value and use
`--prometheus-label` with `--prometheus-cluster-label` to isolate this cluster.
Use existing authentication facilities; keep tokens out of conversation and
artifacts. Follow the installed CLI's authentication and TLS options.

## 3. Collect one recommendation snapshot

Create a private working directory. Before scanning, save a JSON inventory of the
in-scope workloads, including UIDs, images, Pod templates, and resource settings.
This provides an identity baseline: the public recommendation report omits UIDs.
For a single namespace, the inventory and scan can be collected as follows:

```sh
umask 077
KEDR_WORKDIR=$(mktemp -d)
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  get deployments,statefulsets,daemonsets,pods -o json \
  > "$KEDR_WORKDIR/inventory-before.json"
"$KEDR_BIN" simple \
  --context "$KEDR_CONTEXT" \
  --namespace "$KEDR_NAMESPACE" \
  --resource Deployment --resource StatefulSet --resource DaemonSet --resource Pod \
  --formatter json --logtostderr --no-color \
  > "$KEDR_WORKDIR/recommendations.json"
```

Adapt namespace, label, kubeconfig, and endpoint flags to the established scope;
use `--all-namespaces` for a cluster-wide kubectl inventory, and omit the namespace
flag for a cluster-wide kedr scan. Stop if the inventory read fails.
`--resource Pod` means standalone Pods; their report kind is `Standalone Pod`.
Do not obtain a separate table scan to generate IDs: observations and results can
change between scans. Verify exit status, parse the JSON, and inspect `errors`
even after exit code zero, because a report can contain partial failures.

Use the default `simple` strategy unless the user requests another strategy.
It sizes CPU requests while retaining CPU limits, and sizes memory requests and
limits subject to its guards. Defaults include a 48-hour query window, one hour
of observed sizing history, and 30 distinct observations. Missing metrics or
insufficient history are reasons to explain and wait or repair the metrics path;
do not lower evidence requirements or enable `--allow-hpa` just to produce values.

Read each `scans[]` entry as follows:

| Field | Meaning |
| --- | --- |
| `object.cluster`, `.namespace`, `.kind`, `.name`, `.container` | Full target identity, with the chosen context as cluster identity |
| `object.allocations.requests` / `.limits` | Current CPU and memory values |
| `recommended.requests.cpu.value` / `.memory.value` | Recommended requests |
| `recommended.limits.cpu.value` / `.memory.value` | Recommended limits |
| `suppressedResources`, `object.warnings`, `recommended.info`, `analysis` | Suppression, evidence, and explanation |

CPU numbers in allocations and `recommended` are **cores**; memory numbers are
**bytes**. Raw `analysis` uses its own units (including CPU millicores); use the
public `recommended` fields for application and honor `suppressedResources`.
For example, `0.25` CPU is `250m`, and `268435456` bytes is `256Mi`.

`"?"` means unknown, and `null` means unset. Neither is a numeric recommendation
or permission to remove a field. Offer only finite, positive, unsuppressed values
that change the current allocation. Preserve existing fields for unknown,
unchanged, suppressed, or unselected values. Use exact quantities for patches;
human-readable rounding must not change the value offered for application. If
Kubernetes quantity precision requires normalization, show and validate that
normalized quantity before offering it for selection.

Assign IDs from the original array index plus one, retaining gaps after filtering.
Persist the mapping to context, namespace, kind, workload, container, exact old/new
values, and scan time. IDs belong to this snapshot. If using `kedr explain`, bind
it to this scan's saved run and row; do not assume a later `latest` run matches.

## 4. Prepare changes for review

Before asking the user to authorize application, prepare and server-dry-run the
concrete changes being offered. For large reports, offer a manageable page or
requested subset and prepare that subset first.

Fetch each candidate from the chosen context/namespace as JSON. Capture its UID,
resourceVersion, container names, images, and current resource settings in the
private working directory. Compare identity and template with the pre-scan
inventory, and resource values with the scan after normalizing units. If the workload is new, has been
recreated or rolled out, or its relevant settings changed, refresh its baseline
and recommendation instead of applying stale values.

Inspect applicable HPA/VPA ownership, GitOps/operator management, update strategy,
and namespace resource constraints. Disclose any reconciliation conflict or
rollout effect relevant to the proposed change. Keep HPA suppression intact;
changing utilization-based requests can change scaling behavior. Do not modify
autoscalers or management configuration to make a patch stick.

Map report targets to these API resources and container paths:

| Report kind | kubectl resource | Container array |
| --- | --- | --- |
| `Deployment` | `deployment` | `/spec/template/spec/containers` |
| `StatefulSet` | `statefulset` | `/spec/template/spec/containers` |
| `DaemonSet` | `daemonset` | `/spec/template/spec/containers` |
| `Standalone Pod` | `pod` | `/spec/containers`, via the `resize` subresource |

For controllers, patch the Pod template so future Pods retain the settings.
Resolve a managed Pod to its controller instead of patching an individual replica.
For a standalone Pod, verify it has no owner references and is not a static/mirror
Pod. Do not extend this workflow to Jobs, CronJobs, or arbitrary custom resources.

Create one JSON Patch file per workload, grouping its selected container changes:

1. Start with `test` operations on `/metadata/uid` and
   `/metadata/resourceVersion` using the captured values.
2. Locate each container by name in the captured object; test the name at its
   resolved array index before editing it.
3. Copy its existing `resources` object, then merge only the offered CPU/memory
   request/limit leaves. Preserve CPU limits retained by `simple`, sidecars,
   ephemeral-storage, extended resources, and all other settings.
4. Use `add` at the container's `/resources` path with the merged object. JSON
   Patch `add` also replaces an existing object and handles absent resources.
   Keep requests within limits; if partial selection violates that constraint,
   explain the coupled change needed rather than silently changing another field.

With `KEDR_RESOURCE`, `KEDR_NAME`, and `KEDR_PATCH` set to the concrete resource,
name, and patch file, validate a controller candidate:

```sh
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  patch "$KEDR_RESOURCE" "$KEDR_NAME" \
  --type=json --patch-file "$KEDR_PATCH" --dry-run=server -o json
```

Inspect the returned resources for admission changes and validation failures.
Show admitted quantities in the proposal; disclose any admission changes outside
the offered fields before asking for application.
A dry-run validates the API request; it does not prove the workload will remain
healthy. If denied by RBAC, present the recommendation and exact missing access.

For standalone Pods, use the same patch structure at `/spec/containers` and add
`--subresource=resize` to both dry-run and application. Check client/server support,
`pods/resize` patch permission, the Pod's QoS constraints, and `resizePolicy`.
In-place resizing may restart a container and may remain pending. If unsupported
or rejected, explain the blocker and offer a replacement manifest for review.
Deleting/recreating the Pod requires separate explicit authorization for that
disruption; never use `replace --force` as an automatic fallback. See the official
[container resize documentation](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/).

## 5. Offer recommendations and let the user choose

Present the chosen context, scope, strategy, scan time, and a numbered table with
namespace, kind/name, container, and current → proposed CPU request, CPU limit,
memory request, and memory limit. Include brief rationale, evidence limitations,
and the expected rollout or resize effect. Mark unavailable or blocked entries
clearly; an empty actionable set is a valid result.

Ask which IDs to **apply**, allowing choices such as `apply 2 and 5`, `apply all
shown`, `only memory for 3`, or `none`. Clarify that selecting an offered ID for
application authorizes its displayed changes on the named context. A request to
view recommendations alone does not authorize mutation. Resolve ambiguous choices
before writing; `all` refers only to the explicitly offered actionable entries.

If the user selects a subset of fields or containers, rebuild and dry-run exactly
that subset. Reuse existing authorization when the values and effects match what
the user selected. Ask again only if the resulting proposal materially changes
or needs additional coupled changes or disruption.

## 6. Apply the selected changes and verify

Apply only authorized patches, one workload at a time, with the same context,
namespace, JSON Patch preconditions, and Pod subresource used in the dry-run.
Run the validated command with `--dry-run=server` removed. The UID and
resourceVersion tests prevent applying the patch to a different or changed object.

If a precondition fails, re-fetch and compare. An unrelated status update may only
require rebuilding the precondition and repeating the dry-run; changed identity,
image, template, or resources requires a refreshed recommendation and review of
any materially changed proposal. Do not retry by removing concurrency checks.

After applying, read back the live resources and compare them with the selected
values. For a Deployment, StatefulSet, or DaemonSet, run a bounded check:

```sh
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  rollout status "$KEDR_RESOURCE/$KEDR_NAME" --timeout=120s
```

Inspect the update strategy: paused Deployments, StatefulSet partitions, and
`OnDelete` strategies can leave existing Pods unchanged. Report those cases
accurately; do not delete Pods or resume a rollout without authorization.
For ordinary rollouts, verify the new Pods' resource settings and readiness.
For a standalone Pod, check readiness, resize conditions, and the actual resources
reported in `status.containerStatuses`, not just the desired spec.

If a rollout or resize fails or remains pending, stop further applications and
inspect relevant conditions/events. Report partial completion and offer a focused
rollback using the captured original resource fields; do not automatically undo
other configuration or reapply a complete saved object. Keep the original field
presence so rollback can restore an initially unset request or limit.

Finish with the context, selected IDs, actual changed values, verification outcome,
anything skipped or pending, and paths to useful report/patch/rollback artifacts.
Distinguish API acceptance, template updates, and effective resources on running
Pods. If the user chose no changes, return the recommendations and stop.
