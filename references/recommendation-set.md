# Recommendation handoff

All three recommendation skills use this source-independent handoff. It can be a
structured conversation result or a private JSON artifact when files are
available. Preserve the original source response alongside the normalized rows;
do not put raw metrics, kubeconfigs, or credentials into Git or PR descriptions.

## Snapshot and target identity

Record the retrieval time, source (`mcp`, `kedr`, or a supplied report), namespace
and workload scope, observation window, strategy/policy when reported, and any
partial failures or evidence limitations. Keep these cluster identifiers distinct:

- MCP: organization, cluster ID/name, and agent ID when available.
- Local access: kubeconfig location and explicitly chosen context, when available.

An MCP cluster name is not a kubeconfig context. Before application through local
tools, establish that the selected context points to the recommendation's cluster;
never infer the mapping from a similar name. Keep management-cluster contexts
separate from the workload context.

## Rows

Each row represents one container in one workload and carries:

| Field | Contents |
| --- | --- |
| Selection ID | Stable ID within this saved snapshot |
| Source identity | MCP record IDs/fingerprints/action locators, or kedr scan index/run |
| Workload | Cluster identity, namespace, API kind, name, and container name |
| Baseline | UID, release/images/template and capture time, to the extent actually observed |
| Changes | Exact current and proposed quantities for each offered CPU/memory request/limit leaf |
| Evidence | Observation window, confidence, quality, warnings, suppression, lifecycle status |
| Applicability | Actionable or a concrete reason the row is blocked/review-only |

If multiple source records describe a container, retain their individual IDs and
per-field provenance when grouping them into one selection row. Conflicting
proposals for the same leaf are blocked until resolved. Do not change IDs or
silently replace values when paging, filtering, or handing off to another skill.

Represent CPU/memory leaves as `requests.cpu`, `limits.cpu`, `requests.memory`,
and `limits.memory`, with exact Kubernetes quantity strings such as `250m` or
`256Mi`. Distinguish an explicitly unset current field from an unknown current
field. Missing values do not authorize deleting a field. Offer only finite,
positive, unsuppressed proposed values whose units and target are established.
Preserve all unselected leaves and unrelated resources.

Only active/waiting MCP recommendations are offered for application. Other
lifecycle states can be displayed for review, but need fresh actionable evidence
before application. Preserve source suppression and quality guards; do not remove
them or reactivate dismissed records merely to create a change.

The kedr adapter interprets CPU cores and memory bytes as documented in its
reference. MCP payloads have their own schema: retain quantity strings or use
explicitly documented units; never assume kedr's numeric units apply to MCP.
Do not turn prose, rounded display values, or a suggested shell command into an
executable patch. Missing structured target/value information makes a row
review-only until reliable evidence or user clarification supplies it.

## Application boundary

The retrieval skill presents recommendations without preparing live patches or
repository changes. Application skills accept this handoff or normalize an
existing user-supplied report to the same information. They do not require kedr,
Prometheus, or another retrieval run when usable recommendations already exist.

Application skills re-read the chosen targets/source files, compare known UIDs,
release/template identity and current quantities, and account for constraints and
ownership before preparing the concrete changes. A read taken during application
is a new baseline, not proof of the workload state when metrics were collected.
If identity or sizing inputs changed, retrieve fresh recommendations through the
chosen source and review materially changed values. If historical identity is
missing, disclose that limitation and resolve applicability with source evidence
or the user's review before treating the row as executable.

Selections and authorization travel with the handoff: record selected IDs,
fields, delivery method, and any already-authorized external actions. A request
to retrieve recommendations alone does not authorize application. Reuse explicit
prior authorization for an unchanged proposal; ask again only for a material
change in values, target, impact, or scope. `All` means the displayed actionable
rows, not unseen pages or other clusters.
