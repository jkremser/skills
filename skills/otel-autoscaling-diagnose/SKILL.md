---
name: otel-autoscaling-diagnose
description: >-
  Diagnose missing metrics, incorrect query results, and scaling failures in
  existing Kedify OTel add-on and KEDA installations. Trace the application,
  collector, scaler metric store, KEDA, and HPA; apply focused fixes when requested.
---

# Diagnose OTel autoscaling

Find the first failing boundary between application metrics and replica changes,
supporting conclusions with observations. Use local tools or supplied manifests,
logs, and diagnostic responses; Kedify MCP and a Dashboard account are not
prerequisites. A metric in Grafana proves availability in that backend, not in
the add-on's independent in-memory store.

## Establish the affected path

Reuse setup's workload, cluster context, release, source files, query, and prior
checks when available. Otherwise inspect the supplied artifacts and discover
the owning configuration. Honor a selected context; if absent, list contexts
and ask which to use before contacting a cluster. Pass explicit context and
namespace flags on Kubernetes/Helm calls, without changing kubeconfig defaults.
Discover APIs before querying optional CRs; distinguish RBAC failures from absence.

Inspect the ScaledObject, its target and generated HPA, add-on Service and
Endpoints/EndpointSlices, versions, collector pipelines, recent logs, conditions,
and events. Read [topology selection](../../references/otel-autoscaling/topology.md)
only for an unclear metric route or collector ownership. Do not install a demo
stack or reset the metric store during diagnosis.

## Trace the first failing boundary

Read [verification](../../references/otel-autoscaling/verification.md) for the
API contract and evidence checks, and
[metric semantics](../../references/otel-autoscaling/metrics.md) to interpret
names, labels, rates, aggregation, and freshness. Follow the evidence upstream
or downstream rather than assuming every incident needs all checks:

| Boundary | Evidence and likely causes |
| --- | --- |
| App → collector | Raw metric/sample, scrape or export errors, actual target port/path, receiver and discovery configuration; preserve each producer's identity. |
| Collector → scaler | Scaling pipeline references the intended receiver, filter, and OTLP/gRPC exporter; check export errors, Service endpoints, namespace, DNS, NetworkPolicy, and protocol. |
| Scaler store → query | Inspect stored names, labels, timestamps and sample counts; evaluate the exact query and operation. Check name conversion, filter timing, missing labels, temporality, lazy subscription/warm-up, and staleness. |
| Scaler → KEDA | Match `scalerAddress` to the KEDA gRPC port, inspect KEDA errors and TLS configuration, trigger metadata, target value and clamps. |
| KEDA/HPA → workload | Check target identity, Ready/Active/paused conditions, HPA metrics and events, replica limits, stabilization/cooldown, unavailable Pods, scheduling, and other controllers managing replicas. |

For TLS/authentication failures, read
[TLS and credentials](../../references/otel-autoscaling/tls.md). On default
ports, `4317` accepts OTLP/gRPC metrics while `4318` serves the KEDA external
scaler; neither is the add-on's OTLP/HTTP ingestion endpoint.

Do not equate HTTP 200, `ok: true`, a nonempty metric-name list, a zero result,
or a Ready ScaledObject with a fresh matching metric. Interpret the response
body and stored series together. Without access, identify which supplied
evidence supports the finding and provide the next discriminating check.

## Explain and, when requested, fix

Report the affected workload, first failing boundary, evidence, likely cause,
and smallest corrective change. Separate confirmed causes from hypotheses.
Keep a diagnosis-only request observational; the documented query call can warm
lazy subscriptions, but does not authorize changing deployment configuration.

When a fix is requested, prepare the concrete diff, preserve monitoring output
and unrelated autoscalers, validate it, and use the existing GitOps/live delivery
choice and authorization. Follow
[otel-autoscaling-setup](../otel-autoscaling-setup/SKILL.md) for delivery when
needed; do not restart discovery or require reinstallation. Do not turn absent
data into a zero fallback, widen filters/collection to the whole cluster, or
disable TLS checks to hide the failure.

Repeat the check that demonstrated the fault, then check metric freshness,
query correctness, KEDA/HPA readiness, and observed scaling where possible.
Report prepared, applied, pending, and verified states separately, including
partial failures and anything that still needs traffic or deployment.
