# 06 · Managed Identities & Key Vault

Every module so far that needed a secret (a Cosmos DB key, a storage
account key) put it in an app setting or environment variable — better than
hardcoding it in source, but it's still a credential sitting somewhere that
could leak. This module covers the two pieces that get you to **zero
stored credentials**: **managed identities** (an app's own Azure AD
identity, with no password to manage) and **Key Vault** (centralized,
audited secret storage).

## Managed identities: system-assigned vs. user-assigned

A **managed identity** is an Azure AD identity that Azure creates and
rotates the credentials for automatically — your code never sees a
password or client secret, it just asks the local metadata endpoint for a
token.

- **System-assigned**: tied 1:1 to a single resource's lifecycle. Delete
  the Web App, the identity is deleted with it.
- **User-assigned**: created independently, can be attached to multiple
  resources at once, and outlives any single resource.

```bash
# System-assigned: enable directly on an existing Web App
az webapp identity assign \
  --name webapp-demo \
  --resource-group rg-appsvc-demo

# User-assigned: create once, attach to as many resources as needed
az identity create \
  --name id-shared \
  --resource-group rg-appsvc-demo

IDENTITY_ID=$(az identity show --name id-shared --resource-group rg-appsvc-demo --query id --output tsv)

az webapp identity assign \
  --name webapp-demo \
  --resource-group rg-appsvc-demo \
  --identities $IDENTITY_ID
```

**Gotcha:** having an identity doesn't grant it any permissions — it's
just an identity, like creating a user account with no group memberships.
You still assign it access via RBAC role assignment or a Key Vault access
policy, covered below.

## Key Vault

**Key Vault** stores secrets, keys, and certificates centrally, with
fine-grained access control and a full audit log of every read — instead
of a secret living in ten different app settings across ten apps, it lives
in one vault that all ten apps read from at runtime.

```bash
az keyvault create \
  --name kv-demo$RANDOM \
  --resource-group rg-appsvc-demo \
  --location eastus \
  --enable-rbac-authorization true

KEYVAULT=$(az keyvault list --resource-group rg-appsvc-demo --query "[0].name" --output tsv)

az keyvault secret set \
  --vault-name $KEYVAULT \
  --name CosmosDbKey \
  --value "<the-actual-cosmos-key>"
```

`--enable-rbac-authorization true` makes the vault use standard Azure RBAC
role assignments for access (the modern approach); the legacy alternative
is vault-specific **access policies** (`az keyvault set-policy`) — new
vaults should use RBAC unless you have a specific reason not to.

## Granting a managed identity access to Key Vault

```bash
PRINCIPAL_ID=$(az webapp identity show \
  --name webapp-demo --resource-group rg-appsvc-demo \
  --query principalId --output tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name $KEYVAULT --query id --output tsv)
```

**Key Vault Secrets User** grants read-only access to secret *values*
(not the ability to create/delete secrets or manage the vault itself) —
the least-privilege role an app consuming secrets actually needs.

## Reading secrets without an SDK: Key Vault references

App Service and Functions support **Key Vault references** — an app
setting whose *value* is a pointer to a vault secret, resolved
automatically at startup, so your code reads a normal environment
variable and never calls the Key Vault SDK at all:

```bash
az webapp config appsettings set \
  --name webapp-demo \
  --resource-group rg-appsvc-demo \
  --settings COSMOS_KEY="@Microsoft.KeyVault(SecretUri=https://$KEYVAULT.vault.azure.net/secrets/CosmosDbKey/)"
```

The app reads `os.environ["COSMOS_KEY"]` exactly as if it were a plain app
setting — App Service resolves the `@Microsoft.KeyVault(...)` reference
using the app's own managed identity behind the scenes, and the
role assignment above is what makes that resolution succeed.

**Gotcha:** if the role assignment is missing or wrong, the app setting's
value in the portal shows as `#REF!` instead of resolving — that's the
tell that it's a permissions problem, not a syntax problem with the
reference string itself.

## Using the identity from code (no reference, direct SDK call)

For code paths that need to call Key Vault (or any other Azure service)
programmatically rather than via an app setting, `DefaultAzureCredential`
automatically finds and uses whichever managed identity the compute
resource has:

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://kv-demo.vault.azure.net/",
    credential=credential,
)
secret = client.get_secret("CosmosDbKey")
print(secret.value)
```

`DefaultAzureCredential` tries several credential sources in order (managed
identity first when running in Azure, then environment variables, then
Azure CLI login) — the same code runs unchanged locally (using your `az
login` session) and in production (using the managed identity), with zero
`if environment == "prod"` branching.

## How It Actually Works

A **system-assigned managed identity** is implemented by Azure creating a
hidden service principal in your Entra tenant tied 1:1 to the resource's
lifecycle, and injecting a small **metadata endpoint** into that resource's
runtime environment (a link-local address, `169.254.169.254`, unreachable
from outside the instance) that your code calls to fetch a token — when
your app requests a token from that endpoint, the underlying platform
(App Service, a VM's IMDS extension, etc.) authenticates on your app's
behalf to Entra ID using a certificate it manages entirely outside your
code, and hands back a short-lived access token. This is the actual reason
managed identities eliminate secrets from your code: your application
never sees or handles any credential at all — only the token, which expires
in about an hour and is silently refreshed by the same metadata call.

**Key Vault** doesn't store secrets in a general-purpose database — secrets,
keys, and certificates are held in a **hardware security module (HSM)-backed
or software-isolated vault** (Premium tier uses FIPS 140-2 Level 2 HSMs)
where cryptographic keys never leave the HSM boundary in plaintext: an
operation like "decrypt with this key" is executed *inside* the HSM and
only the result crosses back over the API, which is why Key Vault can
offer "keys that can't be exported" as a real security property, not just a
policy setting. Access to a vault is authorized by two independent layers —
Azure RBAC (or the older vault access policy model) determines whether the
calling identity's token is allowed to call a given Key Vault data-plane
operation, and the vault's own firewall/private-endpoint rules determine
whether the network path is even allowed to reach the vault at all — both
gates must pass, mirroring the RBAC + network-boundary pattern used
throughout Azure's other data-plane services.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az webapp identity assign` | Enable system-assigned identity, or attach a user-assigned one. |
| `az identity create` | Create a standalone user-assigned identity. |
| `az keyvault create --enable-rbac-authorization` | Create a vault using RBAC (not legacy access policies). |
| `az keyvault secret set --name --value` | Store a secret. |
| `az role assignment create --role "Key Vault Secrets User"` | Grant read-only secret access to an identity. |
| `@Microsoft.KeyVault(SecretUri=...)` app setting | Auto-resolved Key Vault reference, no SDK code needed. |
| `DefaultAzureCredential` (Python/`.NET`/etc. SDK) | Uses the compute resource's managed identity automatically. |

## Exercise

1. Enable a system-assigned managed identity on a Web App from
   [Module 01](01-app-service-deep-dive.md).
2. Create a Key Vault with RBAC authorization enabled, and store a secret
   in it.
3. Grant the Web App's identity the `Key Vault Secrets User` role scoped
   to that vault.
4. Add an app setting using the `@Microsoft.KeyVault(SecretUri=...)`
   syntax, and confirm in the portal that it resolves to the actual value
   (not `#REF!`).
5. Delete the resource group when finished.
