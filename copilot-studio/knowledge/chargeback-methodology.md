# Chargeback Methodology

## Source
Helios report `storage-consumption-group` ("Storage Consumption by Protection Groups"), read through `reporting-get-report-preview` with `componentIds: ["400","401"]` and `dataComponentLimit: 5000`.
- Component 400: one row per protection group and target.
- Component 401: summary with `countGroupId` and `sumScResiliencyBytes`, used to check the rows.

## Billing basis
| Basis | Helios field | Meaning |
|---|---|---|
| Storage consumed (default) | `scRetainedBytes` | Physical storage used by the group's retained data, after dedupe and compression |
| Including resiliency | `scResiliencyBytes` | The same, plus the extra copies the cluster keeps for resiliency (erasure coding or replication factor) |
| Front-end data | `dataIngestedRetainedBytes` | Logical size of the data protected and retained, before data reduction |

Binary units: 1 GiB = 2^30 bytes.

## Calculations
- Charge per month per row = bytes on the chosen basis / 2^30 x price per GiB.
- Yearly figure = monthly x 12.
- Share of total = row charge / total charge. Cluster share = sum of its rows / total.
- Change = the growth Helios reports for the same basis over the report's own window (`scRetainedBytesGrowth`, `scResiliencyBytesGrowth`, `dataIngestedRetainedBytesGrowth`). The window is Helios's default, not chosen by you.
- Data reduction = Helios's `dataReduction` ratio.
- The price is whatever the user gives. Do not assume a price list, convert currencies or produce an invoice.

## Row fields
| Meaning | Source field |
|---|---|
| Cluster | `system` |
| Group | `groupName` (strip a leading `_DELETED_` and label the group deleted) |
| Sources | `sourceNames` (show at most 4) |
| Environment | `environment` (raw code such as kVMware; show a readable label) |
| Policy | `policyName` |
| Front-end data | `dataIngestedRetainedBytes` |
| Storage consumed | `scRetainedBytes` |
| Including resiliency | `scResiliencyBytes` |
| Reduction | `dataReduction` |

- **Cluster overhead**: a row with an empty `groupName` is storage Helios could not attribute to a protection group. Hide it unless the user asks, and never add it silently to a team's charge.
- **Replication or archive targets**: rows with a target other than Local are separate rows with the same group name. Keep them and show the target name next to the policy.
- **Empty groups**: groups with zero storage on the chosen basis are hidden unless the user asks.

## Checks
- Number of 400 rows must equal 401 `countGroupId`.
- Sum of `scResiliencyBytes` over the 400 rows must equal 401 `sumScResiliencyBytes`.
- If the 400 row count equals the limit, raise the limit and call again.
- If a check fails, say so in the footnote.
- Clusters registered in Helios but missing from the report are listed in the footnote.
