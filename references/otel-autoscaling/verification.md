# Validate configuration and prove the metric path

## Render before deployment

Use the pinned chart and collector versions, actual release/namespace, ordered
values, and GitOps overrides. Inspect `helm show chart` and `helm show values`
for that version. Do not assume the README's example version is current or that
`opentelemetry-collector` is the correct parent values key: the inspected add-on
chart uses the `otelCollector` and `otelOperator` aliases.

Render before/after with `helm template` (or the repository's Kustomize/plain
manifest tooling). Compare resource identities, Service ports/selectors,
Deployments, ConfigMaps/CRs, RBAC, and ScaledObject targets. Confirm existing
monitoring receivers/processors/exporters and logs/traces are preserved and
scaling filters affect only the selected branch. Repeat preparation should
update the same objects and named map entries without duplicate pipelines,
exporters, releases, scrape jobs, or autoscalers.

Validate the **final rendered** collector configuration with the collector
distribution and version that will run it, for example:

```sh
otelcol-contrib validate --config=collector.yaml
```

The bundled chart stores configuration in its rendered ConfigMap `relay` key;
operator configurations are under `spec.config`, possibly nested in a hook
ConfigMap's `content` before the chart creates the CR. Extract the relevant
configuration and resolve its environment/file requirements privately. Validating
a fragment or parsing YAML alone does not check processor syntax or components
available in a distribution. See the
[collector validation documentation](https://opentelemetry.io/docs/collector/configuration/).
Use `helm lint` and repository checks when applicable; schema validation alone
does not reject every bad alias or incorrect collector configuration.

Server-side dry-run can check installed APIs/admission if access exists. Bind
cluster-facing commands to the selected context and namespace. Without the CRDs,
controller access, or required tools, state exactly what could not be checked;
do not install prerequisites simply to validate a GitOps diff.

## Observe the scaler

Resolve the real Service name, namespace, and REST port. For example, after
setting `OTEL_CONTEXT`, `OTEL_SCALER_NAMESPACE`, `OTEL_SCALER_SERVICE`, and
`OTEL_REST_PORT` from discovery, use a local-only forward:

```sh
kubectl --context "$OTEL_CONTEXT" --namespace "$OTEL_SCALER_NAMESPACE" \
  port-forward --address 127.0.0.1 "service/$OTEL_SCALER_SERVICE" \
  "19090:$OTEL_REST_PORT"
```

Keep the forward in a managed session and close only that session when finished.
If the existing topology has multiple scaler Pods, identify which Pod is queried
and which receives the exports; storage is per-process. Probe relevant Pods
individually if needed rather than assuming a Service aggregates their stores.

```sh
curl --fail --silent --show-error http://127.0.0.1:19090/info
curl --fail --silent --show-error http://127.0.0.1:19090/memstore/names
curl --fail --silent --show-error http://127.0.0.1:19090/memstore/data
curl --fail --silent --show-error \
  -H 'Content-Type: application/json' \
  --data-binary @query.json \
  http://127.0.0.1:19090/memstore/query
```

Create `query.json` from the **actual** trigger's `metricQuery` and
`operationOverTime`. This example matches the policy in [metrics](metrics.md):

```json
{
  "query": "sum(checkout_requests{namespace=shop,deployment=checkout})",
  "operationOverTime": "rate"
}
```

The data dump is keyed by stored metric name. Each series includes `labels`,
`data` samples with `value`/`time`, `aggregatesOverTime`, and `lastUpdate`.
Inspect only relevant metrics and avoid publishing a complete potentially
sensitive label dump. Check timestamps against the observation time and
retention, producer identities, and the expected set of matching series.

Important response behavior in the inspected version:

- Missing metrics can return HTTP 200 with `ok: true`, `value: -1`, and
  `error: "Metric not found"`. Inspect the error field as well as HTTP status.
- A zero result can mean genuine zero traffic, insufficient samples, no matching
  fresh series, or a missing-data fallback. Check raw samples and collection
  health before calling it a valid zero.
- Querying in lazy mode subscribes to the metric/operation; the first response
  can be missing. Wait for subsequent real exports and, for rate, at least two
  fresh samples with distinct timestamps, then repeat the query.
- Raw query results do not apply trigger clamps or KEDA/HPA calculations.
  Compare those separately. A dump entry alone does not prove query freshness.

Never call `POST /memstore/reset` during ordinary setup verification or diagnosis.
The documented read/query operations need no store reset.

## Observe KEDA and scaling

In the workload namespace, inspect the ScaledObject conditions/events, its
actual `status.hpaName`, the generated HPA's metrics/conditions/events, and the
target workload's desired/ready replicas. Confirm KEDA can reach the external
scaler's **KEDA** port and the target reference is in the ScaledObject namespace.
Inspect APIService/metrics-server health only when the observed failure points
there; the collector alone cannot prove KEDA metrics delivery.

Explain the expected scaling response using observed metric units, target,
metric type, bounds, polling/export delay, cooldown, and HPA stabilization.
Under existing traffic or a requested bounded test, observe the metric changing
and the corresponding replica decision; stop at configured test limits. Check
that original monitoring output still receives fresh data when accessible.
Do not delete or take ownership of an existing HPA/ScaledObject to force success.

Report checks as configuration prepared/validated, deployed/ready, metric
verified, and scaling observed, with actual evidence at each reached stage.
A GitOps PR is not a deployed installation, and without suitable traffic, a
healthy pipeline and HPA do not prove a replica transition.
