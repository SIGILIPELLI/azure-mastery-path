# 06 · Defender for Cloud & Governance (Policy, Blueprints)

Individually secure resources don't add up to a secure environment without
something enforcing consistency across all of them and watching for
threats. This module covers **Microsoft Defender for Cloud** (posture
management and threat detection) and **Azure Policy** (enforcing rules at
scale, e.g. "every storage account must disable public blob access").

## Enabling Defender for Cloud plans

Defender for Cloud's free tier gives baseline recommendations; paid plans
per resource type add active threat detection:

```bash
az security pricing create \
  --name VirtualMachines \
  --tier Standard

az security pricing create \
  --name StorageAccounts \
  --tier Standard

az security pricing list --query "[].{name:name, tier:pricingTier}" -o table
```

```text
Name                Tier
------------------  --------
VirtualMachines     Standard
StorageAccounts     Standard
AppServices         Free
SqlServers          Free
```

**Gotcha:** Defender plans are billed **per resource, per hour**, not
per-subscription flat rate — enabling `Standard` tier on `VirtualMachines`
for a subscription with hundreds of VMs is a very different cost than a
dev subscription with three, and it's easy to enable a plan subscription-
wide, forget, and get a much larger bill than expected the following month.

## Reading the Secure Score

```bash
az security secure-scores list --query "[].{score:score.current,max:score.max}" -o table

az security assessment list \
  --query "[?status.code=='Unhealthy'].{name:displayName, severity:metadata.severity}" \
  -o table
```

The **Secure Score** is a percentage across weighted recommendations
(enable MFA, encrypt disks, close open management ports); it's a prioritization
tool, not a compliance certificate — a 100% score does not mean "no
security risk," it means every recommendation Defender currently checks is
addressed.

## Azure Policy: enforcing rules at scale

A **policy definition** describes a rule; an **assignment** applies it to a
scope (management group, subscription, or resource group):

```bash
az policy definition create \
  --name "deny-public-blob-access" \
  --display-name "Deny storage accounts with public blob access" \
  --rules '{
    "if": {
      "allOf": [
        { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
        { "field": "Microsoft.Storage/storageAccounts/allowBlobPublicAccess", "equals": "true" }
      ]
    },
    "then": { "effect": "deny" }
  }'

az policy assignment create \
  --name "deny-public-blob-prod" \
  --policy "deny-public-blob-access" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-prod"
```

```text
$ az storage account create -g rg-prod -n stpublictest --allow-blob-public-access true
(RequestDisallowedByPolicy) Resource 'stpublictest' was disallowed by policy.
```

**Gotcha:** policies with effect `Deny` block *new* non-compliant resources
but do **not** retroactively fix existing ones — an account created before
the policy existed stays non-compliant until you either fix it manually or
trigger a **remediation task** for policies with a `DeployIfNotExists`
effect, which is a separate step (`az policy remediation create`) that
people often assume happens automatically.

## Initiatives (policy sets)

An **initiative** groups related policies (e.g. an entire compliance
framework like CIS or NIST) so you assign and track compliance as one unit
instead of dozens of individual assignments:

```bash
az policy set-definition create \
  --name "baseline-security-initiative" \
  --display-name "Baseline Security Initiative" \
  --definitions '[
    { "policyDefinitionId": "/subscriptions/xxxx/providers/Microsoft.Authorization/policyDefinitions/deny-public-blob-access" },
    { "policyDefinitionReferenceId": "require-https", "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/404c3081-a854-4457-ae30-26a93ef643f9" }
  ]'

az policy assignment create \
  --name "baseline-security-prod" \
  --policy-set-definition "baseline-security-initiative" \
  --scope "/subscriptions/$(az account show --query id -o tsv)"
```

## Blueprints → Template Specs / deployment stacks

Azure Blueprints (which bundled policies, role assignments, and ARM
templates as one versioned artifact) is **deprecated** in favor of
combining **Azure Policy initiatives** with **Template Specs** or
**deployment stacks** for the resource-provisioning half:

```bash
az ts create \
  --name "landing-zone-baseline" \
  --version "1.0" \
  --resource-group rg-shared \
  --template-file landing-zone.bicep
```

**Gotcha:** if you inherited an environment using Blueprints, plan a
migration rather than extending it further — Microsoft's guidance is that
existing Blueprint assignments continue to function but no new features
land there, and net-new governance-as-code should use Policy initiatives +
Template Specs / deployment stacks instead.

## How It Actually Works

**Microsoft Defender for Cloud** doesn't run as an agent you deploy
manually into every subscription — it operates by continuously querying
the ARM control plane's resource graph (the same underlying inventory
`az resource list` reads from) against a set of **security
recommendations**, each backed by a specific ARM/API check (e.g. "is
`Microsoft.Storage/storageAccounts.properties.minimumTlsVersion` set to
TLS1_2") evaluated on a recurring scan cycle; for deeper runtime signals
(process activity, network connections inside a VM) it deploys the Azure
Monitor Agent or relies on Defender's own sensors sending telemetry to a
Log Analytics workspace, where Defender's detection engine correlates that
stream against known attack patterns to raise alerts — recommendations
(posture) and alerts (active threats) are genuinely two different pipelines
feeding the same Secure Score.

**Azure Blueprints** (and their successor, Template Specs + deployment
stacks) work by packaging a set of ARM/Bicep artifacts — role assignments,
policy assignments, and resource templates — into one **versioned,
lockable bundle**; assigning a blueprint to a subscription doesn't just
deploy the artifacts once, it creates a tracked blueprint assignment
resource that can enforce **resource locks** the blueprint itself defines,
preventing someone from deleting or modifying an artifact the blueprint
created even if their RBAC role would otherwise permit it — the lock is
checked by ARM at request time exactly like a manually-applied
`CanNotDelete` lock, just applied and lifecycle-managed by the blueprint
assignment instead of a person. This is the mechanical difference between
a blueprint and a plain template deployment: the blueprint assignment
persists as a governance object ARM continues to enforce, not just a
one-time provisioning action.

## Cheat sheet

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

## Exercise

1. Enable the `Standard` Defender plan for `StorageAccounts` and read the
   current Secure Score for your subscription.
2. Create and assign a custom policy denying storage accounts with public
   blob access at a resource group scope; confirm creation is blocked.
3. Find one existing non-compliant resource (or create one before the
   policy existed) and trigger a remediation task for it.
4. Group two policies into an initiative and assign the initiative instead
   of the individual policies.
5. Delete the resource group when finished.
