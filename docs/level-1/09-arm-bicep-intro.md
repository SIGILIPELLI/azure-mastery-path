# 09 · ARM Templates / Bicep Intro

Every command you've run in this course so far (`az group create`,
`az vm create`, ...) is **imperative**: you tell Azure the steps to take.
**Infrastructure as Code (IaC)** flips that around — you write a
**declarative** description of what you want to exist, and a tool
reconciles reality to match it, safely re-runnable any number of times.
Azure's native IaC formats are **ARM templates** (JSON) and **Bicep** (a
cleaner syntax that compiles down to ARM JSON) — this module focuses on
Bicep, since it's what Microsoft now recommends for all new work.

## ARM templates vs. Bicep

An **ARM template** is a JSON document describing one or more resources,
their properties, and how they depend on each other. It works, but JSON is
verbose and unforgiving to hand-write — lots of brackets, no comments in
strict JSON, and expressions like `[resourceId(...)]` embedded as strings.

**Bicep** is a domain-specific language that compiles 1:1 to ARM JSON, but
reads like a typed configuration file: no brackets-as-strings, real
comments, module reuse, and editor tooling with autocomplete. Nobody should
be hand-writing new ARM JSON in 2026 — write Bicep and let the tooling
generate JSON only where something downstream still needs it (e.g. a
CI system that only accepts raw ARM).

## Install Bicep

The `az` CLI can manage the Bicep compiler for you:

```bash
az bicep install
az bicep upgrade
az bicep version
```

## Your first Bicep file

`main.bicep`:

```bicep
@description('Name of the storage account (must be globally unique)')
param storageAccountName string = 'stbiceplearn${uniqueString(resourceGroup().id)}'

@description('Azure region for all resources')
param location string = resourceGroup().location

@allowed([
  'Standard_LRS'
  'Standard_ZRS'
  'Standard_GRS'
])
param storageSku string = 'Standard_LRS'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
}

output storageAccountId string = storageAccount.id
output primaryEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

Key things to notice, compared to the raw `az storage account create` from
[Module 04](04-blob-storage.md):

- `param` declares an input with an optional default and validation
  (`@allowed`) — no more remembering valid SKU strings from memory.
- `resource <symbolicName> '<type>@<api-version>' = { ... }` declares a
  resource. The symbolic name (`storageAccount`) is how you refer to it
  elsewhere in the same file — it's not the Azure resource name.
  `uniqueString(resourceGroup().id)` deterministically generates a short
  unique suffix so the same template is re-runnable without a name clash.
- `output` exposes values (like the resulting endpoint URL) back to
  whoever deployed the template, or to a calling module.

## Deploy a Bicep file

```bash
az group create --name rg-bicep-demo --location eastus

az deployment group create \
  --resource-group rg-bicep-demo \
  --template-file main.bicep \
  --parameters storageSku=Standard_LRS
```

Behind the scenes this compiles `main.bicep` to ARM JSON and submits it to
Azure Resource Manager, which computes a deployment plan and creates/updates
resources to match. Run the exact same command again with no changes to the
file, and Azure reports "no changes" — this **idempotency** is the core
value of declarative IaC over imperative scripts.

### Preview changes before applying (what-if)

```bash
az deployment group what-if \
  --resource-group rg-bicep-demo \
  --template-file main.bicep \
  --parameters storageSku=Standard_GRS
```

`what-if` shows a diff (create/modify/delete) without actually changing
anything — always run this before applying a template to production-like
resources.

## A multi-resource example

Bicep resources can reference each other by symbolic name, and Azure
Resource Manager figures out the dependency order automatically:

```bicep
param location string = resourceGroup().location
param vnetName string = 'vnet-bicep-demo'

resource vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.10.0.0/16']
    }
    subnets: [
      {
        name: 'subnet-app'
        properties: {
          addressPrefix: '10.10.1.0/24'
        }
      }
    ]
  }
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name: 'nsg-bicep-demo'
  location: location
  properties: {
    securityRules: [
      {
        name: 'allow-https'
        properties: {
          priority: 200
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

// Referencing vnet.name / vnet.properties creates an implicit dependency --
// this subnet update is guaranteed to run after the VNet exists.
resource subnetNsg 'Microsoft.Network/virtualNetworks/subnets@2023-09-01' = {
  parent: vnet
  name: 'subnet-app'
  properties: {
    addressPrefix: '10.10.1.0/24'
    networkSecurityGroup: {
      id: nsg.id
    }
  }
}
```

`parent: vnet` (instead of manually building the nested resource name
string) is Bicep's way of expressing "this subnet belongs to that VNet" —
one of the ergonomic wins over hand-written ARM JSON.

## Converting between ARM and Bicep

```bash
# Decompile an existing ARM JSON template to Bicep
az bicep decompile --file azuredeploy.json

# Compile Bicep down to ARM JSON (e.g. for a pipeline that only accepts JSON)
az bicep build --file main.bicep
```

`az bicep decompile` is a great way to learn Bicep syntax from templates
you export from the Portal (every resource's blade has an "Export template"
option that gives you its current ARM JSON).

## How It Actually Works

Bicep isn't a separate deployment engine — it's a **domain-specific language
that compiles down to ARM JSON** (`az deployment group create` on a `.bicep`
file first transpiles it to the equivalent ARM template in memory, then
submits that JSON to the exact same ARM deployment API used by hand-written
templates or the Portal's "Export template" feature). ARM itself processes
a deployment as a **declarative graph**: it parses the template's
`resources` array, builds a dependency graph from explicit `dependsOn`
entries and implicit references (`reference()`/symbolic references pulled
from one resource's properties into another's), topologically sorts that
graph, and then provisions resources in parallel wherever the graph allows
it — this is why two unrelated resources in the same template deploy
concurrently while a NIC correctly waits for its VNet/subnet to exist first,
without you writing any explicit ordering logic.

**Idempotency** is enforced by ARM computing a diff against the resource's
last-known desired state (tracked per deployment in the resource group's
deployment history) rather than blindly re-running create calls — re-
deploying an unchanged template is a no-op at the resource-provider level
because ARM detects no property delta and skips the PUT. **What-if**
(`az deployment group what-if`) works by having ARM run this same diff
engine against your *proposed* template without actually calling any
resource provider's write API, which is how it can report additions/
modifications/deletions before you commit. Because Bicep compiles to plain
ARM JSON, there's no runtime dependency on Bicep at deployment time — the
CLI's compile step is purely a client-side convenience, and the resource
group's deployment history only ever stores the resulting ARM template, not
your original Bicep source.

## Cheat sheet

| Command / syntax | Purpose |
|---|---|
| `az bicep install` / `upgrade` | Install or update the Bicep compiler. |
| `param <name> <type> = <default>` | Declare an input parameter. |
| `resource <symbolicName> '<type>@<api>' = {...}` | Declare a resource. |
| `output <name> <type> = <expr>` | Expose a value after deployment. |
| `az deployment group create --template-file` | Deploy a Bicep/ARM file to a resource group. |
| `az deployment group what-if --template-file` | Preview changes before applying. |
| `az bicep build` / `decompile` | Convert Bicep → ARM JSON, or JSON → Bicep. |

## Exercise

1. Write a Bicep file that declares a storage account (parameterized name
   and SKU) and a blob container inside it
   (`Microsoft.Storage/storageAccounts/blobServices/containers`).
2. Run `az deployment group what-if` against a new resource group to
   preview it, then `az deployment group create` to actually deploy it.
3. Re-run the same `create` command unchanged and confirm Azure reports no
   changes needed — that's idempotency in action.
4. Change the SKU parameter and re-deploy, using `what-if` first to see
   exactly what will change.
5. Delete the resource group when finished.
