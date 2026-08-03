# 04 · Azure Kubernetes Service (AKS) Basics

**Azure Kubernetes Service (AKS)** is Azure's managed Kubernetes control
plane — Azure runs and patches the API server and etcd for you (free of
charge); you manage and pay only for the worker node VMs. This module
covers standing up a cluster, deploying a containerized app, and basic
scaling — [Level 2, Module 07](07-aci-acr.md) covers the container image
side (building/pushing to ACR) that feeds into this.

## Create a cluster

```bash
az group create --name rg-aks-demo --location eastus

az aks create \
  --resource-group rg-aks-demo \
  --name aks-demo \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --enable-managed-identity

az aks get-credentials \
  --resource-group rg-aks-demo \
  --name aks-demo
```

`az aks get-credentials` merges the cluster's connection details into your
local `~/.kube/config`, so `kubectl` immediately targets the right cluster.
`--enable-managed-identity` gives the cluster its own identity for talking
to other Azure resources (like ACR) without a stored secret — the same
managed identity concept covered in
[Module 06](06-managed-identities-key-vault.md).

**Gotcha:** cluster creation takes 5-10 minutes, and each node is a real VM
billed at its normal compute rate the moment it exists — a 2-node
`Standard_B2s` cluster keeps costing money whether or not you deploy
anything to it, until the cluster (or resource group) is deleted.

```bash
kubectl get nodes
# NAME                                STATUS   ROLES   AGE   VERSION
# aks-nodepool1-12345678-vmss000000   Ready    <none>  4m    v1.29.2
# aks-nodepool1-12345678-vmss000001   Ready    <none>  4m    v1.29.2
```

## Deploy a containerized workload

Kubernetes objects are usually described declaratively in YAML and applied
with `kubectl apply` — the same "declare desired state, let the system
reconcile" idea as Bicep, just for a different control plane.

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-aks
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-aks
  template:
    metadata:
      labels:
        app: hello-aks
    spec:
      containers:
        - name: hello-aks
          image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-aks-svc
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: hello-aks
```

```bash
kubectl apply -f deployment.yaml

kubectl get deployments
kubectl get pods
kubectl get service hello-aks-svc
# Watch EXTERNAL-IP go from <pending> to a real public IP
```

The **Deployment** keeps 3 pod replicas running (restarting any that
crash); the **Service** of `type: LoadBalancer` provisions an actual Azure
Load Balancer with a public IP in front of them, using the same Load
Balancer resource from [Module 03](03-networking-deep-dive.md) under the
hood — Kubernetes and Azure networking meet at this point.

**Gotcha:** `resources.requests`/`limits` aren't optional in any real
cluster — without them, one greedy pod can starve every other pod on the
same node, and the scheduler can't make sensible bin-packing decisions.
Always set them, even generously, from the first deployment.

## Scaling

**Manual pod scaling** changes replica count immediately:

```bash
kubectl scale deployment hello-aks --replicas=5
```

**Horizontal Pod Autoscaler (HPA)** adjusts replica count automatically
based on observed CPU/memory:

```bash
kubectl autoscale deployment hello-aks --cpu-percent=70 --min=2 --max=10

kubectl get hpa
```

**Cluster autoscaler** (node-level, not pod-level) adds or removes *nodes*
when pods can't be scheduled due to insufficient capacity:

```bash
az aks update \
  --resource-group rg-aks-demo \
  --name aks-demo \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

**Gotcha:** HPA and the cluster autoscaler solve different problems and
you usually want both — HPA reacts fast by adding pods to existing node
capacity; the cluster autoscaler reacts slower (a new node takes a minute
or two to provision and join) to add capacity when there's no room left
for HPA to add more pods.

## Logs, exec, and troubleshooting

```bash
kubectl logs deployment/hello-aks
kubectl logs -f pod/<pod-name>          # follow/tail
kubectl exec -it pod/<pod-name> -- sh   # shell into a running container
kubectl describe pod <pod-name>         # events: scheduling failures, image pull errors, OOMKills
```

`kubectl describe` is usually the first thing to run when a pod is stuck
`Pending` or `CrashLoopBackOff` — the **Events** section at the bottom
tells you why (can't pull the image, no node has enough CPU/memory free,
liveness probe failing, and so on).

## Cheat sheet

| Command | Purpose |
|---|---|
| `az aks create --node-count --generate-ssh-keys` | Create an AKS cluster. |
| `az aks get-credentials` | Merge cluster access into local `~/.kube/config`. |
| `kubectl apply -f <file>.yaml` | Create/update objects from a manifest. |
| `kubectl get nodes/pods/deployments/service` | List cluster objects. |
| `kubectl scale deployment --replicas` | Manually change replica count. |
| `kubectl autoscale deployment --cpu-percent --min --max` | Create an HPA (pod-level autoscale). |
| `az aks update --enable-cluster-autoscaler --min-count --max-count` | Enable node-level autoscale. |
| `kubectl logs` / `exec -it` / `describe pod` | Debug a running or failing pod. |

## Exercise

1. Create a 2-node AKS cluster and confirm `kubectl get nodes` shows both
   as `Ready`.
2. Deploy the `hello-aks` Deployment + LoadBalancer Service above, and
   confirm you can `curl` the app at its external IP once it's assigned.
3. Scale to 5 replicas manually, then remove the manual scale and instead
   create an HPA with `min=2 max=10` targeting 70% CPU.
4. Deliberately request more CPU than any node has free in a test pod, and
   use `kubectl describe pod` to find the scheduling failure event
   explaining why it's stuck `Pending`.
5. Delete the resource group when finished.
