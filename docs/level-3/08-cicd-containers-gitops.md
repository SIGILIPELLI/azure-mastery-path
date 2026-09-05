# 08 · CI/CD for Containers & GitOps

[Level 2, Module 05](../level-2/05-azure-devops-cicd.md) covered a
straightforward build-and-deploy pipeline. Container workloads on AKS
benefit from a different pattern: build and push an image in CI, then let a
**GitOps** controller reconcile the cluster's actual state to match a Git
repo, rather than a pipeline `kubectl apply`-ing directly.

## Building and pushing to ACR from a pipeline

```bash
az acr build \
  --registry acrplatformdemo \
  --image orders-api:$(git rev-parse --short HEAD) \
  --file Dockerfile .
```

`az acr build` runs the build **inside ACR's managed build service**
(no local Docker daemon required, works the same from a laptop or a
pipeline agent), and pushes automatically on success:

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]

stages:
- stage: Build
  jobs:
  - job: BuildAndPush
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'sc-acr-platform'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az acr build --registry acrplatformdemo \
            --image orders-api:$(Build.SourceVersion) \
            --file Dockerfile .
```

**Gotcha:** tagging images `:latest` in CI defeats rollback and makes it
impossible to know which commit is actually running — always tag with the
Git SHA (or a semantic version plus SHA) and let GitOps or a deployment
manifest reference the immutable tag, never `:latest`, in any environment
past local dev.

## GitOps with Flux on AKS

AKS has a first-class **GitOps extension** (Flux v2) so the cluster pulls
desired state from Git instead of a pipeline pushing to it:

```bash
az k8s-configuration flux create \
  --resource-group rg-aks-gitops \
  --cluster-name aks-gitops-cluster \
  --cluster-type managedClusters \
  --name flux-config \
  --namespace flux-system \
  --scope cluster \
  --url https://github.com/example-org/aks-manifests \
  --branch main \
  --kustomization name=apps path=./apps prune=true
```

```text
$ az k8s-configuration flux show -g rg-aks-gitops --cluster-name aks-gitops-cluster --cluster-type managedClusters -n flux-config --query complianceState -o tsv
Compliant
```

The pipeline's job shrinks to: build image, push to ACR, **update the image
tag in the Git manifest repo** (a commit, often via a small automation
step or a tool like Flux image automation) — Flux notices the Git change
and reconciles the cluster, not the pipeline directly.

**Gotcha:** with `prune=true`, deleting a manifest file from the Git repo
**deletes the corresponding resource from the cluster** on the next
reconciliation — this is the intended GitOps behavior (Git is the sole
source of truth) but catches teams off guard the first time someone
reorganizes the repo and unintentionally deletes a namespace's worth of
resources in production.

## Progressive delivery: canary via Flagger (concept)

Beyond plain rolling updates, a **canary** shifts a small percentage of
traffic to the new version, watches metrics (error rate, latency), and
automatically promotes or rolls back:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: orders-api
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders-api
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    stepWeight: 10
    maxWeight: 50
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
```

This shifts traffic in 10% steps up to 50%, checking a 99% success-rate
threshold every minute, and automatically rolls back if the metric fails
`threshold` (5) consecutive checks — no manual "watch the dashboard and
decide" step, which is the difference between progressive delivery and a
plain blue/green swap.

## CI/CD pipeline comparison

| | Push (pipeline `kubectl apply`) | GitOps (Flux/Argo CD) |
|---|---|---|
| **Source of truth** | Pipeline's last run | Git repo state |
| **Drift detection** | None — manual `kubectl` changes persist silently | Automatic — reconciled away |
| **Rollback** | Re-run an old pipeline | `git revert` |
| **Cluster credentials** | Pipeline needs cluster access | Only the in-cluster agent needs it |
| **Audit trail** | Pipeline run logs | Git history |

## How It Actually Works

**GitOps** inverts the normal CI/CD push model at the mechanical level: a
controller running *inside* the cluster (Flux or Argo CD, both built on
Kubernetes' own controller-runtime pattern) continuously polls a Git
repository for the desired-state manifests, diffs them against the
cluster's actual live objects via the Kubernetes API server, and applies
any delta itself — your pipeline's job is only to update the manifests in
Git (usually by bumping an image tag after a successful build/push to
ACR), never to run `kubectl apply` against the cluster directly. This is
the concrete reason GitOps clusters can run with no inbound deployment
credentials or open network path from CI at all: the pull-based controller
inside the cluster is the only thing that ever needs API server access to
write changes, reversing the trust direction of a traditional push
pipeline.

Under a traditional push pipeline, `kubectl apply` (or a pipeline's
equivalent Helm/kubectl task) authenticates to the AKS API server using a
kubeconfig token derived from either an Entra-integrated RBAC binding or a
cluster-issued service account token, and the API server validates that
token against Kubernetes RBAC role bindings before admitting the request —
this is a separate authorization layer from Azure RBAC on the AKS resource
itself (which only controls who can manage the *cluster resource*, e.g.
via `az aks` commands), which is why a user can have full Azure RBAC
Contributor on an AKS cluster and still be denied by Kubernetes RBAC when
trying to deploy a workload into it, or vice versa.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az acr build --registry --image` | Build and push an image via ACR's managed build. |
| `az k8s-configuration flux create --kustomization prune=true` | Wire a cluster to reconcile from a Git repo. |
| `az k8s-configuration flux show --query complianceState` | Check whether the cluster matches Git. |
| `git rev-parse --short HEAD` | Get a stable, unique image tag for a commit. |
| `Canary` (Flagger CRD) | Define automated, metric-gated progressive rollout. |

## Exercise

1. Use `az acr build` to build and push an image tagged with the current
   Git SHA — never `:latest`.
2. Set up the AKS GitOps extension pointed at a manifest repo, and confirm
   `complianceState` reports `Compliant`.
3. Delete a manifest file from the Git repo (in a test namespace) and
   observe Flux prune the corresponding resource from the cluster.
4. Read through a Flagger `Canary` spec and explain what happens if the
   success-rate metric fails 3 consecutive checks with `threshold: 5`.
5. Delete the resource group when finished.
