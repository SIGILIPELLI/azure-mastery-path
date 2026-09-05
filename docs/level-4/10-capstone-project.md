# 10 · Capstone Project

This capstone combines every Level 4 module into one enterprise
platform build-out: a management group hierarchy with governed landing
zones, a multi-cluster AKS mesh, hybrid servers projected via Arc, full
observability and load-tested autoscaling, a Sentinel-driven security
layer, an analytics data platform, and FinOps cost governance tying it all
together financially.

## Architecture

```
Tenant Root
└── alz
    ├── Platform
    │   ├── Management     (law-platform: Log Analytics + Sentinel)
    │   └── Connectivity   (hub VNet, Azure Firewall, VPN GW)
    ├── Landing Zones
    │   └── Corp
    │       └── sub-orders-platform
    │           ├── AKS cluster (mesh-enabled, Fleet-managed)
    │           ├── Arc-connected on-prem batch server
    │           ├── Synapse workspace (dedicated + serverless pools)
    │           └── Data Factory pipelines
    └── Sandbox

Governance: Policy initiative (tags, HTTPS-only, Defender Standard tier)
            assigned at alz-landingzones
Security:   Sentinel analytics rules + automation playbook
Cost:       Management-group budget with forecasted alert,
            reservation utilization tracked
```

## Step 1 — Management group hierarchy and subscription vending

```bash
az account management-group create --name alz --display-name "Enterprise-Scale"
az account management-group create --name alz-platform --parent alz
az account management-group create --name alz-landingzones --parent alz
az account management-group create --name alz-corp --parent alz-landingzones

az account management-group subscription add \
  --name alz-corp \
  --subscription "<orders-platform-subscription-id>"
```

## Step 2 — Baseline governance initiative

```bash
az policy set-definition create \
  --name capstone-baseline \
  --display-name "Capstone Baseline Initiative" \
  --definitions '[
    { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/require-cost-center-tag" },
    { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/deny-public-blob-access" },
    { "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/404c3081-a854-4457-ae30-26a93ef643f9" }
  ]'

az policy assignment create \
  --name capstone-baseline-assignment \
  --policy-set-definition capstone-baseline \
  --scope "/providers/Microsoft.Management/managementGroups/alz-landingzones"

az security pricing create --name KubernetesService --tier Standard
az security pricing create --name Arm --tier Standard
```

## Step 3 — AKS with mesh, autoscaling, and Fleet membership

```bash
az aks create \
  --resource-group rg-orders-platform \
  --name aks-orders-capstone \
  --zones 1 2 3 \
  --enable-cluster-autoscaler --min-count 2 --max-count 10 \
  --network-plugin azure \
  --generate-ssh-keys

az aks mesh enable --resource-group rg-orders-platform --name aks-orders-capstone

az fleet create --resource-group rg-orders-platform --name fleet-capstone
az fleet member create \
  --resource-group rg-orders-platform \
  --fleet-name fleet-capstone \
  --name member-orders \
  --member-cluster-id $(az aks show -g rg-orders-platform -n aks-orders-capstone --query id -o tsv)
```

## Step 4 — Arc-connect a hybrid batch server

```bash
az connectedmachine onboarding-script generate \
  --resource-group rg-orders-platform \
  --location eastus \
  --output-directory ./onboarding

# run generated script on the on-prem batch server, then confirm:
az connectedmachine show -g rg-orders-platform -n onprem-batch-01 --query status -o tsv
```

## Step 5 — Data platform for order analytics

```bash
az synapse workspace create \
  --resource-group rg-orders-platform \
  --name synw-orders-capstone \
  --storage-account stadls2orderscap \
  --file-system synapsefs \
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password "ReplaceWithARealSecret1!"

az datafactory create --resource-group rg-orders-platform --factory-name adf-orders-capstone

az datafactory pipeline create \
  --resource-group rg-orders-platform \
  --factory-name adf-orders-capstone \
  --pipeline-name pl-nightly-orders-etl \
  --pipeline '{"activities":[{"name":"CopyOrders","type":"Copy"}]}'
```

## Step 6 — Observability and load-tested autoscaling

```bash
az monitor app-insights component create \
  --resource-group rg-orders-platform \
  --app orders-capstone-insights \
  --workspace $(az monitor log-analytics workspace show -g rg-orders-platform -n law-platform --query id -o tsv)

az load create --resource-group rg-orders-platform --name alt-orders-capstone

az load test create \
  --resource-group rg-orders-platform \
  --load-test-resource alt-orders-capstone \
  --test-id orders-capstone-ramp \
  --load-test-config-file loadtest-config.yaml
```

After running the ramp test, correlate `PodReadyCount` against the load
timeline (as in Module 06) to confirm scale-up lag is acceptable before
calling the cluster production-ready.

## Step 7 — Sentinel and automated response

```bash
az sentinel workspace create --resource-group rg-orders-platform --workspace-name law-platform

az sentinel alert-rule create \
  --resource-group rg-orders-platform \
  --workspace-name law-platform \
  --rule-id capstone-deadletter-spike \
  --kind Scheduled \
  --display-name "Order fulfillment dead-letter spike" \
  --query-frequency PT10M --query-period PT10M --severity Medium

az sentinel automation-rule create \
  --resource-group rg-orders-platform \
  --workspace-name law-platform \
  --automation-rule-name auto-tag-deadletter-incident \
  --display-name "Tag dead-letter incidents for triage" \
  --order 1 \
  --triggering-logic '{"triggersOn":"Incidents","triggersWhen":"Created"}'
```

## Step 8 — FinOps close-out

```bash
az policy assignment create \
  --name enforce-cost-center-tag-capstone \
  --policy require-cost-center-tag \
  --scope "/subscriptions/<orders-platform-subscription-id>"

az consumption budget create \
  --budget-name budget-capstone \
  --amount 10000 --category Cost --time-grain Monthly \
  --start-date 2026-08-01 --end-date 2027-08-01 \
  --scope "/providers/Microsoft.Management/managementGroups/alz-corp" \
  --notifications '{"Forecasted_GreaterThan_90_Percent":{"enabled":true,"operator":"GreaterThan","threshold":90,"thresholdType":"Forecasted","contactEmails":["finops@example.com"]}}'
```

## Verifying the full build

```text
$ az account management-group subscription show --name alz-corp --subscription "<sub-id>" --query displayName
"Orders Platform"

$ az policy state summarize --management-group alz-landingzones --query "{compliant:results.nonCompliantResources}"
{ "compliant": 0 }

$ az connectedmachine show -g rg-orders-platform -n onprem-batch-01 --query status -o tsv
Connected

$ az sentinel alert-rule list -g rg-orders-platform --workspace-name law-platform --query "[].displayName"
["Order fulfillment dead-letter spike"]
```

## Design decisions worth defending

- **Why a Fleet Manager membership even for one cluster today?** Adding
  members later (a second region, blue/green cluster swap) is then a
  one-command join instead of retrofitting fleet tooling under load.
- **Why Arc for the batch server instead of migrating it to a VM in
  Azure?** The workload has hardware dependencies that make migration
  non-trivial short-term; Arc gets it under the same policy/monitoring
  umbrella as everything else without forcing that migration first.
- **Why a forecasted budget threshold, not just actual?** A forecasted
  alert gives days of lead time to intervene before the money is spent;
  an actual-spend alert only confirms the overrun already happened.

## How It Actually Works

This capstone's landing zone + AKS + service mesh + Sentinel + FinOps stack
is a composition of every control-plane mechanism covered across all four
levels operating on the same substrate: every resource, regardless of
which service it belongs to, is still created through one ARM request
pipeline (authenticate via Entra token → evaluate RBAC at every scope up
the management-group tree → evaluate Azure Policy → forward to the owning
resource provider) from Level 1's capstone; every network boundary,
whether a landing-zone hub, a service-mesh sidecar, or a Private Endpoint,
is still enforced by the same SDN/VFP packet-filtering substrate from
Level 1's networking module; and every piece of telemetry feeding Sentinel
and the FinOps dashboards is still landing in the identical Log Analytics/
Kusto workspace architecture from Level 1's Monitor module. Building this
capstone successfully is really evidence that these independently-learned
mechanisms compose correctly under one governance hierarchy, not that a
new mechanism has been introduced.

The genuinely new thing this capstone tests is **cross-cutting consistency
enforcement**: a management-group-scoped Policy assignment (Level 4,
Module 2) must correctly cascade down through the landing zone's
subscriptions to constrain what the AKS cluster's Bicep/Terraform module
(Level 3) is even allowed to deploy, while the service mesh's mTLS
certificates (Module 3) and Sentinel's Conditional Access-gated
Zero Trust policies (Module 7) both ultimately depend on the same Entra
token-issuance chokepoint from Module 1's `az login` — verifying the full
build means confirming that chokepoint's policies apply uniformly whether
the caller is a human via the Portal, a pipeline's service principal, or a
workload identity federated from AKS, because all three are just different
callers of the identical OAuth token endpoint.

## Stretch goals

1. Add a second AKS cluster in a different region, join it to the same
   Fleet, and configure a cross-cluster multi-primary mesh so
   `orders-api` in one cluster can call `payments-api` in the other over
   mTLS.
2. Build a Sentinel playbook that automatically opens a ticket (via a
   webhook to an external system) rather than just tagging the incident,
   and stage it behind manual approval before enabling full automation.
3. Add a serverless Synapse SQL pool query over partitioned order history
   in the data lake, and confirm via query stats that only the relevant
   date partitions are scanned.
4. Run a full load test against the capstone environment, capture the
   real HPA/cluster-autoscaler lag, and tune `--min-count` to close any
   gap that exceeds your target user-facing latency budget.
5. Produce a one-page FinOps report grouping cost by `CostCenter` tag
   across the whole `alz-corp` management group for the last month.
