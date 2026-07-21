# 02 · Azure AD & RBAC / Security Basics

Before you create anything else, it's worth understanding *who* is allowed to
do *what* in your subscription. Azure's identity service is **Microsoft Entra
ID** (formerly **Azure Active Directory / Azure AD** — you'll see both names
in docs, portal menus, and CLI output, since the rename is still rolling
out). Access to resources is controlled by **Role-Based Access Control
(RBAC)**, which assigns *roles* to *identities* over a *scope*.

## Microsoft Entra ID (Azure AD) basics

Every Azure subscription is linked to one Entra ID **tenant** — the directory
that holds users, groups, and app registrations. When you signed up for a
free account, Azure created a tenant for you automatically.

```bash
# Show the Entra ID tenant behind your current subscription
az account show --query tenantId --output tsv

# List users in the tenant (requires Entra ID read permission,
# which your own account has by default as the tenant creator)
az ad user list --output table --query "[].{Name:displayName, UPN:userPrincipalName}"

# Show your own signed-in identity
az ad signed-in-user show --output table
```

Identities you'll assign roles to fall into three buckets:

| Identity type | Example |
|---------------|---------|
| **User** | A person signing in with a Microsoft account or work/school account. |
| **Group** | A collection of users, so you assign a role once to the group instead of per-person. |
| **Service principal / Managed Identity** | An application or Azure resource acting non-interactively (covered in depth in [Level 2, Module 06](../level-2/06-managed-identities-key-vault.md)). |

## RBAC: roles, scopes, and assignments

RBAC access is defined by three things: **who** (the identity), **what**
(a role, i.e. a set of permitted actions), and **where** (the scope — how
much of your resource hierarchy the permission applies to).

Scope is hierarchical, narrowest to broadest:

```
Management Group → Subscription → Resource Group → Resource
```

A role assigned at a broader scope is inherited by everything beneath it —
e.g. a role granted at the subscription level applies to every resource
group and resource inside that subscription.

### Built-in roles you'll use constantly

| Role | Grants |
|------|--------|
| **Owner** | Full access, including managing access for others. |
| **Contributor** | Full access to manage resources, but *cannot* grant access to others. |
| **Reader** | View-only access to everything in scope. |
| **User Access Administrator** | Manage user access to resources, without managing the resources themselves. |

### Assigning a role

```bash
# Get your own user's object ID (or a colleague's)
az ad signed-in-user show --query id --output tsv

# Assign the "Reader" role to a user, scoped to one resource group
az role assignment create \
  --assignee "<user-object-id-or-email>" \
  --role "Reader" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-learn"

# List role assignments at a scope
az role assignment list \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-learn" \
  --output table

# Remove a role assignment
az role assignment delete \
  --assignee "<user-object-id-or-email>" \
  --role "Reader" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-learn"
```

Get your subscription ID any time with `az account show --query id --output tsv`.

## The principle of least privilege

Grant the **narrowest role at the narrowest scope** that lets someone do
their job — not "Owner on the whole subscription" out of convenience. In
practice:

- Prefer a resource-group scope over subscription scope whenever the work is
  confined to one project.
- Prefer **Contributor** over **Owner** unless the person genuinely needs to
  manage other people's access.
- Use **custom roles** (below) when a built-in role grants more than is
  needed — e.g. a role that can restart VMs but not delete them.
- Review role assignments periodically; `az role assignment list --all
  --output table` from the subscription scope shows everything at once.

## Custom roles (a peek ahead)

When built-in roles don't fit, you can define a custom role as JSON:

```json
{
  "Name": "VM Operator",
  "Description": "Can start/restart VMs but not delete them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/<sub-id>"]
}
```

```bash
az role definition create --role-definition vm-operator-role.json
```

You won't need custom roles often at this level, but knowing they exist
explains *why* RBAC has "Actions" strings like
`Microsoft.Compute/virtualMachines/start/action` under the hood — every
built-in role is just a named bundle of these.

## Multi-factor authentication (MFA)

Enable MFA on your own account from **Entra ID → Security → MFA** in the
Portal (this is a portal-only setting for personal accounts; organizational
tenants manage it via Conditional Access policies). This single setting
blocks the overwhelming majority of account-takeover attempts and costs you
one extra tap on your phone per sign-in — turn it on before you go any
further in this course.

## Cheat sheet

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

## Exercise

1. Run `az account show --query tenantId --output tsv` and note your tenant
   ID.
2. Create a resource group `rg-rbac-demo`.
3. Assign yourself the **Reader** role at that resource group's scope (even
   though you're already Owner via the subscription — this is just for
   practice with the command), then confirm it with
   `az role assignment list --scope <scope> --output table`.
4. Remove the assignment with `az role assignment delete`, then delete the
   resource group.
