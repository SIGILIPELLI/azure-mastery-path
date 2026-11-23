---
title: "Azure Interview Questions and Answers (179 Questions)"
description: "179 Azure interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Azure Mastery Path."
---

# Azure Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Azure Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; Azure CLI/Portal](level-1/01-setup-cli-portal.md)

**Q1.** What does a new Azure free account include?

**Answer:** A 30-day/$200 credit plus 'Always Free' services within usage caps

A card is needed for identity verification; Azure won't auto-upgrade to a paid plan.

**Q2.** What is the #1 way a free trial turns into a bill?

**Answer:** An idle VM or database left running; delete practice resource groups

Delete resource groups after practice.

**Q3.** Which two ways manage Azure?

**Answer:** The web-based Azure Portal and the az command-line interface (also Cloud Shell)

Cloud Shell has az installed and authenticated.

**Q4.** How do you log in on a machine without a browser?

**Answer:** az login --use-device-code

Open microsoft.com/devicelogin and enter the code.

### [02 · Azure AD &amp; RBAC / Security Basics](level-1/02-azure-ad-rbac-security.md)

**Q5.** What is Microsoft Entra ID?

**Answer:** Azure's identity service (formerly Azure AD), holding a tenant's users, groups and app registrations

Each subscription is linked to one tenant.

**Q6.** What three things define RBAC access?

**Answer:** Who (identity), what (role), where (scope)

Roles are sets of permitted actions.

**Q7.** What's the scope hierarchy?

**Answer:** Management Group → Subscription → Resource Group → Resource, with inheritance downward

A role assigned at a broader scope is inherited by everything beneath.

**Q8.** Why assign roles to groups?

**Answer:** Assign once to the group instead of per person

Identity types: user, group, service principal/managed identity.

### [03 · Virtual Machines](level-1/03-virtual-machines.md)

**Q9.** Which VM family is burstable and free-tier eligible?

**Answer:** B-series (e.g. Standard_B1s)

Low steady CPU with burst credits.

**Q10.** Which family suits balanced web apps, and which suits CPU-heavy batch?

**Answer:** D-series general purpose; F-series compute optimised

E-series is memory-optimised.

**Q11.** Which family suits in-memory caches and larger databases?

**Answer:** E-series (memory optimised)

For example Standard_E2s_v5.

**Q12.** How do you connect to a Linux VM?

**Answer:** SSH to its public IP (the CLI can look the IP up for you)

Stop idle VMs to avoid cost.

### [04 · Blob Storage](level-1/04-blob-storage.md)

**Q13.** What do you create first for Blob Storage?

**Answer:** A storage account, then containers holding blobs

Containers group blobs.

**Q14.** How can you authorise blob commands without account keys?

**Answer:** --auth-mode login using your RBAC identity

Or use a connection string.

**Q15.** Which commands upload, list and download blobs?

**Answer:** az storage blob upload, list and download

Used with a container name.

**Q16.** How much Blob Storage is 'Always Free' per the lesson?

**Answer:** 5 GB

Within usage caps.

### [05 · Virtual Networks Basics](level-1/05-virtual-networks.md)

**Q17.** What is a VNet?

**Answer:** A private network in Azure with an address space divided into subnets

Subnets are slices of the address space.

**Q18.** How do you add another subnet?

**Answer:** az network vnet subnet create on the same VNet

List with az network vnet subnet list.

**Q19.** What does an NSG on a subnet do?

**Answer:** Filters inbound and outbound traffic by rules

Network Security Groups apply allow/deny rules.

**Q20.** What must address ranges avoid?

**Answer:** Overlapping ranges between networks you may peer or connect

Plan address spaces up front.

### [06 · Azure SQL Database / Cosmos DB Basics](level-1/06-azure-sql-cosmos-db.md)

**Q21.** Which SQL tier is free-tier eligible in the lesson?

**Answer:** General Purpose serverless

Serverless can auto-pause.

**Q22.** Why add a firewall rule for your IP?

**Answer:** The server blocks connections by default, so your machine's IP must be allowed

'Allow Azure services' lets App Service or Functions connect.

**Q23.** How does Cosmos DB differ from Azure SQL?

**Answer:** Cosmos DB is a globally distributed NoSQL database; Azure SQL is a managed relational database

Cosmos has a free tier.

**Q24.** What's 'Allow Azure services' rule do?

**Answer:** Lets other Azure services such as App Service or Functions connect

Keep rules as narrow as possible.

### [07 · Azure Functions](level-1/07-azure-functions.md)

**Q25.** Which are Functions triggers?

**Answer:** HTTP, Timer, Blob, Queue, Cosmos DB (change feed) and Event Grid

A trigger fires the function.

**Q26.** What does the Consumption plan do?

**Answer:** Scales automatically and bills per execution, with a free monthly grant

Good for spiky workloads.

**Q27.** Why does a Function App need a storage account?

**Answer:** For internal state: triggers and logs

Created alongside the app.

**Q28.** When does a Timer trigger fire?

**Answer:** On a CRON-like schedule

NCRONTAB expressions.

### [08 · Azure Monitor &amp; Log Analytics](level-1/08-azure-monitor-log-analytics.md)

**Q29.** What are platform metrics?

**Answer:** Free, automatic numeric data like CPU percentage for Azure resources

Read via az monitor metrics list.

**Q30.** What is a Log Analytics workspace?

**Answer:** A store where logs are collected and queried

Queried with KQL.

**Q31.** What is KQL?

**Answer:** Kusto Query Language, used to query logs

Pipe-based queries.

**Q32.** Why average CPU in 5-minute buckets?

**Answer:** To read the last hour of trend with less noise

Aggregation interval.

### [09 · ARM Templates / Bicep Intro](level-1/09-arm-bicep-intro.md)

**Q33.** What is Bicep?

**Answer:** A concise declarative language that compiles to ARM templates

ARM templates are JSON.

**Q34.** Why prefer Bicep over raw ARM JSON?

**Answer:** It's shorter and easier to read and maintain

Same resources and deployment engine.

**Q35.** How do you install Bicep?

**Answer:** az bicep install

The CLI bundles it.

**Q36.** What does infrastructure as code give you?

**Answer:** Repeatable, reviewable deployments

Deploy with az deployment group create.

### [10 · Capstone Project — End-to-End Web App](level-1/10-capstone-project.md)

**Q37.** What does the capstone build?

**Answer:** An end-to-end web app: resource group, database, Function App backend and frontend

It combines earlier modules.

**Q38.** Why start with a resource group?

**Answer:** It holds related resources so they can be managed and deleted together

Deleting the group cleans up everything.

**Q39.** How is the backend delivered?

**Answer:** As an Azure Function App

Serverless HTTP backend.

**Q40.** What should you do at the end?

**Answer:** Delete the resource group to avoid costs

Cost awareness.

## Level 2 · Intermediate

### [01 · App Service &amp; Web Apps Deep Dive](level-2/01-app-service-deep-dive.md)

**Q41.** Which App Service tier is for learning only?

**Answer:** F1 (Free), with a capped daily CPU quota and no slots

Basic adds custom domains but no slots.

**Q42.** Which tiers support deployment slots and autoscale?

**Answer:** Standard (5 slots) and Premium v3 (20 slots)

Premium v3 adds VNet integration.

**Q43.** What do deployment slots do?

**Answer:** Let you deploy to staging and swap into production with minimal downtime

Swap staging with production.

**Q44.** What does an App Service plan define?

**Answer:** The compute (size and tier) that apps in it share

Multiple apps can share a plan.

### [02 · Storage Deep Dive (Files, Queues, Tables)](level-2/02-storage-deep-dive.md)

**Q45.** What is Azure Files?

**Answer:** Managed SMB/NFS file shares mountable like network drives

Blob is object storage.

**Q46.** What is Queue Storage for?

**Answer:** Decoupling components with messages processed asynchronously

A Queue trigger can drive a function.

**Q47.** What are Tables?

**Answer:** A NoSQL key-attribute store for structured data

Cheap, schemaless.

**Q48.** Why use queues between services?

**Answer:** A burst doesn't overwhelm the consumer; messages wait until processed

Load levelling and resilience.

### [03 · Networking Deep Dive (Load Balancer, App Gateway)](level-2/03-networking-deep-dive.md)

**Q49.** Which layer does Azure Load Balancer work at?

**Answer:** Layer 4 (TCP/UDP)

It distributes by IP and port.

**Q50.** What does Application Gateway add?

**Answer:** Layer 7 routing (HTTP paths, hosts), TLS termination and optionally WAF

Path-based routing is the typical use.

**Q51.** Which fits path-based routing /api versus /images?

**Answer:** Application Gateway

Load Balancer can't inspect HTTP.

**Q52.** What do health probes do?

**Answer:** Remove unhealthy backends from rotation

Both services use them.

### [04 · Azure Kubernetes Service (AKS) Basics](level-2/04-aks-basics.md)

**Q53.** What is AKS?

**Answer:** Azure Kubernetes Service: managed Kubernetes where Azure runs the control plane

You manage node pools and workloads.

**Q54.** What does `kubectl get nodes` show?

**Answer:** Node names, status (Ready), roles, age and version

Nodes are VM scale set instances.

**Q55.** How do you get kubectl credentials for AKS?

**Answer:** az aks get-credentials

It merges into kubeconfig.

**Q56.** Why delete AKS clusters after practice?

**Answer:** Nodes bill while running

Cost awareness.

### [05 · Azure DevOps &amp; CI/CD Pipelines](level-2/05-azure-devops-cicd.md)

**Q57.** What does Azure DevOps organise?

**Answer:** Organizations, projects, repos, pipelines, boards and artifacts

Repos host Git code.

**Q58.** Where is a pipeline defined?

**Answer:** YAML (azure-pipelines.yml) in the repo

Triggers, stages, jobs and steps.

**Q59.** What does the trigger in pipeline YAML do?

**Answer:** Starts runs on events such as pushes to a branch

For example branches: include: main.

**Q60.** Why pipeline-as-code?

**Answer:** Versioned and reviewable with the code

Same as app code.

### [06 · Managed Identities &amp; Key Vault](level-2/06-managed-identities-key-vault.md)

**Q61.** System-assigned versus user-assigned managed identity?

**Answer:** System-assigned is tied to one resource's lifecycle; user-assigned is created once and attached to many resources

Both remove stored credentials.

**Q62.** Why use a managed identity?

**Answer:** The app authenticates to Azure services without secrets in code

Azure manages the credentials.

**Q63.** What is Key Vault for?

**Answer:** Storing and controlling access to secrets, keys and certificates

Grant access via RBAC to the identity.

**Q64.** How should an app read a secret?

**Answer:** Through its managed identity from Key Vault, not a hard-coded value

No secrets in repos.

### [07 · Container Instances &amp; Container Registry](level-2/07-aci-acr.md)

**Q65.** What is Azure Container Registry?

**Answer:** A private registry for container images

Push images with az acr build or docker push.

**Q66.** What are Container Instances (ACI)?

**Answer:** Run a container serverlessly without managing VMs or clusters, billed per second

Good for simple or bursty workloads.

**Q67.** How can ACR build an image without local Docker?

**Answer:** az acr build (builds in the cloud)

Then deploy to ACI.

**Q68.** How should ACI pull from a private ACR?

**Answer:** With credentials or a managed identity granted pull access

Avoid admin user in production.

### [08 · Cosmos DB Deep Dive](level-2/08-cosmos-db-deep-dive.md)

**Q69.** What makes a good Cosmos DB partition key?

**Answer:** High cardinality with even request distribution, avoiding hot partitions

Queries that include the key are cheaper.

**Q70.** What is a Request Unit (RU)?

**Answer:** A normalised measure of the throughput cost of operations (reads, writes, queries)

You provision or autoscale RU/s.

**Q71.** What happens with a hot partition?

**Answer:** One partition gets most traffic and is throttled even if total RU/s is available

Choose keys carefully.

**Q72.** Why are cross-partition queries costly?

**Answer:** They fan out to many partitions, consuming more RUs

Include the partition key.

### [09 · Azure Functions Advanced](level-2/09-azure-functions-advanced.md)

**Q73.** What problem do Durable Functions solve?

**Answer:** Stateful, long-running workflows in serverless code, with checkpointing

Orchestrations survive restarts.

**Q74.** What are the three function types?

**Answer:** Client (starts), orchestrator (coordinates), activity (does work)

Orchestrators must be deterministic.

**Q75.** Why must orchestrator code be deterministic?

**Answer:** It's replayed from history, so non-deterministic calls would break replay

Do I/O in activities.

**Q76.** What pattern does chaining give?

**Answer:** Run activities in sequence, passing outputs along

Fan-out/fan-in is the parallel pattern.

### [10 · Project — Multi-Tier Web Application](level-2/10-project-multi-tier-app.md)

**Q77.** What does the multi-tier project architecture isolate?

**Answer:** Tiers into subnets of a VNet, with data reachable via a private endpoint

The database isn't exposed to the internet.

**Q78.** What does a private endpoint give Cosmos DB?

**Answer:** A private IP in your VNet so traffic stays off the public internet

Disable public network access.

**Q79.** Why start with resource group, VNet and subnets?

**Answer:** Network layout and boundaries underpin every later tier

Step 1 of the project.

**Q80.** What should NSGs allow between tiers?

**Answer:** Only the traffic each tier needs

Least privilege.

## Level 3 · Advanced

### [01 · Advanced Networking (VPN Gateway, ExpressRoute)](level-3/01-advanced-networking.md)

**Q81.** What is a site-to-site VPN Gateway?

**Answer:** An encrypted tunnel over the internet between an on-premises network and an Azure VNet

Needs a gateway subnet and on-prem device.

**Q82.** How does ExpressRoute differ from VPN?

**Answer:** A private dedicated connection through a provider, not over the public internet, with more predictable performance

Higher cost, higher reliability.

**Q83.** Which special subnet does a VPN gateway need?

**Answer:** GatewaySubnet

Name must be exact.

**Q84.** When choose ExpressRoute?

**Answer:** Large, latency-sensitive or compliance-driven hybrid connectivity

VPN suits modest needs.

### [02 · Infrastructure as Code at Scale (Bicep/Terraform)](level-3/02-iac-at-scale.md)

**Q85.** What do Bicep modules do?

**Answer:** Package reusable resource definitions called from other Bicep files

They enable composition.

**Q86.** How do parameter files help with environments?

**Answer:** One template, a parameter file per environment (dev, test, prod)

Same code, different values.

**Q87.** What does `what-if` do?

**Answer:** Previews changes a deployment would make before applying

Run it before every apply.

**Q88.** Bicep or Terraform?

**Answer:** Both express IaC; Bicep is Azure-native, Terraform is multi-cloud with state

Choose by needs.

### [03 · Advanced AKS (Helm, Ingress, Autoscaling)](level-3/03-advanced-aks.md)

**Q89.** What is Helm?

**Answer:** A package manager for Kubernetes: charts with configurable values

helm install deploys a release.

**Q90.** What does an Ingress do?

**Answer:** Routes external HTTP(S) to services, e.g. by path, via an ingress controller

Path-based routing is the lesson's example.

**Q91.** What does the Horizontal Pod Autoscaler scale?

**Answer:** Pod replicas based on metrics such as CPU

The cluster autoscaler scales nodes.

**Q92.** Why use values files in Helm?

**Answer:** To configure charts per environment without editing templates

Keep secrets out.

### [04 · Event-Driven Architecture (Event Grid, Service Bus)](level-3/04-event-driven-architecture.md)

**Q93.** What is Event Grid for?

**Answer:** Reactive, fan-out events (resource changes, custom events) delivered to many subscribers

Lightweight notifications.

**Q94.** What is Service Bus for?

**Answer:** Durable queues and topics for enterprise messaging with ordering, dead-lettering and transactions

Topics allow multiple subscriptions.

**Q95.** Which fits 'a blob was created, notify several systems'?

**Answer:** Event Grid

Fan-out of a notification.

**Q96.** Which fits 'process each order exactly via a durable queue'?

**Answer:** Service Bus

Messages must not be lost.

### [05 · API Management](level-3/05-api-management.md)

**Q97.** What is API Management (APIM)?

**Answer:** A gateway to publish, secure, throttle and monitor APIs in front of backends

Import a backend and apply policies.

**Q98.** What do APIM policies do?

**Answer:** Apply rate limiting, authentication and transformations to requests and responses (XML)

Applied at global, product, API or operation scope.

**Q99.** How do you import a backend API?

**Answer:** From an OpenAPI definition or an existing service

The gateway exposes operations.

**Q100.** Why rate-limit at the gateway?

**Answer:** To protect backends from abuse or overload

Policy rate-limit.

### [06 · Defender for Cloud &amp; Governance (Policy, Blueprints)](level-3/06-defender-governance.md)

**Q101.** What is Defender for Cloud?

**Answer:** A service providing security posture management and threat protection with a Secure Score

Enable plans per resource type.

**Q102.** What does Secure Score show?

**Answer:** A measure of your security posture with recommendations to improve it

Higher is better.

**Q103.** What does Azure Policy do?

**Answer:** Enforces or audits rules at scale (e.g. allowed regions, required tags) with effects like deny

Assign at management group or subscription scope.

**Q104.** Audit versus deny effect?

**Answer:** Audit reports non-compliance; deny blocks the non-compliant deployment

Start with audit then deny.

### [07 · High Availability &amp; Disaster Recovery](level-3/07-ha-disaster-recovery.md)

**Q105.** Availability Zones versus Availability Sets?

**Answer:** Zones are physically separate datacentres in a region; sets spread VMs across fault/update domains within a datacentre

Zones protect against datacentre failure.

**Q106.** What are RTO and RPO?

**Answer:** RTO: how long you can be down; RPO: how much data loss (time) is acceptable

They drive DR design.

**Q107.** What does Azure Site Recovery do?

**Answer:** Replicates VMs to another region for failover during a regional disaster

Cross-region DR.

**Q108.** Zone-redundant versus zonal PaaS?

**Answer:** Zone-redundant replicates across zones automatically; zonal pins to one zone

Choose by availability needs.

### [08 · CI/CD for Containers &amp; GitOps](level-3/08-cicd-containers-gitops.md)

**Q109.** What does the pipeline do for containers?

**Answer:** Builds the image, pushes it to ACR and updates a deployment

azure-pipelines.yml with Docker tasks.

**Q110.** What is GitOps?

**Answer:** Git is the source of truth for desired cluster state, applied automatically by an agent

Flux syncs a repo to the cluster.

**Q111.** What does Flux do on AKS?

**Answer:** Continuously reconciles the cluster to the Git repo's manifests

Drift is corrected.

**Q112.** Benefit of GitOps?

**Answer:** Auditable, reviewable changes and easy rollback via Git history

Pull-based deployment.

### [09 · Cost Management &amp; Optimization](level-3/09-cost-management-optimization.md)

**Q113.** How do budgets help?

**Answer:** Alert (or act) when spending crosses thresholds

Set alerts at percentages.

**Q114.** What does Azure Advisor recommend?

**Answer:** Right-sizing and cost, reliability, security and performance improvements

For example resize or shut down underused VMs.

**Q115.** What is right-sizing?

**Answer:** Matching resource size to actual usage to avoid overpaying

Check CPU and memory.

**Q116.** How can you read cost data?

**Answer:** Cost analysis grouped by resource group, tag or service, and via az consumption/CLI

Tags enable chargeback.

### [10 · Project — Highly Available Microservices](level-3/10-project-ha-microservices.md)

**Q117.** Why spread AKS nodes across zones?

**Answer:** Survive a zone failure

Zone spread with the cluster autoscaler.

**Q118.** What does the cluster autoscaler do?

**Answer:** Adds or removes nodes based on pending pods

Combined with HPA.

**Q119.** Why a network foundation first?

**Answer:** Subnets, NSGs and address plans underpin the cluster and data tier

Step 1.

**Q120.** What makes the microservices 'highly available'?

**Answer:** Redundancy across zones, health probes and autoscaling, with no single point of failure

Architecture section.

## Level 4 · Master

### [01 · Enterprise Landing Zones (CAF)](level-4/01-enterprise-landing-zones.md)

**Q121.** What is a landing zone?

**Answer:** A pre-configured, governed environment (management groups, policy, networking, identity) where workloads land, per the Cloud Adoption Framework

CAF defines the pattern.

**Q122.** What does the management group hierarchy do?

**Answer:** Organises subscriptions so policy and RBAC inherit downward

For example platform and landing zones branches.

**Q123.** How do you move a subscription into a landing zone?

**Answer:** Place it under the right management group so policies apply

az account management-group subscription add.

**Q124.** Why standardise landing zones?

**Answer:** Consistent guardrails and faster, safer onboarding of workloads

Governance at scale.

### [02 · Management Groups &amp; Governance at Scale](level-4/02-management-groups-governance.md)

**Q125.** How does policy inherit?

**Answer:** Downward through management groups, subscriptions, resource groups and resources

Assign once at a high scope.

**Q126.** What's the difference between exemptions and exclusions?

**Answer:** Exemptions are time-bound, auditable waivers for a resource; exclusions (notScopes) remove a scope from assignment

Prefer exemptions for traceability.

**Q127.** What is the evaluation behaviour with deny?

**Answer:** A deny effect blocks non-compliant creates/updates regardless of lower-level wishes

Deny wins over allow-like effects.

**Q128.** Why govern at management-group level?

**Answer:** Consistent rules across many subscriptions with less repetition

Scale.

### [03 · Advanced AKS (Service Mesh, Multi-Cluster)](level-4/03-advanced-aks-service-mesh.md)

**Q129.** What does a service mesh like Istio add?

**Answer:** mTLS, traffic management and observability between services via sidecars/proxies

Enabled as an AKS add-on.

**Q130.** What does mTLS enforcement do?

**Answer:** Requires mutual authentication and encryption between workloads

Strict mode rejects plaintext.

**Q131.** How do canary releases use traffic splitting?

**Answer:** Send a small percentage (e.g. 10%) to the new version, then increase if healthy

Weighted routes.

**Q132.** Why a multi-cluster setup?

**Answer:** Resilience, isolation and regional reach

At higher cost and complexity.

### [04 · Azure Arc &amp; Hybrid Cloud](level-4/04-azure-arc-hybrid-cloud.md)

**Q133.** What is Azure Arc?

**Answer:** Extends Azure management to servers and Kubernetes clusters outside Azure

On-prem or other clouds.

**Q134.** How do you Arc-enable a server?

**Answer:** Install the Connected Machine agent and onboard it to a resource group

It appears as an Azure resource.

**Q135.** What can you apply to Arc servers?

**Answer:** Azure Policy, monitoring and RBAC like native resources

One governance plane.

**Q136.** Why use Arc?

**Answer:** Consistent governance and tooling across hybrid estates

Hybrid management.

### [05 · Advanced Observability (App Insights, Tracing)](level-4/05-advanced-observability.md)

**Q137.** What is Application Insights?

**Answer:** APM that collects requests, dependencies, exceptions and traces for applications

Part of Azure Monitor.

**Q138.** What does distributed tracing show?

**Answer:** A request's path across services via an operation id, with timings

KQL: where operation_Id == ...

**Q139.** Which KQL fields help trace a request?

**Answer:** operation_Id, name, duration, success, cloud_RoleName

Order by timestamp.

**Q140.** Why add cloud role names?

**Answer:** To tell which service produced each telemetry item

Needed for service maps.

### [06 · Performance Engineering &amp; Load Testing](level-4/06-performance-load-testing.md)

**Q141.** What is Azure Load Testing for?

**Answer:** Generating load against an endpoint to measure performance and find limits

Configured with a YAML test config.

**Q142.** What's 'finding the breaking point'?

**Answer:** Increasing load until errors or latency exceed thresholds, to learn capacity limits

Do it in non-production.

**Q143.** Which metrics matter?

**Answer:** Response time percentiles, throughput and error rate

Percentiles show tail latency.

**Q144.** Why run load tests in CI?

**Answer:** Catch regressions before release

Fail the build on thresholds.

### [07 · Security Deep Dive (Sentinel, Zero Trust)](level-4/07-security-sentinel-zero-trust.md)

**Q145.** What is Microsoft Sentinel?

**Answer:** A cloud-native SIEM/SOAR that collects security data, detects threats and automates response

Built on a Log Analytics workspace.

**Q146.** What does the analytics rule detect?

**Answer:** More than 5 failed sign-ins from more than 2 countries for a user: a correlation of failures and locations

KQL with summarize and make_set.

**Q147.** What is Zero Trust?

**Answer:** Never trust, always verify: authenticate and authorise every request regardless of network location

Least privilege and assume breach.

**Q148.** What do analytics rules produce?

**Answer:** Alerts and incidents for investigation

Playbooks can automate response.

### [08 · Data Platform at Scale (Synapse, Data Factory)](level-4/08-data-platform-at-scale.md)

**Q149.** What does Data Factory do?

**Answer:** Orchestrates data pipelines that move and transform data between sources

Activities and triggers.

**Q150.** Synapse dedicated SQL pools versus serverless?

**Answer:** Dedicated: provisioned capacity for predictable heavy workloads; serverless: pay per query over data in the lake

Choose by workload shape.

**Q151.** When does serverless fit?

**Answer:** Ad-hoc exploration of files in a data lake

No capacity to manage.

**Q152.** What's the typical pattern?

**Answer:** Ingest with Data Factory, store in the lake, analyse in Synapse

Platform at scale.

### [09 · FinOps &amp; Cost Governance at Scale](level-4/09-finops-cost-governance.md)

**Q153.** Why tagging as the foundation of FinOps?

**Answer:** Tags attribute costs to owners, projects and environments

Enforce with Policy.

**Q154.** Showback versus chargeback?

**Answer:** Showback reports costs to teams; chargeback actually bills them internally

Both need good allocation.

**Q155.** How do you enforce tags?

**Answer:** Azure Policy requiring or inheriting tags

Deny or modify effects.

**Q156.** What's FinOps?

**Answer:** A culture and practice of cloud financial accountability across engineering, finance and business

Continuous optimisation.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone begin with?

**Answer:** A management group hierarchy and subscription vending

Governance first.

**Q158.** What is subscription vending?

**Answer:** Automated creation of subscriptions placed in the right management group with baseline policies

Repeatable onboarding.

**Q159.** What's a baseline governance initiative?

**Answer:** A bundle of policies applied together as standard guardrails

Step 2.

**Q160.** What does the capstone integrate?

**Answer:** Landing zones, governance, networking, security, observability and cost controls

All levels.

## Mixed questions across all levels

**Q161.** A new Azure free account is created. What happens when the credit or 30 days runs out?

**Answer:** The trial stops working unless you manually upgrade; it won't auto-charge

A card is only for identity verification.

**Q162.** An idle test VM is left running for a month. What's the likely result?

**Answer:** A bill: idle VMs and databases are the #1 way a free trial costs money; delete practice resource groups

Delete the resource group to clean up everything.

**Q163.** A user needs read access to every resource in one resource group, and nothing else. Which RBAC setup?

**Answer:** Assign Reader to the user (or a group) at that resource group's scope

Narrow scope and least privilege; roles inherit downward.

**Q164.** Which VM family suits a low-traffic dev box eligible for the free tier?

**Answer:** B-series (burstable), e.g. Standard_B1s

F is compute-optimised, E memory-optimised, D general purpose.

**Q165.** A blob upload command should avoid account keys. Which option?

**Answer:** --auth-mode login with an RBAC identity

Data-plane RBAC avoids key sprawl.

**Q166.** A website needs routing /api to one backend pool and /images to another. Which service?

**Answer:** Application Gateway (layer 7)

Load Balancer works on IP and port.

**Q167.** A burst of orders must not overwhelm a slow processor. What decouples them?

**Answer:** A queue (Storage Queue or Service Bus) between them

Messages wait until the consumer is ready.

**Q168.** Event: 'a blob was created; notify five systems'. Event Grid or Service Bus?

**Answer:** Event Grid: reactive fan-out notifications

Service Bus fits durable, ordered business messages.

**Q169.** Your Cosmos DB uses a constant partition key. What goes wrong?

**Answer:** A hot partition: one physical partition takes all traffic and is throttled even if total RU/s is available

Choose a high-cardinality key. Verified with a small distribution check.

**Q170.** A Function App needs a secret. How should it get it?

**Answer:** Through a managed identity reading Key Vault, with no secret in code

Grant the identity access via RBAC.

**Q171.** Orchestrations must survive restarts and be replayed. Which Functions feature?

**Answer:** Durable Functions (orchestrator, activity, client), with deterministic orchestrator code

I/O belongs in activities.

**Q172.** Which App Service tier is the first with deployment slots?

**Answer:** Standard (S1+), with 5 slots

Premium v3 offers 20 slots.

**Q173.** A deployment is previewed before applying. Which command?

**Answer:** az deployment group what-if (Bicep/ARM)

Run it before every apply.

**Q174.** You need a private, dedicated connection from on-prem to Azure with predictable performance. Which?

**Answer:** ExpressRoute

VPN is encrypted over the public internet.

**Q175.** An RTO of 1 hour and RPO of 5 minutes mean:

**Answer:** Service must be back within an hour, with at most five minutes of data loss

RTO is time to recover; RPO is data loss window.

**Q176.** Which policy effect blocks non-compliant resource creation?

**Answer:** Deny (Audit only reports)

Start with audit, then deny.

**Q177.** Where should most guardrails be assigned so many subscriptions inherit them?

**Answer:** At a management group

Policy and RBAC inherit downward.

**Q178.** Where do you analyse a request's path across microservices in Application Insights?

**Answer:** Distributed tracing via operation_Id, with cloud_RoleName identifying each service

KQL with where operation_Id == ...

**Q179.** Costs must be attributed to teams. What foundation do you need?

**Answer:** A consistent tagging strategy enforced by Azure Policy

Tags drive showback and chargeback.

## Practical coding questions

### Plan non-overlapping subnets

Allocate subnets from a 10.0.0.0/16 VNet for web (120 hosts), app (250), db (60) and gateway (20). Azure reserves 5 addresses per subnet, so each needs hosts + 5. Use the smallest power-of-two block, largest first, aligned and never overlapping. Print each subnet and its usable count, then whether any overlap.

Starter code:

```
import ipaddress

VNET = ipaddress.ip_network("10.0.0.0/16")
WANTED = {"web": 120, "app": 250, "db": 60, "gateway": 20}   # hosts needed per subnet

def plan(vnet, wanted):
    # largest first; each subnet needs hosts + 5 (Azure reserves 5 addresses);
    # pick the smallest power-of-two block, align it, and never overlap
    pass

result = plan(VNET, WANTED)
for name in WANTED:
    n = result[name]
    print(f"{name}: {n} usable={n.num_addresses - 5}")
nets = list(result.values())
print("overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
```

Expected output:

```text
web: 10.0.1.0/25 usable=123
app: 10.0.0.0/24 usable=251
db: 10.0.1.128/25 usable=123
gateway: 10.0.2.0/27 usable=27
overlaps: False
```

??? success "Show a solution"

    ```
    import ipaddress

    VNET = ipaddress.ip_network("10.0.0.0/16")
    WANTED = {"web": 120, "app": 250, "db": 60, "gateway": 20}   # hosts needed per subnet

    def plan(vnet, wanted):
        """Allocate the smallest subnet that fits each request (Azure reserves 5 addresses per subnet)."""
        cursor = int(vnet.network_address)
        out = {}
        for name, hosts in sorted(wanted.items(), key=lambda kv: -kv[1]):
            needed = hosts + 5
            prefix = 32
            while 2 ** (32 - prefix) < needed:
                prefix -= 1
            size = 2 ** (32 - prefix)
            cursor = (cursor + size - 1) // size * size      # align to the subnet size
            net = ipaddress.ip_network((cursor, prefix))
            assert net.subnet_of(vnet)
            out[name] = net
            cursor += size
        return out

    result = plan(VNET, WANTED)
    for name in WANTED:
        n = result[name]
        print(f"{name}: {n} usable={n.num_addresses - 5}")
    nets = list(result.values())
    print("overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
    ```

    Azure reserves five addresses in each subnet, so a /24 gives 251 usable. Planning address space up front avoids overlaps that block peering later. Verified by running the solution.

### Spot a hot partition

Write hot_partitions(keys, partitions=4, threshold=0.4): hash each key to a partition (sum of character codes modulo partitions) and return {partition: share} for any partition holding more than the threshold of requests. Compare a well-distributed key set with a tenant-skewed one.

Starter code:

```
from collections import Counter

def hot_partitions(keys, partitions=4, threshold=0.4):
    # hash each key to a partition (sum of char codes % partitions),
    # return {partition: share} for partitions holding more than `threshold` of requests
    pass

good = [f"user-{i}" for i in range(100)]
bad = ["tenant-a"] * 70 + [f"tenant-{i}" for i in range(30)]
print("good:", hot_partitions(good))
print("bad:", hot_partitions(bad))
```

Expected output:

```text
good: {}
bad: {0: 0.78}
```

??? success "Show a solution"

    ```
    from collections import Counter

    def hot_partitions(keys, partitions=4, threshold=0.4):
        """Hash each partition key to a physical partition and flag any holding more than `threshold` of requests."""
        def h(k):  # simple stable hash
            return sum(ord(c) for c in k) % partitions
        counts = Counter(h(k) for k in keys)
        total = len(keys)
        return {p: round(c / total, 2) for p, c in sorted(counts.items()) if c / total > threshold}

    good = [f"user-{i}" for i in range(100)]
    bad = ["tenant-a"] * 70 + [f"tenant-{i}" for i in range(30)]
    print("good:", hot_partitions(good))
    print("bad:", hot_partitions(bad))
    ```

    A partition key with low cardinality or skew concentrates traffic on one physical partition, which gets throttled even when total RU/s is available. Verified by running the solution.

---

More free practice: the [Azure Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
