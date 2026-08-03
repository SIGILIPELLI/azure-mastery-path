# 07 · Container Instances & Container Registry

[Module 04](04-aks-basics.md) deployed containers to AKS, using a public
image (`mcr.microsoft.com/...`). This module covers the other half:
building your **own** image, storing it in **Azure Container Registry
(ACR)**, and running it — either serverlessly with **Azure Container
Instances (ACI)** (no orchestrator at all) or on AKS pulling from your own
registry instead of a public one.

## Azure Container Registry

ACR is a private, Azure-hosted Docker registry — `docker push`/`pull`
target it exactly like Docker Hub, just with Azure RBAC controlling who
can push or pull.

```bash
az group create --name rg-aci-acr --location eastus

az acr create \
  --resource-group rg-aci-acr \
  --name acrdemo$RANDOM \
  --sku Basic \
  --admin-enabled false
```

`--admin-enabled false` disables the registry-wide admin username/password
login — RBAC (via managed identity or `az acr login`) is the modern,
auditable way in, and admin credentials are a shared secret with no audit
trail of *who* used them.

## Build and push an image

Given a simple `Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Option A — build locally, push to ACR:**

```bash
ACR_NAME=$(az acr list --resource-group rg-aci-acr --query "[0].name" --output tsv)

az acr login --name $ACR_NAME

docker build -t $ACR_NAME.azurecr.io/hello-app:v1 .
docker push $ACR_NAME.azurecr.io/hello-app:v1
```

**Option B — build in the cloud (`az acr build`)**, no local Docker
daemon required — useful in CI runners without Docker installed, or just
to offload the build:

```bash
az acr build \
  --registry $ACR_NAME \
  --image hello-app:v1 \
  .
```

`az acr build` uploads your build context, runs the build on ACR's own
build compute, and pushes the resulting image directly — one command
instead of build-then-push, and no local `docker` requirement at all.

```bash
az acr repository list --name $ACR_NAME --output table
az acr repository show-tags --name $ACR_NAME --repository hello-app --output table
```

**Gotcha:** ACR Basic tier has meaningfully less storage and throughput
than Standard/Premium — fine for learning, but a real CI pipeline pushing
frequently benefits from Standard, and Premium adds geo-replication
(useful if AKS clusters in multiple regions pull from the same registry).

## Run it serverlessly with Container Instances

**ACI** runs a single container (or a small group) with no cluster, no
orchestrator, billed per-second while running — the "just run this one
container" option, sitting between "no infrastructure" (App Service) and
"full orchestration" (AKS).

```bash
az acr update --name $ACR_NAME --anonymous-pull-enabled false

az container create \
  --resource-group rg-aci-acr \
  --name aci-hello \
  --image $ACR_NAME.azurecr.io/hello-app:v1 \
  --registry-login-server $ACR_NAME.azurecr.io \
  --acr-identity [system] \
  --cpu 1 \
  --memory 1.5 \
  --ports 80 \
  --ip-address Public \
  --dns-name-label hello-aci-demo
```

`--acr-identity [system]` gives the container group a system-assigned
managed identity used purely to authenticate the image pull from ACR — no
registry password stored anywhere, same principle as
[Module 06](06-managed-identities-key-vault.md) applied to pulling images
instead of reading secrets.

```bash
az container show \
  --resource-group rg-aci-acr \
  --name aci-hello \
  --query "{fqdn:ipAddress.fqdn, state:instanceView.state}" \
  --output table

curl http://hello-aci-demo.eastus.azurecontainer.io
```

```bash
az container logs --resource-group rg-aci-acr --name aci-hello
az container exec --resource-group rg-aci-acr --name aci-hello --exec-command "sh"
```

**Gotcha:** ACI containers don't restart on crash by default the way a
Kubernetes Deployment does (`--restart-policy` defaults to `Always` for the
*container process*, but the container **group** itself has no
self-healing across host failures, no rolling updates, and no built-in
load balancing across multiple instances) — for anything needing real
availability guarantees or more than one replica, that's what AKS or App
Service are for. ACI shines for short-lived jobs, batch tasks, and
dev/test — not as a production web tier replacement.

## ACI as a background job

A common ACI pattern is a one-shot task, not a long-running server:

```bash
az container create \
  --resource-group rg-aci-acr \
  --name aci-batch-job \
  --image $ACR_NAME.azurecr.io/report-generator:v1 \
  --registry-login-server $ACR_NAME.azurecr.io \
  --acr-identity [system] \
  --restart-policy Never \
  --cpu 2 \
  --memory 4
```

`--restart-policy Never` means the container runs once to completion and
stops — you're billed only for the seconds it actually ran, then it's
gone (unlike a Web App or AKS pod, which keeps a reserved slot even when
idle). Check its exit code:

```bash
az container show \
  --resource-group rg-aci-acr \
  --name aci-batch-job \
  --query "containers[0].instanceView.currentState" \
  --output json
```

## Connecting an AKS cluster to ACR

If [Module 04](04-aks-basics.md)'s cluster needs to pull your own images
instead of `mcr.microsoft.com`'s public ones:

```bash
az aks update \
  --resource-group rg-aks-demo \
  --name aks-demo \
  --attach-acr $ACR_NAME
```

This grants the AKS cluster's kubelet identity `AcrPull` on the registry
automatically — no image pull secret to create or rotate manually.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az acr create --sku Basic --admin-enabled false` | Create a private registry, RBAC-only access. |
| `az acr login --name` | Authenticate local `docker` to the registry. |
| `docker build/push` | Build and push an image (local Docker daemon). |
| `az acr build --registry --image .` | Build **and** push in the cloud, no local Docker needed. |
| `az acr repository list/show-tags` | Inspect what's stored in the registry. |
| `az container create --acr-identity [system]` | Run a container from ACR, pulled via managed identity. |
| `az container logs` / `exec` | View logs / shell into a running container instance. |
| `az aks update --attach-acr` | Grant an AKS cluster pull access to a registry. |

## Exercise

1. Create an ACR instance (Basic SKU, admin disabled) and build a small
   image using `az acr build` (no local Docker needed).
2. Run that image as a public-facing ACI container using
   `--acr-identity [system]`, and `curl` it at its `.azurecontainer.io`
   FQDN.
3. Run a second container with `--restart-policy Never` that just prints
   something and exits, and confirm its exit state with `az container
   show`.
4. Attach the ACR to the AKS cluster from [Module 04](04-aks-basics.md)
   with `az aks update --attach-acr`, and redeploy the `hello-aks`
   Deployment to instead reference your own image.
5. Delete the resource group when finished.
