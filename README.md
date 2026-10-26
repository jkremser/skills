# Kedify

A plugin for retrieving Kubernetes CPU and memory recommendations through
Kedify MCP or local kedr, then applying selected changes through GitOps or
kubectl. It also provides read-only access to Kedify organizations, clusters,
agents, savings, and autoscaling diagnostics through MCP.

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

## How to use it

Add the GitHub repository as a marketplace, then install the Kedify plugin:

```sh
codex plugin marketplace add jkremser/skills
codex plugin add kedify@kedify
```

The first command adds the `kedify` marketplace. The second installs the
`kedify` plugin from that marketplace.

When prompted to connect the MCP server, choose **Kedify Dashboard** and sign in
with your existing dashboard account. Start a new Codex session after
installation to use the bundled tools and skills.
