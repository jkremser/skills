# Prepare live resource patches

Before asking the user to authorize application, prepare and server-dry-run the
concrete changes being offered. For large reports, offer a manageable page or
requested subset and prepare that subset first.

Fetch each candidate from the chosen context/namespace as JSON. Capture its UID,
resourceVersion, container names, images, and current resource settings in the
private working directory. Compare identity/template with the handoff baseline
when recorded, and compare known current quantities after normalizing units.
Follow the shared handoff rules for absent historical identity. If the workload
has been recreated or rolled out, or its relevant settings changed, refresh the
recommendation through its chosen source instead of applying stale values.

Inspect applicable HPA/VPA ownership, GitOps/operator management, update strategy,
and namespace resource constraints. Disclose any reconciliation conflict or
rollout effect relevant to the proposed change. Keep HPA suppression intact;
changing utilization-based requests can change scaling behavior. Do not modify
autoscalers or management configuration to make a patch stick.

Map recommendation targets to these API resources and container paths:

| Recommendation kind | kubectl resource | Container array |
| --- | --- | --- |
| `Deployment` | `deployment` | `/spec/template/spec/containers` |
| `StatefulSet` | `statefulset` | `/spec/template/spec/containers` |
| `DaemonSet` | `daemonset` | `/spec/template/spec/containers` |
| `Standalone Pod` / standalone `Pod` | `pod` | `/spec/containers`, via the `resize` subresource |

For controllers, patch the Pod template so future Pods retain the settings.
Resolve a managed Pod to its controller with evidence that the recommendation
covers that controller; do not broaden a single-Pod recommendation to every
replica automatically. For a standalone Pod, verify it has no owner references
and is not a static/mirror Pod. Do not extend this workflow to Jobs, CronJobs, or
arbitrary custom resources.

Create one JSON Patch file per workload, grouping its selected container changes:

1. Start with `test` operations on `/metadata/uid` and
   `/metadata/resourceVersion` using the captured values.
2. Locate each container by name in the captured object; test the name at its
   resolved array index before editing it.
3. Copy its existing `resources` object, then merge only the offered CPU/memory
   request/limit leaves. Preserve unselected CPU limits, sidecars,
   ephemeral-storage, extended resources, and all other settings.
4. Use `add` at the container's `/resources` path with the merged object. JSON
   Patch `add` also replaces an existing object and handles absent resources.
   Keep requests within limits; if partial selection violates that constraint,
   explain the coupled change needed rather than silently changing another field.

With `RECOMMENDATIONS_RESOURCE`, `RECOMMENDATIONS_NAME`, and
`RECOMMENDATIONS_PATCH` set to the concrete resource, name, and patch file,
validate a controller candidate:

```sh
kubectl --context "$RECOMMENDATIONS_CONTEXT" --namespace "$RECOMMENDATIONS_NAMESPACE" \
  patch "$RECOMMENDATIONS_RESOURCE" "$RECOMMENDATIONS_NAME" \
  --type=json --patch-file "$RECOMMENDATIONS_PATCH" --dry-run=server -o json
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
