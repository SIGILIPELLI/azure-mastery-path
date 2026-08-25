# 10 · Project — Highly Available Microservices

This project combines everything from Level 3: networking, AKS, event-driven
messaging, API Management, governance, HA/DR, GitOps, and cost controls,
into one architecture — a highly available order-processing platform
running two microservices across a hub-and-spoke network with automated
deployment and monitoring guardrails.

## Architecture

```
                         ┌─────────────────────┐
  Internet ──► APIM ────►│   AKS (2+ zones)     │
                         │  orders-api          │
                         │  fulfillment-worker  │
                         └──────────┬───────────┘
                                    │
                          ┌─────────┴─────────┐
                          ▼                   ▼
                  Service Bus queue    Azure SQL (zone-redundant,
                  (order-fulfillment)   failover group to westus2)
                          │
                          ▼
                  fulfillment-worker (KEDA-scaled on queue depth)

  Hub VNet (Azure Firewall, VPN GW) ── peered ── Spoke VNet (AKS, SQL private endpoint)
  Azure Policy: initiative enforcing HTTPS-only, no public blob access
  Defender for Cloud: Standard tier on AKS + SQL
  Flux GitOps: cluster reconciled from a manifests repo
  Budget: subscription-level alert at 80%
```

## Step 1 — Network foundation

```bash
az group create --name rg-orders-platform --location eastus

az network vnet create \
  --resource-group rg-orders-platform \
  --name vnet-hub \
  --address-prefix 10.0.0.0/16 \
  --subnet-name GatewaySubnet \
  --subnet-prefix 10.0.255.0/27

az network vnet create \
  --resource-group rg-orders-platform \
  --name vnet-spoke-orders \
  --address-prefix 10.1.0.0/16 \
  --subnet-name snet-aks \
  --subnet-prefix 10.1.0.0/22

az network vnet peering create \
  --resource-group rg-orders-platform --name hub-to-spoke \
  --vnet-name vnet-hub --remote-vnet vnet-spoke-orders \
  --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit

az network vnet peering create \
  --resource-group rg-orders-platform --name spoke-to-hub \
  --vnet-name vnet-spoke-orders --remote-vnet vnet-hub \
  --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways
```

## Step 2 — AKS with zone spread and cluster autoscaler

```bash
az aks create \
  --resource-group rg-orders-platform \
  --name aks-orders \
  --vnet-subnet-id $(az network vnet subnet show -g rg-orders-platform --vnet-name vnet-spoke-orders -n snet-aks --query id -o tsv) \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 8 \
  --network-plugin azure \
  --generate-ssh-keys
```

## Step 3 — Data and messaging tier

```bash
az sql server create --resource-group rg-orders-platform --name sql-orders-primary --admin-user sqladmin --admin-password "ReplaceWithARealSecret1!"
az sql db create --resource-group rg-orders-platform --server sql-orders-primary --name db-orders --zone-redundant true --edition Premium

az sql server create --resource-group rg-orders-platform --name sql-orders-secondary --admin-user sqladmin --admin-password "ReplaceWithARealSecret1!" --location westus2
az sql failover-group create --resource-group rg-orders-platform --server sql-orders-primary --name fg-orders --partner-server sql-orders-secondary --add-db db-orders

az servicebus namespace create --resource-group rg-orders-platform --name sb-orders-ns --sku Standard
az servicebus queue create --resource-group rg-orders-platform --namespace-name sb-orders-ns --name q-order-fulfillment --max-delivery-count 5
```

## Step 4 — API Management façade

```bash
az apim create --resource-group rg-orders-platform --name apim-orders --publisher-name "Platform Team" --publisher-email platform@example.com --sku-name Developer --no-wait

az apim api import \
  --resource-group rg-orders-platform --service-name apim-orders \
  --api-id orders-api --path orders --specification-format OpenApi \
  --specification-url https://raw.githubusercontent.com/example-org/orders-api/main/openapi.json \
  --display-name "Orders API"
```

## Step 5 — GitOps deployment and KEDA autoscaling

```bash
az k8s-configuration flux create \
  --resource-group rg-orders-platform --cluster-name aks-orders --cluster-type managedClusters \
  --name flux-config --namespace flux-system --scope cluster \
  --url https://github.com/example-org/orders-manifests --branch main \
  --kustomization name=apps path=./apps prune=true
```

```yaml
# apps/keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: fulfillment-worker-scaler
spec:
  scaleTargetRef:
    name: fulfillment-worker
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: q-order-fulfillment
      messageCount: "5"
```

## Step 6 — Governance and cost guardrails

```bash
az policy assignment create \
  --name deny-public-blob-orders \
  --policy deny-public-blob-access \
  --scope /subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-orders-platform

az security pricing create --name SqlServers --tier Standard
az security pricing create --name KubernetesService --tier Standard

az consumption budget create \
  --budget-name budget-orders-platform --amount 2000 --category Cost \
  --time-grain Monthly --start-date 2026-08-01 --end-date 2027-08-01 \
  --resource-group rg-orders-platform \
  --notifications '{"Actual_GreaterThan_80_Percent":{"enabled":true,"operator":"GreaterThan","threshold":80,"contactEmails":["platform@example.com"]}}'
```

## Verifying the whole path

```text
$ curl https://apim-orders.azure-api.net/orders/health -H "Ocp-Apim-Subscription-Key: <key>"
{"status":"healthy","zone":"eastus-2"}

$ az servicebus queue show -g rg-orders-platform --namespace-name sb-orders-ns -n q-order-fulfillment --query messageCount -o tsv
0

$ kubectl get scaledobject fulfillment-worker-scaler
NAME                        SCALETARGETKIND      SCALETARGETNAME       MIN   MAX   READY
fulfillment-worker-scaler   apps/v1.Deployment    fulfillment-worker    1     20    True
```

## Design decisions worth defending

- **Why Service Bus, not Event Grid, between orders-api and
  fulfillment-worker?** Order fulfillment is a durable workflow that must
  not be dropped or double-processed casually — Service Bus's
  dead-lettering and delivery-count control fit better than Event Grid's
  fire-and-forget model.
- **Why a SQL failover group instead of just zone-redundant?**
  Zone-redundancy protects against a datacenter failure within `eastus`;
  the failover group protects against losing the entire `eastus` region.
- **Why KEDA instead of a plain HPA?** Queue depth isn't a CPU/memory
  metric — KEDA's Service Bus scaler reacts to the actual backlog driving
  the work, which is the signal that matters for a queue-consuming worker.

## Stretch goals

1. Add Azure Firewall in the hub VNet and force spoke egress through it via
   a user-defined route, instead of direct internet egress from AKS nodes.
2. Add a canary rollout (Flagger) for `orders-api` gated on a
   request-success-rate metric instead of a plain rolling update.
3. Add an Azure Monitor alert rule that pages when the Service Bus
   dead-letter queue depth exceeds a threshold, not just the main queue.
4. Run a simulated regional failure: fail over the SQL failover group to
   `westus2` and measure actual RTO against your documented target.
5. Add a second APIM product ("partner") with stricter rate limits and
   require subscription approval, separate from the default product.
