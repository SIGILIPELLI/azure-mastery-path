---
title: "Azure Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Azure cheat sheet: 38 quick-reference tables from every level of the Azure Mastery Path, each linked to its full lesson."
---

# Azure Cheat Sheet

Quick-reference tables from **38 lessons** of the free Azure Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; Azure CLI/Portal](level-1/01-setup-cli-portal.md)

| Command | Purpose |
|---------|---------|
| `az login` | Authenticate the CLI to your Azure account. |
| `az account list -o table` | List subscriptions you can access. |
| `az account set --subscription <name-or-id>` | Switch the active subscription. |
| `az group create -n <name> -l <region>` | Create a resource group. |
| `az group list -o table` | List resource groups. |
| `az group delete -n <name> --yes --no-wait` | Delete a resource group and everything in it. |
| `az account list-locations -o table` | List available Azure regions. |
| `az version` | Show the installed CLI version. |

### [02 · Azure AD &amp; RBAC / Security Basics](level-1/02-azure-ad-rbac-security.md)

| Command / concept | Purpose |
|---|---|
| `az account show --query tenantId` | Find your Entra ID tenant ID. |
| `az ad signed-in-user show` | Show your own signed-in identity. |
| `az role assignment create --assignee --role --scope` | Grant a role to an identity at a scope. |
| `az role assignment list --scope <scope>` | List who has access at a scope. |
| `az role assignment delete --assignee --role --scope` | Revoke a role assignment. |
| Owner / Contributor / Reader | The three built-in roles you'll use most. |
| Scope hierarchy | Management Group → Subscription → Resource Group → Resource. |
| Least privilege | Narrowest role, narrowest scope, reviewed periodically. |

### [03 · Virtual Machines](level-1/03-virtual-machines.md)

| Command | Purpose |
|---|---|
| `az vm list-sizes --location <region>` | List available VM sizes in a region. |
| `az vm create -g -n --image --size --admin-username --generate-ssh-keys` | Create a Linux VM with SSH key auth. |
| `az vm show --show-details --query publicIps -o tsv` | Get a VM's public IP. |
| `az network nsg rule list -g --nsg-name` | List NSG rules. |
| `az network nsg rule create ... --priority --destination-port-ranges --access` | Add an NSG rule. |
| `az vm deallocate -g -n` | Stop the VM and release compute billing. |
| `az vm start` / `az vm restart` | Start / restart a VM. |
| `az vm resize -g -n --size` | Change a VM's size. |

### [04 · Blob Storage](level-1/04-blob-storage.md)

| Command | Purpose |
|---|---|
| `az storage account create --sku --kind` | Create a storage account. |
| `az storage container create --public-access` | Create a container (private by default). |
| `az storage blob upload/download/list` | Upload, download, or list blobs. |
| `az storage blob generate-sas --permissions --expiry` | Create a time-limited shareable link. |
| `az storage blob set-tier --tier` | Change a blob's access tier. |
| `az storage blob service-properties update --static-website` | Turn on static website hosting. |
| Hot / Cool / Cold / Archive | Access tiers, cheapest storage to cheapest access in that order. |

### [05 · Virtual Networks Basics](level-1/05-virtual-networks.md)

| Command | Purpose |
|---|---|
| `az network vnet create --address-prefix --subnet-name --subnet-prefix` | Create a VNet with an initial subnet. |
| `az network vnet subnet create` | Add another subnet to an existing VNet. |
| `az network vnet subnet list -o table` | List a VNet's subnets. |
| `az network nsg create` / `nsg rule create` | Create an NSG and its rules. |
| `az network vnet subnet update --network-security-group` | Attach an NSG to a subnet. |
| `az network route-table create` / `route create` | Create a route table and a custom route. |
| `az network vnet subnet update --route-table` | Attach a route table to a subnet. |

### [06 · Azure SQL Database / Cosmos DB Basics](level-1/06-azure-sql-cosmos-db.md)

| Command | Purpose |
|---|---|
| `az sql server create --admin-user --admin-password` | Create a logical SQL server. |
| `az sql db create --edition --use-free-limit` | Create an Azure SQL database. |
| `az sql server firewall-rule create --start-ip --end-ip` | Allow an IP range to connect. |
| `az sql db show-connection-string --client` | Print a connection string template. |
| `az cosmosdb create --enable-free-tier true` | Create a Cosmos DB account on the free tier. |
| `az cosmosdb sql database create` / `sql container create` | Create a Cosmos database and container. |
| `az cosmosdb keys list --type connection-strings` | Get connection strings/keys. |

### [07 · Azure Functions](level-1/07-azure-functions.md)

| Command | Purpose |
|---|---|
| `func init <name> --python` | Scaffold a new local Functions project. |
| `func new --name --template` | Add a function with a given trigger type. |
| `func start` | Run functions locally at `localhost:7071`. |
| `az functionapp create --consumption-plan-location --runtime` | Create a Function App on the Consumption plan. |
| `func azure functionapp publish <app>` | Deploy local code to Azure. |
| `az functionapp function keys list` | Get the key(s) needed to call a `function`-level HTTP trigger. |
| Trigger vs. binding | Trigger starts the run (one per function); bindings read/write data declaratively. |

### [08 · Azure Monitor &amp; Log Analytics](level-1/08-azure-monitor-log-analytics.md)

| Command | Purpose |
|---|---|
| `az monitor metrics list --metric --interval` | Read a resource's platform metrics. |
| `az monitor log-analytics workspace create` | Create a Log Analytics workspace. |
| `az monitor diagnostic-settings create --workspace --logs --metrics` | Route a resource's logs/metrics to a workspace. |
| `az monitor log-analytics query --analytics-query` | Run a KQL query from the CLI. |
| `az monitor action-group create --action email` | Create a notification target for alerts. |
| `az monitor metrics alert create --condition --action` | Create a metric-based alert rule. |
| `where` / `summarize by` / `ago()` | Core KQL filtering, aggregation, and time-window keywords. |

### [09 · ARM Templates / Bicep Intro](level-1/09-arm-bicep-intro.md)

| Command / syntax | Purpose |
|---|---|
| `az bicep install` / `upgrade` | Install or update the Bicep compiler. |
| `param <name> <type> = <default>` | Declare an input parameter. |
| `resource <symbolicName> '<type>@<api>' = {...}` | Declare a resource. |
| `output <name> <type> = <expr>` | Expose a value after deployment. |
| `az deployment group create --template-file` | Deploy a Bicep/ARM file to a resource group. |
| `az deployment group what-if --template-file` | Preview changes before applying. |
| `az bicep build` / `decompile` | Convert Bicep → ARM JSON, or JSON → Bicep. |

### [10 · Capstone Project — End-to-End Web App](level-1/10-capstone-project.md)

| Command | Purpose |
|---|---|
| `az cosmosdb create --enable-free-tier true` | Free-tier Cosmos DB account. |
| `az functionapp create --consumption-plan-location --runtime` | Serverless Function App backend. |
| `az functionapp config appsettings set --settings` | Store connection info as app config, not hardcoded. |
| `az staticwebapp create --source --login-with-github` | Create a Static Web App wired to a GitHub repo + Actions deploy. |
| `az staticwebapp backends link --backend-resource-id` | Attach a Function App as the SWA's `/api/*` backend. |
| `az group delete --name --yes --no-wait` | Tear down every resource in the project at once. |
| `az group list -o table` | Sanity-check nothing else is still running. |

## Level 2 · Intermediate

### [01 · App Service &amp; Web Apps Deep Dive](level-2/01-app-service-deep-dive.md)

| Command | Purpose |
|---|---|
| `az appservice plan create --sku --is-linux` | Create the compute plan behind one or more web apps. |
| `az webapp create --plan --runtime` | Create a web app on a plan. |
| `az webapp deployment slot create --slot` | Create a staging (or other named) slot. |
| `az webapp deployment slot swap --slot --target-slot` | Swap a slot into production (or roll back). |
| `az webapp config appsettings set --slot-settings` | Mark a setting as slot-specific (doesn't swap). |
| `az appservice plan update --number-of-workers` | Manual scale-out to a fixed instance count. |
| `az monitor autoscale create` / `rule create` | Rule-based autoscale on a metric. |
| `az webapp config hostname add` | Map a custom domain. |
| `az webapp config ssl create/bind` | Issue and bind a free managed certificate. |
| `az webapp deployment github-actions add` | Wire up GitHub Actions CI/CD. |

### [02 · Storage Deep Dive (Files, Queues, Tables)](level-2/02-storage-deep-dive.md)

| Command | Purpose |
|---|---|
| `az storage share create --quota` | Create an Azure Files share with a provisioned size. |
| `mount -t cifs //<account>.file.core.windows.net/<share>` | Mount a share over SMB. |
| `az storage queue create` | Create a Queue Storage queue. |
| `az storage message put/get/delete` | Enqueue, dequeue (peek+lock), and delete a message. |
| `az storage table create` | Create a Table Storage table. |
| `az storage entity insert/query` | Insert or query entities by partition/row key. |
| `az storage account management-policy create --policy` | Apply a lifecycle rule (tier/delete by age). |

### [03 · Networking Deep Dive (Load Balancer, App Gateway)](level-2/03-networking-deep-dive.md)

| Command | Purpose |
|---|---|
| `az network lb create --sku Standard` | Create a Layer 4 load balancer. |
| `az network lb probe create` / `rule create` | Define health checks and forwarding rules. |
| `az network application-gateway create --sku WAF_v2` | Create a Layer 7 gateway with optional WAF. |
| `az network application-gateway url-path-map create` | Route by URL path to different backend pools. |
| `az network private-endpoint create --group-id` | Attach a private IP to a PaaS resource. |
| `az network private-dns zone create` | Create the `privatelink.*` DNS zone. |
| `az network private-dns link vnet create` | Link the zone to a VNet so names resolve privately. |
| `az network private-endpoint dns-zone-group create` | Auto-populate DNS records for the endpoint. |

### [04 · Azure Kubernetes Service (AKS) Basics](level-2/04-aks-basics.md)

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

### [05 · Azure DevOps &amp; CI/CD Pipelines](level-2/05-azure-devops-cicd.md)

| Command / concept | Purpose |
|---|---|
| `az devops project create` / `az repos create` | Create a project and a Git repo. |
| `azure-pipelines.yml` (`trigger`, `pool`, `stages`) | Define the CI/CD pipeline as code. |
| `az pipelines create --yml-path` | Register a pipeline from a YAML file. |
| Service connection | Stored credential a pipeline uses to reach Azure. |
| Variable group / Key Vault link | Store secrets outside the YAML file. |
| Environment approvals | Manual gate before a stage (e.g. production deploy) runs. |

### [06 · Managed Identities &amp; Key Vault](level-2/06-managed-identities-key-vault.md)

| Command | Purpose |
|---|---|
| `az webapp identity assign` | Enable system-assigned identity, or attach a user-assigned one. |
| `az identity create` | Create a standalone user-assigned identity. |
| `az keyvault create --enable-rbac-authorization` | Create a vault using RBAC (not legacy access policies). |
| `az keyvault secret set --name --value` | Store a secret. |
| `az role assignment create --role "Key Vault Secrets User"` | Grant read-only secret access to an identity. |
| `@Microsoft.KeyVault(SecretUri=...)` app setting | Auto-resolved Key Vault reference, no SDK code needed. |
| `DefaultAzureCredential` (Python/`.NET`/etc. SDK) | Uses the compute resource's managed identity automatically. |

### [07 · Container Instances &amp; Container Registry](level-2/07-aci-acr.md)

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

### [08 · Cosmos DB Deep Dive](level-2/08-cosmos-db-deep-dive.md)

| Command / concept | Purpose |
|---|---|
| `--partition-key-path` | Choose the property Cosmos DB partitions on (immutable after creation). |
| `az cosmosdb sql container throughput update --throughput` | Set manual/provisioned RU/s. |
| `--max-throughput` | Switch to autoscale between 10-100% of this max. |
| `EnableServerless` capability | Per-RU billing, no provisioning, mutually exclusive with provisioned. |
| `--default-consistency-level` | Account-wide default consistency (Strong/BoundedStaleness/Session/ConsistentPrefix/Eventual). |
| `query_items_change_feed()` | Read the ordered log of inserts/updates on a container. |
| `429` response | RU budget exceeded for that partition/account; back off and retry. |

### [09 · Azure Functions Advanced](level-2/09-azure-functions-advanced.md)

| Concept | Purpose |
|---|---|
| Orchestrator function | Describes multi-step workflow logic; must be deterministic (no I/O, no `datetime.now()`, no random). |
| Activity function | Does the actual work (I/O, external calls); called via `context.call_activity`. |
| Client function | Starts a new orchestration instance, returns a status-check URL. |
| `context.task_all([...])` | Fan-out/fan-in: run tasks concurrently, wait for all. |
| `context.call_sub_orchestrator` | Nest one orchestration inside another. |
| `host.json` extensionBundle | Registers non-core trigger/binding types (Event Grid, Durable, Service Bus). |
| `az functionapp deployment slot create/swap` | Slots for Function Apps — Premium/Dedicated plans only. |

### [10 · Project — Multi-Tier Web Application](level-2/10-project-multi-tier-app.md)

| Command | Purpose |
|---|---|
| `az network private-endpoint create --group-id Sql` | Private endpoint for Cosmos DB. |
| `az aks create --enable-oidc-issuer --enable-workload-identity` | AKS cluster ready for federated pod identities. |
| `az identity federated-credential create --subject system:serviceaccount:...` | Bind a K8s service account to an Azure AD identity. |
| `azure.workload.identity/use: "true"` pod label | Opts a pod into workload identity token injection. |
| `az network application-gateway create --sku WAF_v2` | Layer 7 entry point with WAF, in front of AKS. |
| `az aks enable-addons --addons ingress-appgw` | Install AGIC to sync Ingress objects → App Gateway backend pool. |
| `az group delete --yes --no-wait` | Tear down the whole project. |

## Level 3 · Advanced

### [01 · Advanced Networking (VPN Gateway, ExpressRoute)](level-3/01-advanced-networking.md)

| Command | Purpose |
|---|---|
| `az network vnet subnet create --name GatewaySubnet` | Reserve the dedicated subnet gateways require. |
| `az network vnet-gateway create --gateway-type Vpn` | Create a VPN gateway (30-45 min provisioning). |
| `az network local-gateway create` | Define the on-premises endpoint. |
| `az network vpn-connection create` | Establish the IPsec tunnel between gateways. |
| `az network express-route create` | Provision an ExpressRoute circuit. |
| `az network express-route show --query serviceProviderProvisioningState` | Check if the provider has activated the circuit. |
| `az network vnet peering create --allow-gateway-transit` | Let a spoke use the hub's gateway (set on hub side). |
| `az network vnet peering create --use-remote-gateways` | Consume the hub's gateway from a spoke (set on spoke side). |

### [02 · Infrastructure as Code at Scale (Bicep/Terraform)](level-3/02-iac-at-scale.md)

| Command | Purpose |
|---|---|
| `az deployment group create --template-file main.bicep` | Deploy a Bicep template. |
| `az deployment group what-if` | Preview changes before applying. |
| `az deployment operation group list` | Inspect a failed nested deployment. |
| `terraform init` | Initialize providers and backend. |
| `terraform plan -out=plan.tfplan` | Preview and save a plan. |
| `terraform apply plan.tfplan` | Apply a previously reviewed plan. |
| `terraform force-unlock <ID>` | Clear a stale state lock (confirm first). |

### [03 · Advanced AKS (Helm, Ingress, Autoscaling)](level-3/03-advanced-aks.md)

| Command | Purpose |
|---|---|
| `helm repo add` / `helm install` | Add a chart repo and install a release. |
| `helm upgrade` / `helm rollback` | Change or revert a release's config. |
| `kubectl apply -f ingress.yaml` | Create path/host-based routing rules. |
| `kubectl autoscale deployment --cpu-percent` | Configure the HPA for a deployment. |
| `kubectl get hpa` | Check current vs. target utilization and replica count. |
| `az aks update --enable-cluster-autoscaler` | Turn on node autoscaling with min/max bounds. |

### [04 · Event-Driven Architecture (Event Grid, Service Bus)](level-3/04-event-driven-architecture.md)

| Command | Purpose |
|---|---|
| `az eventgrid topic create` | Create a custom Event Grid topic. |
| `az eventgrid event-subscription create --endpoint-type webhook` | Subscribe an endpoint to a topic. |
| `az servicebus namespace create` | Create a Service Bus namespace. |
| `az servicebus queue create --max-delivery-count` | Create a queue with a dead-letter threshold. |
| `az servicebus topic subscription rule create --filter-sql-expression` | Filter which messages a subscription receives. |
| `az servicebus queue show --query messageCount` | Check current queue depth. |

### [05 · API Management](level-3/05-api-management.md)

| Command | Purpose |
|---|---|
| `az apim create --sku-name Developer --no-wait` | Provision an APIM instance (30-45 min). |
| `az apim api import --specification-format OpenApi` | Import a backend API from its OpenAPI spec. |
| `az apim product create --subscription-required true` | Create a product bundling APIs with a subscription key. |
| `az apim product api add` | Attach an API to a product. |
| `az apim api revision create` | Create a testable, non-breaking revision. |
| `<rate-limit-by-key>` policy | Throttle calls per subscription/IP/custom key. |
| `<validate-jwt>` policy | Require and validate a bearer token before the backend. |

### [06 · Defender for Cloud &amp; Governance (Policy, Blueprints)](level-3/06-defender-governance.md)

| Command | Purpose |
|---|---|
| `az security pricing create --tier Standard` | Enable a Defender plan for a resource type. |
| `az security secure-scores list` | Read the current Secure Score. |
| `az security assessment list` | List unhealthy recommendations. |
| `az policy definition create --rules` | Define a custom policy rule. |
| `az policy assignment create --scope` | Apply a policy or initiative at a scope. |
| `az policy remediation create` | Retroactively fix non-compliant existing resources. |
| `az policy set-definition create` | Group policies into an initiative. |
| `az ts create` | Create a versioned Template Spec (Blueprints replacement). |

### [07 · High Availability &amp; Disaster Recovery](level-3/07-ha-disaster-recovery.md)

| Command | Purpose |
|---|---|
| `az vm create --zone` | Pin a VM to a specific Availability Zone. |
| `az vm list-skus --zone` | Check zone support for a SKU in a region. |
| `az sql db create --zone-redundant true` | Create a zone-redundant SQL database. |
| `az backup protection enable-for-vm` | Enable backup/replication for a VM. |
| `az sql failover-group create` | Set up SQL cross-region failover. |
| `az sql failover-group set-primary` | Trigger a manual failover. |
| `az storage account failover` | Manually fail over RA-GRS storage (irreversible). |

### [08 · CI/CD for Containers &amp; GitOps](level-3/08-cicd-containers-gitops.md)

| Command | Purpose |
|---|---|
| `az acr build --registry --image` | Build and push an image via ACR's managed build. |
| `az k8s-configuration flux create --kustomization prune=true` | Wire a cluster to reconcile from a Git repo. |
| `az k8s-configuration flux show --query complianceState` | Check whether the cluster matches Git. |
| `git rev-parse --short HEAD` | Get a stable, unique image tag for a commit. |
| `Canary` (Flagger CRD) | Define automated, metric-gated progressive rollout. |

### [09 · Cost Management &amp; Optimization](level-3/09-cost-management-optimization.md)

| Command | Purpose |
|---|---|
| `az costmanagement query` | Query aggregated cost data programmatically. |
| `az consumption budget create --notifications` | Create a budget with threshold alerts. |
| `az advisor recommendation list --category Cost` | List right-sizing and cost recommendations. |
| `az reservations reservation-order list` | Check existing reservation utilization. |
| `az vm create --priority Spot --max-price -1` | Create a Spot VM (capacity-based eviction only). |

## Level 4 · Master

### [01 · Enterprise Landing Zones (CAF)](level-4/01-enterprise-landing-zones.md)

| Command | Purpose |
|---|---|
| `az account management-group create --parent` | Build the management group hierarchy. |
| `az account management-group subscription add` | Move a subscription into a landing zone. |
| `az network vnet peering create` (cross-subscription) | Peer a spoke in one subscription to a hub in another. |
| `az account management-group list` | Inspect the current hierarchy. |
| `az policy assignment list --scope <mg-id>` | See what a management group scope enforces. |

### [02 · Management Groups &amp; Governance at Scale](level-4/02-management-groups-governance.md)

| Command | Purpose |
|---|---|
| `az policy assignment list-resources` | See what a specific assignment applies to. |
| `az policy exemption create --expires-on` | Grant a documented, time-boxed compliance waiver. |
| `az role assignment create --scope <mg-id>` | Grant RBAC inherited by everything under a management group. |
| `az role definition create` | Define a custom role narrower than built-ins. |
| `az policy state summarize --management-group` | Get an aggregate compliance view. |
| `az policy state trigger-scan` | Force an on-demand compliance re-evaluation. |

### [03 · Advanced AKS (Service Mesh, Multi-Cluster)](level-4/03-advanced-aks-service-mesh.md)

| Command | Purpose |
|---|---|
| `az aks mesh enable` | Turn on the managed Istio add-on. |
| `kubectl label namespace istio.io/rev=<rev>` + rollout restart | Opt a namespace into sidecar injection. |
| `PeerAuthentication` (mtls: STRICT/PERMISSIVE) | Control mTLS enforcement. |
| `VirtualService` weighted routes | Split traffic between service subsets. |
| `az fleet create` / `az fleet member create` | Group clusters for coordinated upgrades. |
| `istioctl proxy-status` | Check sidecar config sync state. |

### [04 · Azure Arc &amp; Hybrid Cloud](level-4/04-azure-arc-hybrid-cloud.md)

| Command | Purpose |
|---|---|
| `az connectedmachine onboarding-script generate` | Generate the per-server Arc agent install script. |
| `az connectedmachine show --query status` | Check agent heartbeat/connection status. |
| `az connectedk8s connect` | Project an external Kubernetes cluster into Azure. |
| `az k8s-configuration flux create --cluster-type connectedClusters` | Apply GitOps to an Arc-enabled cluster. |
| `az policy assignment create --mi-system-assigned` | Create a policy assignment with a remediation identity. |

### [05 · Advanced Observability (App Insights, Tracing)](level-4/05-advanced-observability.md)

| Command | Purpose |
|---|---|
| `az monitor app-insights component create --workspace` | Create workspace-based App Insights. |
| `az monitor app-insights component show --query connectionString` | Get the connection string to wire into an app. |
| `requests \| where operation_Id ==` | Pull one full distributed trace by ID. |
| `dependencies \| summarize percentile(duration, 95)` | Get p95 latency for downstream calls. |
| `az monitor metrics alert create` | Alert on a built-in platform metric. |
| `az monitor scheduled-query create` | Alert on a custom KQL condition. |

### [06 · Performance Engineering &amp; Load Testing](level-4/06-performance-load-testing.md)

| Command | Purpose |
|---|---|
| `az load create` | Create an Azure Load Testing resource. |
| `az load test create --load-test-config-file` | Define a test from a JMeter plan and config. |
| `az load test-run create` | Execute a test run. |
| `az load test-run show --query status` | Check run status/results. |
| `az monitor app-insights component update --set properties.SamplingPercentage` | Adjust Profiler sampling rate. |

### [07 · Security Deep Dive (Sentinel, Zero Trust)](level-4/07-security-sentinel-zero-trust.md)

| Command | Purpose |
|---|---|
| `az sentinel workspace create` | Onboard Sentinel onto a Log Analytics workspace. |
| `az security data-connector create` | Ingest a data source (Azure AD, Defender, etc.). |
| `az sentinel alert-rule create --kind Scheduled` | Create a KQL-based correlation/detection rule. |
| `az sentinel automation-rule create` | Auto-trigger a playbook when an incident is created. |
| `az logic workflow create` | Create the Logic App playbook itself. |
| PIM `roleAssignmentScheduleRequests` (Graph API) | Request just-in-time privileged role activation. |

### [08 · Data Platform at Scale (Synapse, Data Factory)](level-4/08-data-platform-at-scale.md)

| Command | Purpose |
|---|---|
| `az datafactory pipeline create` | Define an orchestration pipeline. |
| `az datafactory trigger create` + `trigger start` | Schedule and actually enable a pipeline run. |
| `az synapse sql pool create --performance-level` | Provision a dedicated SQL pool. |
| `az synapse sql pool pause` / `resume` | Control dedicated pool cost by stopping/starting compute. |
| `OPENROWSET(BULK ... FORMAT='PARQUET')` | Query lake files directly via serverless SQL. |
| `az datafactory pipeline-run query-by-factory` | Check pipeline run history and status. |

### [09 · FinOps &amp; Cost Governance at Scale](level-4/09-finops-cost-governance.md)

| Command | Purpose |
|---|---|
| `az policy definition create` (deny/modify on tags) | Enforce or auto-apply cost allocation tags. |
| `az costmanagement query --dataset-grouping type=TagKey` | Aggregate cost by a tag (e.g. CostCenter). |
| `az consumption reservation-summary list` | Check reservation utilization across the org. |
| `az consumption budget create --scope <mg-id>` | Set a budget at management group scope. |
| `--notifications` with `thresholdType: Forecasted` | Alert before overrun happens, not after. |

---

Go deeper with the full [Azure Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
