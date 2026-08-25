# 09 · FinOps & Cost Governance at Scale

[Level 3, Module 09](../level-3/09-cost-management-optimization.md) covered
cost levers for a single subscription. At enterprise scale, cost management
becomes an organizational practice — **FinOps**: making cost visible per
team/product, allocating shared costs fairly, and building governance that
enforces spending discipline across hundreds of subscriptions without a
human reviewing every one manually.

## Tagging strategy as the foundation

None of enterprise cost allocation works without consistent tags — every
resource needs to answer "which team/product/environment owns this cost":

```bash
az policy definition create \
  --name require-cost-center-tag \
  --display-name "Require CostCenter tag" \
  --rules '{
    "if": {
      "allOf": [
        { "field": "tags[CostCenter]", "exists": "false" },
        { "field": "type", "notEquals": "Microsoft.Resources/resourceGroups" }
      ]
    },
    "then": { "effect": "deny" }
  }'

az policy assignment create \
  --name enforce-cost-center-tag \
  --policy require-cost-center-tag \
  --scope "/providers/Microsoft.Management/managementGroups/alz"
```

A companion `Modify`-effect policy can **inherit** a tag from the resource
group onto resources that don't specify one, rather than only denying
untagged creation:

```bash
az policy definition create \
  --name inherit-costcenter-from-rg \
  --mode Indexed \
  --rules '{
    "if": { "field": "tags[CostCenter]", "exists": "false" },
    "then": {
      "effect": "modify",
      "details": {
        "roleDefinitionIds": ["/providers/Microsoft.Authorization/roleDefinitions/4a9ae827-6dc8-4573-8ac7-8239d42aa03f"],
        "operations": [{
          "operation": "addOrReplace",
          "field": "tags[CostCenter]",
          "value": "[resourcegroup().tags['CostCenter']]"
        }]
      }
    }
  }'
```

**Gotcha:** `Modify`-effect policies need a **remediation identity with
`Tag Contributor`** role granted at the scope, same as the Arc remediation
pattern from [Level 4, Module 04](04-azure-arc-hybrid-cloud.md) — and
existing resources are only fixed by running `az policy remediation
create` against the assignment, not automatically, so a newly assigned
tagging policy leaves the current fleet untagged until remediation runs.

## Cost allocation and showback/chargeback

```bash
az costmanagement query \
  --type Usage \
  --scope "/providers/Microsoft.Management/managementGroups/alz" \
  --dataset-grouping name=CostCenter type=TagKey \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --timeframe MonthToDate
```

- **Showback**: reporting cost per team/tag for visibility, no actual
  billing transfer.
- **Chargeback**: actually billing the cost back to a team's internal cost
  center/budget.

**Gotcha:** shared platform costs (the hub VNet, ExpressRoute circuit,
Log Analytics workspace) don't naturally attribute to any one team's tags —
a mature FinOps practice defines an explicit **allocation method**
(even split, proportional to consumption, or a flat platform tax) for
shared costs, or showback numbers understate real per-team cost by
excluding the platform overhead entirely, which undermines trust in the
reporting once someone notices.

## Reservation and savings plan governance

At enterprise scale, reservation purchases should be centralized (one team
deciding commitment size for the whole org) rather than every subscription
owner buying independently, since scope and utilization tracking get
fragmented otherwise:

```bash
az consumption reservation-recommendation list \
  --scope "/subscriptions/xxxx" \
  --resource-group-name rg-compute \
  --query "[].{sku:properties.skuProperties, recommended:properties.recommendedQuantity}" \
  -o table

az consumption reservation-summary list \
  --scope "/providers/Microsoft.Billing/billingAccounts/xxxx" \
  --grain monthly \
  --query "[].{sku:skuName, utilization:avgUtilizationPercentage}" \
  -o table
```

**Gotcha:** reservation utilization dropping below expectations (say under
80%) often isn't caught until a monthly bill review — set up a scheduled
query alert (from
[Level 4, Module 05](05-advanced-observability.md)) on utilization
percentage so under-utilized reservations get flagged and either resized
(via exchange) or the team is notified, rather than silently wasting
committed spend for months.

## Budget governance across the management group tree

```bash
az consumption budget create \
  --budget-name budget-mg-landingzones \
  --amount 50000 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2026-08-01 \
  --end-date 2027-08-01 \
  --scope "/providers/Microsoft.Management/managementGroups/alz-landingzones" \
  --notifications '{
    "Forecasted_GreaterThan_100_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "thresholdType": "Forecasted",
      "contactEmails": ["finops@example.com"]
    }
  }'
```

A **forecasted** threshold (`thresholdType: Forecasted`) alerts based on
projected month-end spend given current trajectory, catching a budget
overrun days before it actually happens — an **actual** threshold only
fires once the money is already spent, which is too late to intervene.

**Gotcha:** management-group-scoped budgets are a relatively newer
capability with some regional/API version constraints — confirm the
`Microsoft.CostManagement` API version supports management-group scope
budgets in your tenant before designing a governance model entirely around
them, and keep subscription-level budgets as a fallback layer regardless.

## FinOps maturity levels

| Stage | Characteristic |
|---|---|
| **Crawl** | Tags inconsistent, cost visible only at subscription level, reactive |
| **Walk** | Tag policy enforced, per-team showback dashboards, monthly reviews |
| **Run** | Automated chargeback, forecasted budget alerts, reservation utilization actively managed, cost baked into architecture decisions upfront |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az policy definition create` (deny/modify on tags) | Enforce or auto-apply cost allocation tags. |
| `az costmanagement query --dataset-grouping type=TagKey` | Aggregate cost by a tag (e.g. CostCenter). |
| `az consumption reservation-summary list` | Check reservation utilization across the org. |
| `az consumption budget create --scope <mg-id>` | Set a budget at management group scope. |
| `--notifications` with `thresholdType: Forecasted` | Alert before overrun happens, not after. |

## Exercise

1. Create a policy requiring a `CostCenter` tag and assign it at a
   management group scope; confirm a resource without the tag is denied.
2. Add a companion `Modify` policy inheriting the tag from the resource
   group, grant its remediation identity `Tag Contributor`, and run
   remediation against one existing untagged resource.
3. Query cost grouped by `CostCenter` tag for the last month using
   `az costmanagement query`.
4. Check reservation utilization with `az consumption reservation-summary
   list` and decide whether any reservation needs resizing.
5. Create a management-group-scoped budget with a forecasted (not just
   actual) threshold alert.
