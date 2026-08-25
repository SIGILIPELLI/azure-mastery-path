# 09 · Cost Management & Optimization

Provisioning resources without a cost-review habit is how a dev
subscription quietly turns into a five-figure monthly bill. This module
covers reading Azure's cost data programmatically, setting budgets and
alerts, and the concrete levers (reservations, right-sizing, autoscaling
tiers, spot) for reducing spend without necessarily reducing capability.
No specific dollar figures below — pricing changes by SKU, region, and
your agreement type, so always check current pricing in the Azure Pricing
Calculator or `az` cost APIs for your own subscription.

## Reading cost data

```bash
az consumption usage list \
  --start-date 2026-08-01 \
  --end-date 2026-08-25 \
  --query "[].{resource:instanceName, cost:pretaxCost}" \
  -o table
```

```bash
az costmanagement query \
  --type Usage \
  --scope "/subscriptions/$(az account show --query id -o tsv)" \
  --dataset-granularity Daily \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --timeframe MonthToDate
```

**Gotcha:** `az consumption usage list` reflects **billed** usage with a
lag of up to 24-48 hours — costs incurred today won't show up querying
today, which surprises people expecting real-time spend tracking; use
Azure Monitor metrics or resource-level quotas for anything needing
near-real-time signal, and cost APIs for accurate historical totals.

## Budgets and alerts

```bash
az consumption budget create \
  --budget-name budget-dev-monthly \
  --amount 500 \
  --category Cost \
  --time-grain Monthly \
  --start-date 2026-08-01 \
  --end-date 2027-08-01 \
  --resource-group rg-dev \
  --notifications '{
    "Actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["platform@example.com"]
    }
  }'
```

A budget alert **notifies**; it does not stop spend on its own — pairing it
with an Action Group that triggers an Automation runbook (e.g. stop
non-prod VMs) is the common pattern for teams that want a hard cap enforced
automatically rather than relying on someone reading an email.

**Gotcha:** budgets scoped to a resource group don't catch spend from
resources created in the *wrong* resource group by mistake — a genuinely
effective budget strategy usually needs a subscription-level budget as a
backstop in addition to per-team resource-group budgets.

## Right-sizing with Azure Advisor

```bash
az advisor recommendation list \
  --category Cost \
  --query "[].{resource:impactedValue, problem:shortDescription.problem, solution:shortDescription.solution}" \
  -o table
```

```text
Resource         Problem                              Solution
---------------  -----------------------------------  --------------------------------
vm-batch-worker  Low utilization detected             Resize or shut down the VM
vm-legacy-app    Underutilized VM (avg CPU < 5%)       Consider a smaller SKU
```

Advisor's cost recommendations are based on **rolling averages** (often
7-14 days) — a VM that's idle most of the time but spikes hard once a
month (e.g. a monthly batch job) shows as "underutilized" even though
resizing it would break that job; always sanity-check a recommendation
against the resource's actual purpose before acting.

## Reservations and savings plans

- A **Reserved Instance** commits to a specific VM SKU/region for 1 or 3
  years for a lower effective rate than pay-as-you-go.
- A **Savings Plan** commits to an hourly *spend* amount (not a specific
  SKU) and applies across compute services more flexibly.

```bash
az reservations reservation-order list --query "[].{name:displayName, state:provisioningState}" -o table
```

**Gotcha:** reservations don't automatically apply to just any VM of the
matching size — scope matters (shared across the billing account vs.
scoped to one subscription/resource group), and a reservation purchased
with the wrong scope sits unused while pay-as-you-go rates keep being
charged, with no error or warning that it's not being consumed.

## Spot VMs and autoscale tiers

```bash
az vm create \
  --resource-group rg-batch \
  --name vm-spot-worker \
  --image Ubuntu2204 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1
```

Spot VMs run at a deep discount off pay-as-you-go pricing but can be
**evicted with as little as 30 seconds' notice** when Azure needs the
capacity back — appropriate for stateless batch/CI workers that checkpoint
or restart cleanly, never for anything holding state or serving live
traffic without a non-spot fallback path.

**Gotcha:** `--max-price -1` means "never evict for price, only for
capacity" — setting an actual price cap adds a *second* eviction trigger
(price exceeds your cap) on top of capacity-based eviction, which for most
workloads just increases eviction frequency without meaningfully protecting
against a cost spike, since eviction happens either way.

## Cost lever comparison

| Lever | Savings | Commitment | Risk |
|---|---|---|---|
| Reserved Instances | High (multi-year discount) | 1-3 years, specific SKU/region | Unused if workload changes |
| Savings Plans | Moderate-high | 1-3 years, flexible SKU | Lower risk than RIs |
| Spot VMs | Very high | None | Eviction with short notice |
| Right-sizing | Moderate | None | Requires ongoing monitoring |
| Autoscaling | Moderate | None | Needs correct min/max tuning |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az costmanagement query` | Query aggregated cost data programmatically. |
| `az consumption budget create --notifications` | Create a budget with threshold alerts. |
| `az advisor recommendation list --category Cost` | List right-sizing and cost recommendations. |
| `az reservations reservation-order list` | Check existing reservation utilization. |
| `az vm create --priority Spot --max-price -1` | Create a Spot VM (capacity-based eviction only). |

## Exercise

1. Query month-to-date cost for your subscription with
   `az costmanagement query` and identify the top 3 resources by spend.
2. Create a resource-group-scoped budget with an 80% threshold alert.
3. Run `az advisor recommendation list --category Cost` and, for one
   result, decide whether it's safe to act on given the resource's actual
   purpose.
4. Create a Spot VM with `--eviction-policy Deallocate` and explain when
   you would (and would not) use Spot for a real workload.
5. Delete the resource group when finished.
