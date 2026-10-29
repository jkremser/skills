# Kedify

A plugin for retrieving Kubernetes CPU and memory recommendations through
Kedify MCP or local kedr, then applying selected changes through GitOps or
kubectl. It also sets up and diagnoses KEDA autoscaling through the Kedify OTel
add-on using applications' existing metrics, and provides read-only access to
Kedify organizations, clusters, agents, savings, and autoscaling diagnostics
through MCP.

## Recommendation skills

| Skill | Purpose |
| --- | --- |
| [recommendations-get](skills/recommendations-get/SKILL.md) | Ask whether to retrieve stored recommendations through Kedify MCP or run kedr locally, then return a source-independent snapshot. |
| [recommendations-apply-gitops](skills/recommendations-apply-gitops/SKILL.md) | Take existing recommendations, identify the cluster configuration repository, update the effective Argo CD/Flux/Helm/Kustomize/plain-manifest inputs, and create a pull request. |
| [recommendations-apply-kubectl](skills/recommendations-apply-kubectl/SKILL.md) | Ask which existing recommendations to live-apply, warn that GitOps may overwrite them, then patch and verify the selected workloads. |

The retrieval skill does not apply changes. Both application skills accept
recommendations from either source or a supplied report, and load the retrieval
skill only when recommendations are missing. The three skills share a
[recommendation handoff](references/recommendation-set.md) and are distributed
together in this plugin.

Kedify MCP requires a running agent with pod metrics collection enabled for the
target namespaces and enough collected history. Collection can be enabled in the
Kedify web UI or by labeling each namespace `insights.kedify.io/scrape=true`.
MCP retrieval works in a hosted UI with a connected Kedify account. Local kedr
requires access to the user's Kubernetes context and an existing
Prometheus-compatible source with kube-state-metrics data. Live application with
kubectl also requires direct cluster access.

For example, in Codex:

```text
Use $recommendations-get to retrieve recommendations for my cluster.
Use $recommendations-apply-gitops to put recommendations 2 and 5 into a PR.
Use $recommendations-apply-kubectl to live-apply only memory changes for 3.
```

## OTel autoscaling skills

| Skill | Purpose |
| --- | --- |
| [otel-autoscaling-setup](skills/otel-autoscaling-setup/SKILL.md) | Connect an application's existing Prometheus or OTLP metrics to the Kedify OTel add-on and KEDA, prepare Helm/GitOps configuration or install it, and verify the metric and scaling policy. |
| [otel-autoscaling-diagnose](skills/otel-autoscaling-diagnose/SKILL.md) | Trace missing metrics, incorrect query results, and scaling failures across the application, collector, scaler store, KEDA, and HPA; apply focused fixes when requested. |

Setup reuses existing applications, monitoring pipelines, KEDA, and collectors
where suitable. It can deploy a small collector when needed, preserves the
existing monitoring output, and asks for unresolved metric targets and replica
bounds. Adding application instrumentation is outside the initial workflow.
Diagnosis uses the add-on's existing REST API and Kubernetes observations.

These skills share references for [topology selection](references/otel-autoscaling/topology.md),
[metric semantics](references/otel-autoscaling/metrics.md),
[TLS](references/otel-autoscaling/tls.md), and
[verification](references/otel-autoscaling/verification.md).
They use local repository/Helm/Kubernetes tools and do not call Kedify MCP or
require a Dashboard account or Kedify agent. Without cluster access, they can
prepare configuration and explain what remains unverified.

```text
Use $otel-autoscaling-setup to scale checkout from our existing OTel metrics.
Use $otel-autoscaling-setup to prepare a GitOps diff for my app's Prometheus metrics.
Use $otel-autoscaling-diagnose: the metric is in Grafana, but checkout isn't scaling.
```

## How to use it

Add the GitHub repository as a marketplace, then install the Kedify plugin:

```sh
codex plugin marketplace add jkremser/skills
codex plugin add kedify@kedify
```

The first command adds the `kedify` marketplace. The second installs the
`kedify` plugin from that marketplace.

For MCP features, when prompted to connect the MCP server, choose **Kedify
Dashboard** and sign in with your existing dashboard account. Start a new Codex
session after installation to use the bundled tools and skills.

For local OTel workflows without MCP onboarding, use the skills directly from a
checkout: ask your coding agent to read and follow
`skills/otel-autoscaling-setup/SKILL.md` or
`skills/otel-autoscaling-diagnose/SKILL.md`. Keep the checkout's `references/`
directory in place; the shared guidance is part of the skills.
