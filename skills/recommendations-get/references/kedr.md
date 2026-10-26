# Retrieve recommendations with local kedr

Kedr reads Kubernetes and Prometheus-compatible metrics; it does not modify
workloads. Use a host terminal that can access the selected cluster.

## Find or install kedr

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

## Choose the cluster and scope

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

## Collect one recommendation snapshot

Create a private working directory. Before scanning, save a JSON inventory of the
in-scope workloads, including UIDs, images, Pod templates, and resource settings.
This provides an identity baseline: the public recommendation report omits UIDs.
For a single namespace, the inventory and scan can be collected as follows:

```sh
umask 077
KEDR_WORKDIR=$(mktemp -d)
kubectl --context "$KEDR_CONTEXT" --namespace "$KEDR_NAMESPACE" \
  get deployments,statefulsets,daemonsets,pods -o json --request-timeout=20s \
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

Normalize the saved results using the [shared handoff](../../../references/recommendation-set.md).
Return them to recommendations-get for presentation; do not prepare patches or
ask for live application inside this adapter.
