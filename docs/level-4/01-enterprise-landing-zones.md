# 01 · Enterprise Landing Zones (CAF)

Everything through Level 3 assumed a single subscription. Real enterprises
run dozens to hundreds, each needing consistent network connectivity,
identity, policy, and cost boundaries from day one — not bolted on later.
The **Cloud Adoption Framework (CAF) Enterprise-Scale Landing Zone** is
Microsoft's reference architecture for this: a management group hierarchy,
a small set of platform subscriptions, and policy-driven guardrails that
every new "landing zone" subscription inherits automatically.

## The management group hierarchy

```text
Tenant Root Group
└── alz (intermediate root)
    ├── Platform
    │   ├── Management      (Log Analytics, Automation)
    │   ├── Connectivity     (hub VNets, ExpressRoute/VPN, Firewall)
    │   └── Identity         (domain controllers, if applicable)
    ├── Landing Zones
    │   ├── Corp             (internal-facing apps, no direct internet)
    │   └── Online            (internet-facing apps)
    ├── Sandbox               (isolated, no policy inheritance from prod)
    └── Decommissioned
```

```bash
az account management-group create --name alz --display-name "Enterprise-Scale"
az account management-group create --name alz-platform --display-name "Platform" --parent alz
az account management-group create --name alz-connectivity --display-name "Connectivity" --parent alz-platform
az account management-group create --name alz-landingzones --display-name "Landing Zones" --parent alz
az account management-group create --name alz-corp --display-name "Corp" --parent alz-landingzones
az account management-group create --name alz-online --display-name "Online" --parent alz-landingzones
```

**Gotcha:** management group changes (creating, moving a subscription
between groups) can take several minutes to propagate through Azure's
control plane, and policy assignments at a management group scope don't
retroactively evaluate existing resources until the next compliance scan —
plan any reorg during a low-change window, not mid-incident.

## Moving a subscription into a landing zone

```bash
az account management-group subscription add \
  --name alz-corp \
  --subscription "11111111-2222-3333-4444-555555555555"
```

The moment a subscription lands under `alz-corp`, it inherits every policy
assignment scoped to `alz`, `alz-platform`→`alz-landingzones`, and
`alz-corp` in the chain above it — this is the entire point of the
hierarchy: a new application team gets a subscription, and secure
networking, logging, and policy compliance already apply without the
platform team touching that subscription individually.

## Hub-and-spoke at the platform level

The **Connectivity** subscription hosts the hub VNet (from
[Level 3, Module 01](../level-3/01-advanced-networking.md)) shared by every
landing zone subscription's spoke VNet, via **VNet peering across
subscriptions** (peering isn't limited to one subscription):

```bash
az network vnet peering create \
  --resource-group rg-connectivity \
  --name hub-to-corp-app1 \
  --vnet-name vnet-hub \
  --remote-vnet "/subscriptions/aaaa/resourceGroups/rg-corp-app1/providers/Microsoft.Network/virtualNetworks/vnet-spoke-app1" \
  --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
```

**Gotcha:** cross-subscription peering requires the `az network vnet
peering create` command to be run with credentials that have `Network
Contributor` on **both** VNets, or with the remote VNet ID passed and a
matching peering created from the other side separately — a common failure
is creating the peering from the hub side only and forgetting the spoke
side's reciprocal peering, which leaves the connection in
`Disconnected` state indefinitely.

## Subscription vending

**Subscription vending** is the automated, self-service process of
provisioning a new landing zone subscription with all baseline
configuration (policy assignment, budget, network peering, RBAC) applied
consistently, typically via a Bicep/Terraform module triggered by a
pipeline rather than manual portal clicks:

```bicep
// landing-zone-vending.bicep (simplified)
param subscriptionDisplayName string
param managementGroupId string = 'alz-corp'
param budgetAmount int = 5000

module policyAssignment 'modules/policy-assignment.bicep' = {
  name: 'assign-baseline-policy'
  params: {
    scope: subscription()
    initiativeId: baselineInitiativeId
  }
}

module budget 'modules/budget.bicep' = {
  name: 'create-budget'
  params: {
    amount: budgetAmount
  }
}
```

**Gotcha:** subscription vending pipelines are a high-value target for
misconfiguration — a bug in the vending template that, say, forgets to
assign the "deny public IP" policy means every new subscription created
through the (supposedly hardened) automated path is silently
non-compliant from birth; treat the vending template itself as
security-critical code requiring review, not a one-off script.

## Landing zone archetypes: Corp vs. Online

| | Corp | Online |
|---|---|---|
| **Internet exposure** | None direct — routed through hub firewall | Direct (with WAF/DDoS in front) |
| **Typical workloads** | Internal line-of-business apps | Public-facing web/API apps |
| **Network egress** | Forced through hub NVA/firewall | May egress directly, depending on design |
| **Policy strictness** | High (internal data) | High but includes internet-facing baselines (WAF required, DDoS Standard) |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az account management-group create --parent` | Build the management group hierarchy. |
| `az account management-group subscription add` | Move a subscription into a landing zone. |
| `az network vnet peering create` (cross-subscription) | Peer a spoke in one subscription to a hub in another. |
| `az account management-group list` | Inspect the current hierarchy. |
| `az policy assignment list --scope <mg-id>` | See what a management group scope enforces. |

## Exercise

1. Build a 3-level management group hierarchy (root → Platform/Landing
   Zones → Corp/Online) using `az account management-group create`.
2. Move a test subscription into the `Corp` landing zone and confirm (via
   `az policy assignment list`) it now inherits a management-group-level
   policy.
3. Design (on paper) a cross-subscription hub-and-spoke peering and
   identify which credentials/roles are needed on each side.
4. Sketch the inputs a subscription vending Bicep module would need
   (display name, budget, management group, network CIDR) and why treating
   this template as security-critical matters.
5. Clean up the test management groups and subscription move when finished.
