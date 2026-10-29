# Choose a metric path

Use the [add-on chart](https://github.com/kedify/otel-add-on/tree/main/helmchart/otel-add-on)
and examples as starting points, then inspect the selected version. The chart
inspected for this skill was 0.1.5; it is not a promise about the latest published
version. Its defaults enable the scaler and disable both optional subcharts.

## Existing collector

Install/reuse only the scaler. With this chart, `otelCollector.enabled: false`
and `otelOperator.enabled: false` leave the user's collector/operator in place.
Find the real configuration owner: a standalone collector chart, ConfigMap,
`OpenTelemetryCollector` CR, application chart, or GitOps source. Add a named
exporter and a separate `metrics/keda` pipeline alongside the existing metrics
pipeline. Keep existing monitoring exporters, traces, and logs unchanged.

For example, this is an **addition to an existing collector configuration**,
not replacement chart values. It assumes an existing `otlp` receiver, a
cumulative `checkout.requests` metric, and an intentionally plaintext in-cluster
connection. Substitute the actual receiver, name, endpoint, and TLS settings.

```yaml
exporters:
  otlp/keda:
    endpoint: scaling.telemetry.svc:4317
    tls:
      insecure: true
processors:
  filter/keda:
    error_mode: ignore
    metrics:
      metric:
        - 'name != "checkout.requests"'
service:
  pipelines:
    metrics/keda:
      receivers: [otlp]
      processors: [filter/keda]
      exporters: [otlp/keda]
```

Merge the map entries into the effective source; do not replace `exporters`,
`processors`, or `service.pipelines` wholesale. Preserve existing processors
needed for identity, metric conversion, or resource limits in the scaling
branch. Filter conditions drop matching data; an allowlist for two metrics uses
one condition such as `name != "a" and name != "b"`, not two ORed exclusions.
Only the scaling branch should receive scaling-specific filters/transforms.
Read [metrics](metrics.md) for which name to filter and which labels to copy.

## Prometheus metrics without a suitable collector

An existing Prometheus/Grafana installation does not by itself forward metrics
to this scaler. Prefer a suitable collector already scraping the app. Otherwise
deploy a small dedicated collector to scrape the selected app endpoints while
leaving the user's monitoring stack intact. Do not mistake a Prometheus server's
own `/metrics` endpoint for the application's metrics or a PromQL API for an
OTLP source. If only a backend is accessible, establish a supported receiver,
export, or federation path and validate it; report a missing integration instead
of inventing a backend-to-OTLP bridge.

For the inspected add-on chart, the bundled collector uses
`otelCollector.alternateConfig`. This complete example assumes one app producer
at `checkout.shop.svc:8080`, a cumulative Prometheus counter named
`checkout_requests_total`, Helm release `scaling` in namespace `telemetry`, and
plaintext in-cluster export. Adapt it to the selected version and app.

```yaml
replicaCount: 1
otelOperator:
  enabled: false
otelCollector:
  enabled: true
  mode: deployment
  replicaCount: 1
  alternateConfig:
    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
    receivers:
      prometheus:
        config:
          scrape_configs:
            - job_name: checkout
              scrape_interval: 5s
              static_configs:
                - targets: [checkout.shop.svc:8080]
                  labels:
                    namespace: shop
                    deployment: checkout
    processors:
      filter/keda:
        error_mode: ignore
        metrics:
          metric:
            - 'name != "checkout_requests_total"'
    exporters:
      otlp/keda:
        endpoint: scaling.telemetry.svc:4317
        tls:
          insecure: true
    service:
      extensions: [health_check]
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: [filter/keda]
          exporters: [otlp/keda]
```

The collector bundled with the inspected chart preserves this counter's
`checkout_requests_total` name. After confirming the stored series, use
`sum(checkout_requests_total{namespace=shop,deployment=checkout})` with
`operationOverTime: rate`. This is a different received metric from the
`checkout.requests` OTLP example above, which the scaler stores as
`checkout_requests`. Verify names again for the user's collector version and
translation settings. The static labels here identify the known target's
namespace/deployment; they do not provide individual Pod identity.

For a replicated app, scrape individual Pods/endpoints with stable distinct
producer labels. A load-balanced Service counter can mix several producers
into one series; do not use the single-producer example unchanged. Scope pod
discovery to the app namespace and selectors, preserving pod/instance identity
and granting only the discovery RBAC needed. Static targets need no discovery
RBAC. Multiple collectors scraping identical targets can duplicate samples;
reuse existing sharding/target allocation rather than adding identical replicas.

## App exports OTLP without a collector

Deploy a minimal collector with an OTLP receiver for the app's configured
protocol, filtering and exporting metrics through **OTLP/gRPC** to the add-on.
Expose the receiver's matching port through its Service and bind it to a
reachable address, not loopback. OTLP/HTTP from an app must terminate at an
HTTP-capable collector receiver; the add-on does not provide that ingestion
protocol on its default `4318`. Preserve the app's existing destinations and
avoid changing application instrumentation. If exporter configuration must
change, make it in the app's maintained deployment configuration.

## Existing OTel Operator and advanced patterns

Reuse an existing operator and its CRDs. The chart exposes `otelOperatorCrs` with
defaults in `otelOperatorCrDefaultTemplate`; `alternateOtelConfig` supplies a
complete CR configuration, distinct from bundled collector `alternateConfig`.
Chart-created CRs use Helm hook Jobs. Verify the deployment system runs those
hooks; prefer directly managed CR manifests when that is the repository's
established pattern. Do not create two owners for the same collector.

Do not install an operator just to run one collector. Sidecars and target
allocation are useful when already required, not automatic prerequisites.
The upstream `examples/metric-push`, `examples/metric-pull`,
`examples/otel-operator`, `examples/pipelines-tls`, `examples/vllm`, and
`examples/cloudwatch-metric-stream` illustrate patterns. Read only the relevant
example and translate it into the user's deployment; demo setup/cleanup scripts
can create or delete clusters and infrastructure and are not onboarding commands.

## Addresses and replicas

In the inspected chart, Service name precedence is `service.name`, then
`fullnameOverride`, then Helm release name; namespace is `namespace` or release
namespace. Confirm rendered Services rather than inferring from helper names.
Default ports are OTLP/gRPC `4317`, KEDA gRPC `4318`, diagnostics HTTP `9090`,
and internal Prometheus metrics `8080`. Use the actual configured ports.
The operator-generated default exporter hardcodes `4317` in this version;
overriding the Service port requires explicit exporter configuration as well.

Keep one scaler replica unless an established topology routes identical required
data to each queried instance. Storage is local to each process, not shared;
increasing replicas behind a Service is not sufficient for high availability.
