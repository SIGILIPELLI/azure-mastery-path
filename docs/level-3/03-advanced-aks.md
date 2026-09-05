# 03 · Advanced AKS (Helm, Ingress, Autoscaling)

[Level 2, Module 04](../level-2/04-aks-basics.md) covered creating a
cluster and deploying a plain manifest. This module covers running AKS like
production: installing charts with **Helm**, exposing services through an
**ingress controller**, and scaling both pods and nodes automatically.

## Helm basics

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-basic \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

```text
$ helm list -n ingress-basic
NAME            NAMESPACE       REVISION  STATUS    CHART                       APP VERSION
ingress-nginx   ingress-basic   1         deployed  ingress-nginx-4.10.0        1.10.0
```

Helm tracks each install as a **release** with revision history — upgrades
are `helm upgrade`, and a bad rollout rolls back cleanly:

```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-basic \
  --set controller.replicaCount=3

helm rollback ingress-nginx 1 --namespace ingress-basic
```

**Gotcha:** `helm install` on a chart already partially applied via `kubectl
apply` (common when migrating an existing manifest-based deployment to
Helm) produces "resource already exists and is not managed by Helm"
errors — either `kubectl delete` the old resources first or adopt them with
`helm adopt`-style label/annotation patching before the first Helm install.

## Ingress with path-based routing

Once the controller is running, expose services through an `Ingress`
resource rather than individual `LoadBalancer` services per app (each of
those provisions its own costly public IP):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```bash
kubectl apply -f app-ingress.yaml
kubectl get ingress app-ingress
```

For TLS, pair the ingress with **cert-manager** issuing Let's Encrypt
certificates automatically rather than uploading certs by hand — out of
scope here but worth knowing the standard pattern is `cert-manager` +
`ClusterIssuer` + a `tls:` block on the Ingress.

**Gotcha:** the single ingress controller's `LoadBalancer` service is a
throughput and availability bottleneck for the whole cluster if you only
run one replica — always set `controller.replicaCount` to at least 2 and
spread them across nodes with a `podAntiAffinity` rule, since losing the
one ingress pod takes down every app behind it.

## Horizontal Pod Autoscaler (HPA)

```bash
kubectl autoscale deployment api-deployment \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

```text
$ kubectl get hpa
NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS
api-deployment    Deployment/api-deployment    45%/70%   2         10        2
```

HPA needs the **metrics-server** running (enabled by default on AKS) to
read pod CPU/memory; for custom metrics (queue depth, request rate) you
need KEDA or the custom metrics adapter, which is common enough that most
production AKS clusters run KEDA rather than plain HPA.

## Cluster autoscaler (nodes)

```bash
az aks update \
  --resource-group rg-aks-adv \
  --name aks-adv-cluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

The cluster autoscaler adds nodes when pods are **unschedulable** due to
resource requests, and removes nodes that are underutilized *and* whose
pods can all be safely rescheduled elsewhere — a node running a pod with no
`PodDisruptionBudget` allowance or a local `emptyDir` volume with data can
block scale-down indefinitely.

**Gotcha:** cluster autoscaler and HPA operate independently and on
different timescales — HPA reacts to metrics in seconds, but a new node
takes minutes to join and become schedulable. Under a sharp traffic spike,
expect a window where HPA has already requested more pods than the current
nodes can host and they sit `Pending` until the new node is ready; size
`--min-count` with enough headroom that this window doesn't cause visible
latency for real spikes you expect.

## How It Actually Works

The **cluster autoscaler** is a controller pod running inside your AKS
cluster (deployed and managed by AKS, but scheduled like any other
workload) that watches for pods stuck in `Pending` state because no node
has enough allocatable capacity — when it finds one, it calls the Azure
VMSS API directly to increase the node pool's instance count, then waits
for the new VM to boot, join the cluster via kubelet registration, and
report Ready before the scheduler places the pending pod there; scale-down
works by the same controller identifying underutilized nodes whose pods
could be safely rescheduled elsewhere, cordoning and draining them, and
then shrinking the VMSS — this is a fundamentally different, slower loop
than the Horizontal Pod Autoscaler, which only adds/removes pod replicas
within existing node capacity by polling the metrics-server API on a much
shorter interval.

**Node pools with taints and tolerations** work through the scheduler's own
predicate logic: a taint is metadata written onto the node object in etcd,
and the scheduler's filtering phase excludes any pod without a matching
toleration from being scheduled there at all — this is enforced purely at
placement time, not by any network or runtime isolation, which is why a
tainted GPU node pool genuinely prevents non-GPU workloads from landing on
expensive nodes without needing separate clusters. **Azure CNI Overlay**
(a newer mode than the plain Azure CNI from Module 4) assigns pod IPs from
a private overlay address space that's *not* part of the VNet, encapsulating
pod-to-pod traffic across nodes with VXLAN, which solves Azure CNI's IP-
exhaustion problem by trading direct VNet routability of pod IPs for
address-space efficiency — pods still reach VNet resources via NAT at the
node boundary, a materially different packet path than either kubenet or
plain Azure CNI.

## Cheat sheet

| Command | Purpose |
|---|---|
| `helm repo add` / `helm install` | Add a chart repo and install a release. |
| `helm upgrade` / `helm rollback` | Change or revert a release's config. |
| `kubectl apply -f ingress.yaml` | Create path/host-based routing rules. |
| `kubectl autoscale deployment --cpu-percent` | Configure the HPA for a deployment. |
| `kubectl get hpa` | Check current vs. target utilization and replica count. |
| `az aks update --enable-cluster-autoscaler` | Turn on node autoscaling with min/max bounds. |

## Exercise

1. Install the `ingress-nginx` chart via Helm with 2 replicas, then
   `helm upgrade` to 3 and confirm with `helm list`.
2. Deploy two backend services and a single `Ingress` resource that routes
   `/api` to one and everything else to the other.
3. Apply an HPA on one deployment with `--min=2 --max=10 --cpu-percent=70`,
   generate load, and watch `kubectl get hpa` react.
4. Enable the cluster autoscaler with `--min-count 1 --max-count 5` and
   explain why a `PodDisruptionBudget` can block scale-down.
5. Delete the resource group when finished.
