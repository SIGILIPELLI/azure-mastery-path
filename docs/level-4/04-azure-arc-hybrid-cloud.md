# 04 · Azure Arc & Hybrid Cloud

Everything so far assumed workloads run in Azure. Real enterprises usually
have on-premises servers, other clouds, or edge locations that need the
*same* governance, monitoring, and management experience as native Azure
resources — without migrating them. **Azure Arc** projects non-Azure
infrastructure into the Azure Resource Manager control plane so it shows up
alongside real Azure resources for policy, RBAC, and monitoring purposes.

## Arc-enabling a server

```bash
az connectedmachine list --resource-group rg-arc --output table
```

On the actual server (Linux/Windows, on-prem or another cloud), you install
the Azure Connected Machine agent, generated per-server by:

```bash
az connectedmachine onboarding-script generate \
  --resource-group rg-arc \
  --location eastus \
  --output-directory ./onboarding
```

This produces a script the server runs locally to register itself; once
connected, it appears as a `Microsoft.HybridCompute/machines` resource:

```text
$ az connectedmachine show -g rg-arc -n onprem-web-01 --query "{status:status, osName:osName}"
{
  "status": "Connected",
  "osName": "Ubuntu 22.04"
}
```

**Gotcha:** an Arc-connected machine's `status` reflects **agent
heartbeat**, not workload health — a server can show `Connected` while the
application running on it is completely down; Arc gives you the management
plane, not application-level monitoring, which still needs Azure Monitor
agent extensions installed on top.

## Applying Azure Policy to Arc-enabled servers

The entire point of Arc is that governance from
[Level 4, Module 02](02-management-groups-governance.md) now reaches
non-Azure machines:

```bash
az policy assignment create \
  --name require-monitor-agent-arc \
  --policy "deploy-azure-monitor-agent-arc" \
  --scope "/subscriptions/xxxx/resourceGroups/rg-arc" \
  --location eastus \
  --mi-system-assigned
```

A `DeployIfNotExists` policy like this one uses a **system-assigned managed
identity on the policy assignment itself** to deploy the Monitor Agent
extension onto any Arc machine lacking it — the `--mi-system-assigned` and
`--location` flags are required for policies with a remediation identity,
which trips people up since simpler `Deny`-effect policies don't need them.

**Gotcha:** this remediation identity needs its own RBAC role assignment
(typically `Contributor` or a narrower custom role) on the target scope
before it can actually deploy anything — creating the assignment without
granting the identity permissions leaves the policy showing
`NonCompliant` with remediation tasks failing silently in the background.

## Arc-enabled Kubernetes

The same projection works for Kubernetes clusters running outside Azure
(on-prem, another cloud, edge):

```bash
az connectedk8s connect \
  --resource-group rg-arc \
  --name onprem-k8s-cluster \
  --location eastus
```

```text
$ az connectedk8s show -g rg-arc -n onprem-k8s-cluster --query connectivityStatus -o tsv
Connected
```

Once connected, you can layer on the **same GitOps extension** used for
native AKS in [Level 3, Module 08](../level-3/08-cicd-containers-gitops.md):

```bash
az k8s-configuration flux create \
  --resource-group rg-arc \
  --cluster-name onprem-k8s-cluster \
  --cluster-type connectedClusters \
  --name flux-config \
  --namespace flux-system \
  --scope cluster \
  --url https://github.com/example-org/onprem-manifests \
  --branch main \
  --kustomization name=apps path=./apps
```

Note `--cluster-type connectedClusters` instead of `managedClusters` — this
is the only material difference from the native-AKS command, which is the
appeal of Arc: the same tooling and pipelines work whether the cluster is
in Azure or a datacenter.

**Gotcha:** Arc-enabled Kubernetes requires **outbound** connectivity from
the cluster to Azure (agents phone home) — it does not open any inbound
path into the on-prem network. Teams sometimes assume Arc gives Azure a way
to reach into the on-prem cluster directly; it's the reverse, the cluster's
agents push status and pull configuration outward.

## Arc data services (SQL Managed Instance, PostgreSQL on Arc)

Arc also extends to **data services**, running the same SQL/PostgreSQL
engine on your own infrastructure but managed (patching, backups, some
billing) through Azure:

```bash
az sql instance-failover-group-arc create \
  --name fg-arc-sql \
  --mi arc-sql-primary \
  --resource-group rg-arc
```

**Gotcha:** Arc data services run in one of two modes —
**directly connected** (near real-time billing/management sync to Azure)
or **indirectly connected** (periodic export, works in disconnected/air-
gapped environments) — the mode is chosen at deployment and affects which
Azure-side features (like some Azure Monitor integrations) are available;
switching modes later is not a simple config change.

## Arc scope comparison

| | Arc-enabled Servers | Arc-enabled Kubernetes | Arc Data Services |
|---|---|---|---|
| **What it projects** | VM/bare-metal OS | Any CNCF-conformant cluster | SQL MI, PostgreSQL |
| **Requires** | Connected Machine agent | Arc K8s agents (Helm-installed) | Data controller on the cluster |
| **Enables** | Policy, Monitor, Update Manager | GitOps, Policy, Monitor | Managed patching/backup, unified billing |
| **Connectivity direction** | Outbound only | Outbound only | Outbound (mode-dependent detail) |

## How It Actually Works

**Azure Arc** extends ARM's control plane to resources that don't
physically run in Azure by installing a lightweight **Arc agent** (the
Connected Machine agent, for servers; the Arc-enabled Kubernetes agent's
Helm-deployed operators, for clusters) that establishes an *outbound*
HTTPS connection to Azure and registers the on-prem/multi-cloud resource as
a proxy ARM resource — critically, this means the resource gets a real
resource ID, RBAC, tags, and Policy applicability exactly like a native
Azure resource, but Azure has no inbound network path to it at all; every
control action (running a policy remediation, deploying a Kubernetes
config via GitOps) is delivered by the outbound-connected agent *polling*
Azure for pending work rather than Azure pushing to it, mirroring the same
pull-based trust model as the GitOps controller from Level 3.

**Arc-enabled data services** and **Arc-enabled Kubernetes configuration**
extend this further using the exact GitOps mechanism from Level 3, Module 8:
Arc deploys a Flux-based configuration operator onto the connected cluster,
which pulls manifests from a Git repo you specify and reconciles them
locally — Azure never needs to reach into the cluster to apply anything, it
only needs to know (via the agent's periodic status push) whether the
cluster's Flux operator reports the deployment as synced. This is the
concrete reason Arc's governance story (Azure Policy applied to on-prem
Kubernetes, for instance via Gatekeeper policy definitions pushed the same
GitOps way) can claim consistent governance across hybrid infrastructure:
it's reusing the identical pull-based reconciliation loop already proven
inside native AKS, just pointed at infrastructure Azure doesn't own.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az connectedmachine onboarding-script generate` | Generate the per-server Arc agent install script. |
| `az connectedmachine show --query status` | Check agent heartbeat/connection status. |
| `az connectedk8s connect` | Project an external Kubernetes cluster into Azure. |
| `az k8s-configuration flux create --cluster-type connectedClusters` | Apply GitOps to an Arc-enabled cluster. |
| `az policy assignment create --mi-system-assigned` | Create a policy assignment with a remediation identity. |

## Exercise

1. Generate an onboarding script with
   `az connectedmachine onboarding-script generate` and read through what
   it does (agent download, registration) without necessarily running it
   against a real server.
2. Explain why an Arc machine's `Connected` status does not imply the
   application on it is healthy, and name one Azure service that would
   close that gap.
3. Connect a (test or kind/minikube) Kubernetes cluster with
   `az connectedk8s connect` and confirm `connectivityStatus`.
4. Apply the GitOps extension to the Arc-connected cluster using
   `--cluster-type connectedClusters` and compare the command to the
   native-AKS version from Level 3.
5. Clean up the resource group and disconnect the test cluster when
   finished.
