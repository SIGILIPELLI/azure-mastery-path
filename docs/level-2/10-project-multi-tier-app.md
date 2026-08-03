# 10 · Project — Multi-Tier Web Application

This project combines almost everything from Level 2 into one system: a
containerized API on **AKS**, behind an **Application Gateway**, backed by
**Cosmos DB**, with secrets in **Key Vault** via a managed identity, and a
static frontend on **App Service**. It's deliberately more infrastructure
than the [Level 1 capstone](../level-1/10-capstone-project.md) — the point
is practicing how these pieces actually connect, not just each in
isolation.

## Architecture

```
Browser
  │
  ▼
Azure App Service (Static frontend, Linux, Node/Python runtime)
  │  fetch("https://api.example.com/orders")
  ▼
Application Gateway (WAF_v2, public IP, TLS termination)
  │  routes /api/* to the AKS backend pool
  ▼
AKS cluster (LoadBalancer Service → Deployment of order-api pods)
  │  order-api pod reads COSMOS_KEY via Key Vault reference,
  │  uses its pod identity to authenticate — no stored secret
  ▼
Azure Cosmos DB (SQL API, partitioned by /customerId)
```

Everything talks over a shared VNet where it makes sense (App Gateway and
AKS), and Cosmos DB is reached through a private endpoint rather than the
public internet — the same private-endpoint pattern from
[Module 03](03-networking-deep-dive.md), applied to a database instead of
Blob Storage.

## Step 1 — Resource group, VNet, and subnets

```bash
az group create --name rg-multitier --location eastus

az network vnet create --resource-group rg-multitier --name vnet-multitier \
  --address-prefix 10.20.0.0/16 \
  --subnet-name subnet-appgw --subnet-prefix 10.20.1.0/24

az network vnet subnet create --resource-group rg-multitier \
  --vnet-name vnet-multitier --name subnet-aks --address-prefix 10.20.2.0/23

az network vnet subnet create --resource-group rg-multitier \
  --vnet-name vnet-multitier --name subnet-privatelink --address-prefix 10.20.4.0/24
```

`subnet-aks` gets a `/23` (510 usable addresses) since Azure CNI networking
(AKS's default) assigns a real VNet IP to every pod, not just every node —
undersizing this subnet is a common reason clusters can't scale past a
surprisingly small pod count.

## Step 2 — Cosmos DB with a private endpoint

```bash
az cosmosdb create \
  --resource-group rg-multitier \
  --name cosmos-multitier$RANDOM \
  --locations regionName=eastus failoverPriority=0 \
  --default-consistency-level Session \
  --public-network-access Disabled

COSMOS_ACCOUNT=$(az cosmosdb list --resource-group rg-multitier --query "[0].name" --output tsv)

az cosmosdb sql database create \
  --account-name $COSMOS_ACCOUNT --resource-group rg-multitier --name ordersdb
az cosmosdb sql container create \
  --account-name $COSMOS_ACCOUNT --resource-group rg-multitier \
  --database-name ordersdb --name orders \
  --partition-key-path "/customerId" --throughput 400

az network private-endpoint create \
  --resource-group rg-multitier --name pe-cosmos \
  --vnet-name vnet-multitier --subnet subnet-privatelink \
  --private-connection-resource-id $(az cosmosdb show --name $COSMOS_ACCOUNT --resource-group rg-multitier --query id -o tsv) \
  --group-id Sql --connection-name conn-cosmos

az network private-dns zone create --resource-group rg-multitier \
  --name privatelink.documents.azure.com

az network private-dns link vnet create --resource-group rg-multitier \
  --zone-name privatelink.documents.azure.com --name dns-link-multitier \
  --virtual-network vnet-multitier --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group rg-multitier --endpoint-name pe-cosmos --name zone-group \
  --private-dns-zone privatelink.documents.azure.com --zone-name documents
```

`--public-network-access Disabled` means Cosmos DB is reachable **only**
through the private endpoint — unreachable from outside the VNet by
design. Same DNS-resolution gotcha as [Module 03](03-networking-deep-dive.md):
without the private DNS zone steps above, nothing inside the VNet resolves
the Cosmos DB hostname correctly either.

## Step 3 — Key Vault, secret, and managed identity

```bash
az keyvault create --name kv-multitier$RANDOM --resource-group rg-multitier \
  --enable-rbac-authorization true
KEYVAULT=$(az keyvault list --resource-group rg-multitier --query "[0].name" -o tsv)

COSMOS_KEY=$(az cosmosdb keys list --name $COSMOS_ACCOUNT --resource-group rg-multitier --query primaryMasterKey -o tsv)
az keyvault secret set --vault-name $KEYVAULT --name CosmosDbKey --value "$COSMOS_KEY"

# User-assigned identity: order-api's pod uses this via workload identity below.
# Made independent of any single node/pod so it outlives cluster churn.
az identity create --name id-order-api --resource-group rg-multitier
IDENTITY_CLIENT_ID=$(az identity show --name id-order-api --resource-group rg-multitier --query clientId -o tsv)
IDENTITY_PRINCIPAL_ID=$(az identity show --name id-order-api --resource-group rg-multitier --query principalId -o tsv)

az role assignment create --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name $KEYVAULT --query id -o tsv)
```

## Step 4 — Build the order-api image and push to ACR

```bash
az acr create --resource-group rg-multitier --name acrmultitier$RANDOM --sku Basic --admin-enabled false
ACR_NAME=$(az acr list --resource-group rg-multitier --query "[0].name" --output tsv)

az acr build --registry $ACR_NAME --image order-api:v1 .
```

`order-api`'s app code uses `DefaultAzureCredential` exactly as in
[Module 06](06-managed-identities-key-vault.md), reading Cosmos DB's
endpoint as a plain env var and its key from Key Vault at startup — no
secret baked into the image.

## Step 5 — AKS cluster with workload identity, attached to ACR and the VNet

```bash
az aks create \
  --resource-group rg-multitier --name aks-multitier \
  --node-count 2 --node-vm-size Standard_B2s \
  --vnet-subnet-id $(az network vnet subnet show --resource-group rg-multitier --vnet-name vnet-multitier --name subnet-aks --query id -o tsv) \
  --network-plugin azure \
  --enable-oidc-issuer --enable-workload-identity \
  --attach-acr $ACR_NAME --generate-ssh-keys

az aks get-credentials --resource-group rg-multitier --name aks-multitier
```

**Workload identity** federates a Kubernetes service account to the Azure
AD identity `id-order-api`, so pods authenticate to Azure as that identity
without any node-level secret — the AKS equivalent of the pod-identity
pattern from [Module 06](06-managed-identities-key-vault.md):

```bash
AKS_OIDC_ISSUER=$(az aks show --resource-group rg-multitier --name aks-multitier --query "oidcIssuerProfile.issuerUrl" -o tsv)

az identity federated-credential create --name federated-order-api \
  --identity-name id-order-api --resource-group rg-multitier \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:default:order-api-sa \
  --audience api://AzureADTokenExchange
```

`order-api-sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-api-sa
  annotations:
    azure.workload.identity/client-id: "<IDENTITY_CLIENT_ID>"
```

## Step 6 — Deploy order-api to AKS

`order-api-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 3
  selector:
    matchLabels: { app: order-api }
  template:
    metadata:
      labels:
        app: order-api
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: order-api-sa
      containers:
        - name: order-api
          image: <ACR_NAME>.azurecr.io/order-api:v1
          ports: [{ containerPort: 8080 }]
          env:
            - name: COSMOS_ENDPOINT
              value: "<COSMOS_ENDPOINT>"
            - name: KEY_VAULT_URL
              value: "https://<KEYVAULT>.vault.azure.net/"
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits: { cpu: "500m", memory: "512Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: order-api-svc
spec:
  type: ClusterIP
  ports: [{ port: 80, targetPort: 8080 }]
  selector: { app: order-api }
```

```bash
kubectl apply -f order-api-sa.yaml
kubectl apply -f order-api-deployment.yaml
kubectl get pods -w
```

Note the Service is `ClusterIP`, not `LoadBalancer` — traffic reaches
`order-api` **only** through the Application Gateway in the next step,
never with its own public IP.

## Step 7 — Application Gateway in front of AKS

```bash
az network public-ip create --resource-group rg-multitier --name pip-appgw --sku Standard

# Manual/conceptual version — pod IPs as backend pool targets. AGIC (stretch
# goal below) is what keeps this pool in sync automatically as pods scale.
az network application-gateway create \
  --resource-group rg-multitier --name appgw-multitier \
  --sku WAF_v2 --capacity 2 \
  --vnet-name vnet-multitier --subnet subnet-appgw \
  --public-ip-address pip-appgw --priority 100
```

**Gotcha:** wiring App Gateway directly to individual pod IPs is fragile —
pod IPs change on every reschedule. In a real deployment you'd install
**AGIC (Application Gateway Ingress Controller)** so Kubernetes `Ingress`
objects manage the backend pool automatically; the manual `--servers` flag
approach from [Module 03](03-networking-deep-dive.md) is shown here for
clarity but AGIC is what you'd actually run — see Stretch Goals.

## Step 8 — App Service frontend, and end-to-end verification

```bash
az appservice plan create --name asp-multitier --resource-group rg-multitier --sku B1 --is-linux

az webapp create --name webapp-multitier$RANDOM --resource-group rg-multitier \
  --plan asp-multitier --runtime "NODE:20-lts"
WEBAPP=$(az webapp list --resource-group rg-multitier --query "[0].name" -o tsv)

az webapp config appsettings set --name $WEBAPP --resource-group rg-multitier \
  --settings API_BASE_URL="https://$(az network public-ip show --resource-group rg-multitier --name pip-appgw --query ipAddress -o tsv)"
```

The frontend's `fetch(process.env.API_BASE_URL + "/orders")` now traverses
the full chain: App Service → public internet → App Gateway (WAF
inspection, TLS termination) → AKS Service → `order-api` pod → private
endpoint → Cosmos DB. Verify it:

```bash
curl -X POST "https://<appgw-public-ip>/api/orders" \
  -H "Content-Type: application/json" \
  -d '{"customerId": "cust-1", "item": "widget"}'

curl "https://<appgw-public-ip>/api/orders?customerId=cust-1"
```

Confirm the round trip properly: the WAF logs show the request passed
inspection (`az monitor diagnostic-settings` on the App Gateway, or the
portal's WAF log), `kubectl logs deployment/order-api` shows the pod
handled it, and the item is actually in Cosmos DB (query it, or check the
Data Explorer in the portal) — not just returned from an in-memory mock.

## Step 9 — Teardown

```bash
az group delete --name rg-multitier --yes --no-wait
az group show --name rg-multitier --output table   # confirm it's gone
az group list --output table                        # sanity-check nothing else is left running
```

## Stretch goals

- **Install AGIC properly** instead of the manual backend-pool step —
  deploy the AGIC add-on (`az aks enable-addons --addons ingress-appgw
  --appgw-id <appgw-resource-id>`) and replace the raw Service with a
  Kubernetes `Ingress` object; confirm the App Gateway backend pool
  updates itself automatically when you scale `order-api`'s replicas.
- **Add Azure Monitor / Log Analytics** across all four tiers (App
  Service, App Gateway, AKS via Container Insights, Cosmos DB
  diagnostics) into one workspace, and write a single query joining
  request latency across the App Gateway access log and Cosmos DB's RU
  charge for the same request.
- **Add a CI/CD pipeline** (either flavor from
  [Module 05](05-azure-devops-cicd.md)) that builds `order-api`,
  `az acr build`s it, and does a rolling `kubectl set image` deploy on
  every push to `main`.
- **Add autoscaling everywhere**: an HPA on `order-api` (CPU-based, from
  [Module 04](04-aks-basics.md)), the AKS cluster autoscaler, and Cosmos
  DB autoscale throughput — then load-test with a tool like `hey` or `k6`
  and watch all three layers respond independently.
- **Multi-region**: add a second Cosmos DB region with
  `az cosmosdb update --locations`, and reason through (without
  necessarily building) what else would need to be multi-region too
  (App Gateway is regional; Front Door, covered conceptually in
  [Module 03](03-networking-deep-dive.md)'s comparison table, is the
  piece that would actually make the whole stack multi-region).

## Cheat sheet

| Command | Purpose |
|---|---|
| `az network private-endpoint create --group-id Sql` | Private endpoint for Cosmos DB. |
| `az aks create --enable-oidc-issuer --enable-workload-identity` | AKS cluster ready for federated pod identities. |
| `az identity federated-credential create --subject system:serviceaccount:...` | Bind a K8s service account to an Azure AD identity. |
| `azure.workload.identity/use: "true"` pod label | Opts a pod into workload identity token injection. |
| `az network application-gateway create --sku WAF_v2` | Layer 7 entry point with WAF, in front of AKS. |
| `az aks enable-addons --addons ingress-appgw` | Install AGIC to sync Ingress objects → App Gateway backend pool. |
| `az group delete --yes --no-wait` | Tear down the whole project. |

## Exercise

1. Stand up the full chain (Cosmos DB with private endpoint → Key Vault →
   AKS with workload identity → App Gateway → App Service frontend) and
   get one order to round-trip end-to-end via `curl`.
2. Confirm the Cosmos DB account genuinely has no public network access
   (`az cosmosdb show --query publicNetworkAccess`) and that the only way
   your `order-api` pod reaches it is through the private endpoint.
3. Kill an `order-api` pod (`kubectl delete pod <name>`) mid-traffic and
   confirm the Deployment replaces it and the App Gateway keeps serving
   requests from the remaining replicas.
4. Pick at least one stretch goal and implement it.
5. Run the full teardown and confirm with `az group list` that nothing
   from this project is left running.
