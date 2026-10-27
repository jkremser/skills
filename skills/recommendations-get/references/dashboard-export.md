# Read a Kedify Dashboard export

These files are created by clicking **Export** in Kedify Dashboard. Read the
supplied attachment or local path as a saved snapshot; no cluster or MCP reads
are needed. If no file was supplied, ask for the export or its path. If it cannot
be read or parsed, report the problem rather than treating it as an empty set.

Use the format's structure to extract workload identity, resource leaf, and exact
current → proposed quantities. Retain source IDs, cluster identity, timestamps,
status, confidence, and data quality when available for the
[shared handoff](../../../references/recommendation-set.md). Missing evidence is
unknown, not a reason to discard an otherwise readable proposal or invent data.

## Format mapping

| Data | JSON | CSV | Markdown |
| --- | --- | --- | --- |
| Records | Entries in `recommendations[]` | Rows keyed by the header | Each `## <kind> <namespace> / <name>` section |
| Workload | `workload.kind`, `.namespace`, `.name`, `.container`; legacy fallback to top-level `kind`, `namespace`, `name` and `labels.workloadName`, `labels.workloadContainer` | `kind`, `namespace`, `workload`, `container` | Section heading and `Container` |
| Cluster | `clusterId`, `clusterName` | `cluster_id`, `cluster_name` | `Cluster` name and parenthesized ID |
| Source ID | `id` | `recommendation_id` | `Recommendation ID` |
| Resource locator | `workload.fieldPath` / explicit action target when present; otherwise `resourceUID` | `field_path`, otherwise `resource_uid` | `Target`, otherwise `Resource` |
| Current value | `labels.originalValue`, with `labels.originalValueUnset` when present | `original_value` | Left side of `Proposed value` |
| Proposed value | `labels.suggestedValue` or an explicit structured `proposedActions` quantity | `suggested_value` or an explicit structured `proposed_actions` quantity | Right side of `Proposed value` |

For JSON, read the `metadata` envelope for export time, scope, and count; it is not
a recommendation. Prefer typed `workload` identity and top-level `confidence` and
`dataQuality` over legacy label copies, which may contain defaults. Keep workload
UID, release, observation, evidence, policy, and fingerprint when present. Use
the quantity string (`769Mi`, `10m`) rather than guessing units or rounding
`suggestedValueRaw`. Do not derive additional changes from decision traces,
effective policy, or explanations.

For CSV, use a CSV parser such as Python's `csv.DictReader`, not comma splitting.
Quoted cells can contain commas, escaped double quotes, and newlines. Parse
nonempty `data_quality` and `proposed_actions` cells as JSON. An empty cell is
missing data, not zero or an instruction to unset a resource.

For Markdown, read the labeled fields within each section. Decode Markdown
backslash escapes in headings and plain text, such as `prometheus\-server` and
escaped UUIDs; preserve inline code contents literally. Read the two code spans
in `Proposed value` as exact quantities, and `Data quality` as JSON. Empty
`Target` code spans mean no explicit field path. Repeated workload headings are
separate records, distinguished by recommendation ID and resource. Keep the
document's Generated/Scope/Recommendations metadata when present.

## Resolve the resource leaf

Use an explicit, unambiguous CPU/memory request/limit target when available.
Otherwise the export's `resourceUID` / `resource_uid` / `Resource` has the form
`<namespace>/<kind>/<workload>/<container>/<setting>`. Match its identity against
the record (kind is lowercased in this locator), then map its final component:

| Setting | Normalized leaf |
| --- | --- |
| `cpu-requests` | `requests.cpu` |
| `cpu-limits` | `limits.cpu` |
| `memory-requests` | `requests.memory` |
| `memory-limits` | `limits.memory` |

An empty field path is therefore valid when the resource locator identifies the
leaf. Keep conflicting targets or values, unsupported settings, and incomplete
identities review-only; never infer a target from the explanation alone.
`unset` explicitly means the current field is absent; it differs from a missing
or unknown current value and never authorizes deletion.

For example, a Deployment `monitoring/prometheus-server`, container
`prometheus-server`, yields `requests.memory: unset → 769Mi` and
`requests.cpu: unset → 10m`. These can share one container row while retaining
both source IDs and per-field provenance. Neither proposal implies a limit change.

Apply the requested filters within the file and keep clusters distinct. Preserve
lifecycle and quality guards: only active/waiting, unsuppressed proposals with
established targets and quantities can be offered for application; an explicit
`dataQuality.actionable: false` remains blocked. Report export time separately
from file-read and observation times, and disclose missing freshness or identity
evidence. Flag a count mismatch or partially parsed export as incomplete. Return
the normalized snapshot to recommendations-get for presentation without preparing
patches or repository changes.
