# Metric and scaling semantics

Inspect the selected add-on version's `metric/simple_parser.go`,
`metric/mem_store.go`, `receiver/receiver.go`, and `scaler/handlers.go` in
[the source repository](https://github.com/kedify/otel-add-on) when behavior is
unclear. These details describe the implementation inspected with chart 0.1.5.

## Names, labels, and queries

- Query one metric with exact label matches, optionally wrapped in `sum`, `avg`,
  `min`, `max`, or `count`: `sum(checkout_requests{namespace=shop})`.
  The default cross-series aggregation is `sum`. Full PromQL functions,
  range selectors, regex/negative matchers, arithmetic, joins, and grouping
  clauses are not supported. The parser is permissive; accepting a string does
  not prove that it means what a PromQL author intended.
- Set `operationOverTime` separately: `last_one` (default), `rate`, `min`, `max`,
  `avg`, or `count`. The store applies that operation per series before the
  cross-series aggregation. `count` over time counts stored samples, whereas
  `count(metric)` counts series; neither is a request rate.
- The store replaces `.` and `/` in names with `_`. Upstream Prometheus-to-OTLP
  translation can also change suffixes. Inspect stored names instead of assuming
  the name displayed in Grafana is the scaler's name.
- Histograms, exponential histograms, and summaries are stored as `<name>_count`
  and `<name>_sum`. Buckets/quantiles are not available for query-side quantile
  calculation. Filter the original received metric before this conversion,
  not its later `_count` derivative.
- The receiver stores datapoint attributes, with limited resource-label copying
  (`host.name`, or a resource `pod` fallback). It does not automatically expose
  every OTel resource attribute as a query label. Where needed, use a scoped
  transform to copy `k8s.namespace.name`, `k8s.pod.name`,
  `k8s.deployment.name`, or `service.instance.id` into datapoint attributes.
  Confirm the actual stored labels and preserve distinct producer identity.

The inspected store's partial label matching does not reliably reject a series
when a requested label is absent. Check that **all candidate series** have the
intended scoping labels and that selection isolates the intended workload; a
plausible query result alone does not establish isolation. Do not propose a
source-code fix during ordinary onboarding; filter/normalize the selected
scaling stream or report the version's limitation.

## Choose an operation and target

For gauges such as queue length or in-flight requests, typically use `last_one`
and choose `sum`, `max`, or `avg` according to whether the signal is total work,
a bottleneck, or a per-producer observation. Do not sum a global queue length
repeated identically by every replica.

For cumulative monotonic counters, use `rate`. Preserve producer identity so
counter resets and concurrent exporters are not mixed in one series. The
inspected implementation uses positive deltas over the retained sample window,
including resets, and needs at least two samples with different timestamps.
It is not the full Prometheus `rate()` implementation. OTel delta sums require
a suitable conversion to cumulative or another signal; do not take a rate of
delta samples as though they were cumulative counters.

Choose `sum` versus `avg` with the HPA metric type. With the normal
`AverageValue` trigger behavior, total request rate divided by a per-replica
target gives the intended capacity relationship. Averaging per-Pod rates first
and then using that same per-replica target can under-scale. If an existing
configuration uses `metricType: Value`, preserve and explain its different HPA
calculation instead of silently changing it.

Example **ScaledObject manifest** for the existing-collector path's
`checkout.requests` OTLP metric (stored as `checkout_requests`), only after the
user chose 20 requests/second per replica and bounds 1–5 and observation
established the metric/labels. The fresh Prometheus-scrape example in
[topology selection](topology.md) uses `checkout_requests_total` instead.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: checkout-otel
  namespace: shop
spec:
  scaleTargetRef:
    name: checkout
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
    - type: external
      metricType: AverageValue
      metadata:
        scalerAddress: scaling.telemetry.svc:4318
        metricQuery: sum(checkout_requests{namespace=shop,deployment=checkout})
        operationOverTime: rate
        targetValue: "20"
```

`clampMin`/`clampMax` bound the **metric value**, not replica counts. Use
`minReplicaCount`/`maxReplicaCount` for bounds. Do not copy the demo clamps or
aggressive stabilization settings without a reason. The inspected scaler
activates when its resulting metric is greater than zero; do not invent an
activation-threshold metadata field.

## Retention, missing data, and zero replicas

Choose retention to hold multiple useful observations at the actual scrape/
export interval plus expected delay; explain the resulting rate window. A
120-second retention setting is not a universal warm-up period. Check timestamps
as well as sample counts; stale entries can remain visible in the data dump.

Defaults are `lazySeries: false`, `lazyAggregates: false`, `errIfNotFound: false`,
and `valueIfNotFound: 0`. Missing data can therefore appear to KEDA as zero.
Choose missing-data behavior explicitly for the workload; do not use a fallback
to conceal a broken source. In lazy modes, the first query subscribes to the
metric/operation and subsequent exports populate it; allow for fresh samples.

At zero replicas, a Pod-scraped metric or an app's own OTLP stream disappears.
Require an independently available demand signal capable of waking the workload
before enabling scale-to-zero. A separate collector cannot manufacture that
signal if the source itself stops running.
