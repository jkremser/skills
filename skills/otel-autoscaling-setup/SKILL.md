---
name: otel-autoscaling-setup
description: >-
  Install and configure the Kedify OTel add-on for KEDA autoscaling using an
  application's existing Prometheus or OTLP metrics. Use to onboard a workload,
  connect an existing collector or metrics stack, or prepare Helm/GitOps changes.
---

# Set up OTel autoscaling

Connect a real workload's metric to a working KEDA scaling policy. Reuse the
user's applications and monitoring stack. This workflow uses local repository,
Helm, and Kubernetes tools; it does not require Kedify MCP, a Dashboard account,
or a Kedify agent. Without cluster access, prepare reviewable configuration and
verification commands and describe deployment as unverified.

## Discover the workload and configuration owner

Read repository instructions and inspect application manifests, Helm values,
Argo CD/Flux sources, and collector configuration before asking questions.
Establish the target workload, namespace, metric source, and deployment owner.
Honor an already-selected Kubernetes context; otherwise list local contexts and
ask which to use before contacting a cluster. Bind kubectl to `--context` and
Helm to `--kube-context`; never change the current context. Use explicit namespaces.

With access, discover installed APIs, KEDA and add-on releases, collectors and
operators, Services, and any HPA/ScaledObject already targeting the workload.
Distinguish absent components from access failures. Do not replace an existing
autoscaler or install another KEDA/operator merely because a demo does so.

Ask only for unresolved choices: the workload, a suitable metric, positive target
value and its units, minimum/maximum replicas, and GitOps versus live delivery.
Reuse explicit choices and authorization. For GitOps-managed resources, update
the effective source; an explicit request for a live override can use the live
path after explaining reconciliation. If the app exposes no suitable metric,
identify the missing signal and required labels; adding application
instrumentation is outside this skill.

## Choose the metric path and scaling policy

Read [topology selection](../../references/otel-autoscaling/topology.md) for the
applicable existing-collector, Prometheus-scrape, OTLP, or operator path. Prefer
an existing suitable collector with a separate scaling pipeline. Keep existing
monitoring exporters and pipelines intact and scope collection to the selected
workload. Do not deploy demo apps, Grafana, Prometheus, or cloud infrastructure
to onboard the user's app.

Read [metric semantics](../../references/otel-autoscaling/metrics.md) before
choosing the query, operation, and threshold. Inspect an actual sample and, when
available, the scaler's stored names and labels. Account for producer identity,
counter temporality, histogram conversion, scrape/export cadence, and retention.
Require explicit replica bounds; do not enable scale-to-zero unless the metric
source remains available and can activate the workload while it has zero Pods.

Resolve the exact chart version from the user's pin, existing release, or
available published artifacts. Inspect that version's values, dependencies,
schema, and rendered resources; examples in the upstream repository can use
older versions and aliases. Derive service names, namespaces, and ports rather
than copying `keda-otel-scaler.keda.svc`. For TLS or authenticated sources, read
[TLS and credentials](../../references/otel-autoscaling/tls.md).

## Prepare and deliver the change

Produce the minimal add-on values, collector configuration, and `ScaledObject`
changes. Use the established Helm/Kustomize/plain-manifest inputs, preserving
other consumers of shared values. Update the existing named pipeline, exporter,
release, and ScaledObject on a repeat run; do not append duplicates.

Read [validation and verification](../../references/otel-autoscaling/verification.md).
Render with effective values and pinned versions, compare before/after, and
validate the final collector configuration using its actual distribution and
version. Confirm monitoring output remains configured, only intended resources
change, and the query is supported. Show missing tools/access as check limits.

- **GitOps:** prepare a focused diff in the owning repository. Create a PR when
  requested, using the existing authorization; otherwise leave the diff for
  review. Do not merge, force reconciliation, or run a live Helm installation
  as a side effect of preparing a PR. Describe delivery as pending until observed.
- **Live installation:** reuse a suitable installed KEDA, installing a compatible
  pinned version only if missing and within the requested setup. Install/upgrade
  the add-on with the selected values and configure the collector before creating
  or updating the ScaledObject. Wait for required CRDs/controllers. Stop on a
  failed stage, report partial installation, and diagnose before retrying.

## Verify and report

Follow the verification reference to prove metric arrival, producer labels and
freshness, the exact query result, and KEDA/HPA readiness. Observe scaling under
existing or explicitly requested test traffic when possible. Do not generate
production traffic merely to finish verification.

Return the workload/context, chosen metric path, query/units/target and replica
bounds, artifact paths or PR URL, checks and observations, and pending work.
Distinguish configuration prepared, deployment ready, metric verified, and
scaling observed. Retain the concrete release, source, query, and deployment
mapping for diagnosis. For failures, follow
[otel-autoscaling-diagnose](../otel-autoscaling-diagnose/SKILL.md) without losing
the user's scope or repeating resolved questions.
