# 03 · Advanced AKS (Service Mesh, Multi-Cluster)

[Level 3, Module 03](../level-3/03-advanced-aks.md) covered Helm, ingress,
and autoscaling on a single cluster. At enterprise scale you typically need
service-to-service **mTLS and traffic policy** without changing application
code (a service mesh), and **more than one cluster** for blast-radius
isolation, regional failover, or team boundaries.

## Enabling Istio via the AKS mesh add-on

AKS has a managed Istio-based service mesh add-on — no separate Istio
control plane to operate yourself:

```bash
az aks mesh enable \
  --resource-group rg-aks-mesh \
  --name aks-mesh-cluster

az aks show \
  --resource-group rg-aks-mesh \
  --name aks-mesh-cluster \
  --query "serviceMeshProfile.mode" -o tsv
```

```text
Istio
```

Enabling the mesh add-on does **not** automatically inject sidecars into
existing workloads — you opt namespaces in explicitly:

```bash
kubectl label namespace orders istio.io/rev=asm-1-20
kubectl rollout restart deployment -n orders
```

**Gotcha:** the sidecar is only injected on **pod creation**, so labeling
the namespace has no effect on already-running pods — you must trigger a
rollout (as above) or the pods keep running unmeshed with no error or
warning that they're outside the mesh.

## mTLS enforcement

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: orders
spec:
  mtls:
    mode: STRICT
```

`STRICT` mode rejects any plaintext traffic to meshed pods; `PERMISSIVE`
(the default) accepts both, which is the safer rollout path — start
`PERMISSIVE`, confirm all traffic is actually flowing over mTLS via
observability, then flip to `STRICT`.

**Gotcha:** going straight to `STRICT` before every caller of a service is
meshed breaks any non-meshed caller (a health-check probe from outside the
mesh, a cron job in an un-labeled namespace) with connection resets that
give no indication mTLS is the cause — this is the mesh equivalent of the
WAF Prevention-mode mistake from earlier levels: always stage through
permissive/detection first.

## Traffic splitting for canary releases

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: orders-api
  namespace: orders
spec:
  hosts:
  - orders-api
  http:
  - route:
    - destination:
        host: orders-api
        subset: v1
      weight: 90
    - destination:
        host: orders-api
        subset: v2
      weight: 10
```

This is the same canary concept as
[Level 3's Flagger example](../level-3/08-cicd-containers-gitops.md), but
expressed at the mesh layer via native traffic-splitting rather than a
separate progressive-delivery controller — many teams run Flagger *on top
of* Istio, using Flagger to automate the weight changes based on metrics
rather than hand-editing `VirtualService` weights.

## Multi-cluster patterns

| Pattern | Purpose | Complexity |
|---|---|---|
| **Multiple clusters, no mesh connection** | Team/environment isolation, independent blast radius | Low |
| **Fleet management (Azure Kubernetes Fleet Manager)** | Coordinate updates and config propagation across clusters | Moderate |
| **Multi-primary mesh (cross-cluster Istio)** | Services in different clusters call each other over mTLS as if local | High |

```bash
az fleet create \
  --resource-group rg-aks-fleet \
  --name fleet-orders \
  --location eastus

az fleet member create \
  --resource-group rg-aks-fleet \
  --fleet-name fleet-orders \
  --name member-eastus \
  --member-cluster-id $(az aks show -g rg-aks-mesh -n aks-mesh-cluster --query id -o tsv)
```

Fleet Manager lets you define an **update run** that rolls a Kubernetes
version upgrade across member clusters in a controlled order (stage 1
clusters, wait, stage 2 clusters) instead of upgrading each cluster
manually and independently, which is how version drift between clusters
accumulates unnoticed.

**Gotcha:** Fleet Manager coordinates control-plane/node-pool upgrades and
config propagation — it does **not** automatically give you cross-cluster
service discovery or mTLS; that still requires a multi-primary mesh setup
layered on top, so "we use Fleet Manager" and "we have a multi-cluster
mesh" are two different, independent capabilities people sometimes conflate.

## Observability with the mesh

```bash
kubectl get pods -n aks-istio-system

istioctl proxy-status
```

```text
NAME                                CDS        LDS        EDS        RDS          ISTIOD
orders-api-7d9f-xk2p1.orders        SYNCED     SYNCED     SYNCED     SYNCED       istiod-asm-1-20-...
```

A pod's proxy showing `STALE` instead of `SYNCED` means that pod's Envoy
sidecar hasn't received the latest config from `istiod` — traffic rules you
just applied (like the canary split above) won't take effect for that pod
until it resyncs, which is a frequent explanation for "I updated the
`VirtualService` but nothing changed for this one pod."

## Cheat sheet

| Command | Purpose |
|---|---|
| `az aks mesh enable` | Turn on the managed Istio add-on. |
| `kubectl label namespace istio.io/rev=<rev>` + rollout restart | Opt a namespace into sidecar injection. |
| `PeerAuthentication` (mtls: STRICT/PERMISSIVE) | Control mTLS enforcement. |
| `VirtualService` weighted routes | Split traffic between service subsets. |
| `az fleet create` / `az fleet member create` | Group clusters for coordinated upgrades. |
| `istioctl proxy-status` | Check sidecar config sync state. |

## Exercise

1. Enable the AKS mesh add-on, label one namespace for injection, and
   restart its deployments; confirm sidecars are present with
   `kubectl get pods -o jsonpath` checking container count.
2. Apply `PeerAuthentication` in `PERMISSIVE` mode, confirm traffic still
   flows from an unmeshed caller, then flip to `STRICT` and observe the
   unmeshed caller break.
3. Create a `VirtualService` splitting traffic 90/10 between two subsets of
   one service.
4. Create a Fleet Manager instance with two member clusters and describe
   (without executing) what an update run would coordinate.
5. Delete the resource groups when finished.
