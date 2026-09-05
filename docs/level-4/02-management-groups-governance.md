# 02 · Management Groups & Governance at Scale

[Level 4, Module 01](01-enterprise-landing-zones.md) built the management
group tree; this module goes deeper on the mechanics of governing through
it: how policy and RBAC actually inherit, how to model exceptions without
weakening the baseline, and how to keep hundreds of subscriptions provably
compliant rather than compliant-by-hope.

## Policy inheritance and evaluation order

Policies assigned at a higher scope apply to everything below it; a
resource can be subject to policies from the tenant root, every management
group above its subscription, the subscription, and the resource group —
all simultaneously.

```bash
az policy assignment list-resources \
  --name deny-public-blob-access \
  --resource-group rg-corp-app1
```

When multiple assignments conflict (one `Audit`, one `Deny` on the same
condition), **`Deny` always wins** regardless of assignment order or
scope depth — there's no "closer scope overrides" rule for effect severity,
only for whether a policy applies at all (via exclusions).

**Gotcha:** a resource group nested three management groups deep can be
non-compliant against a policy assigned at the tenant root without anyone
at the resource group level realizing it — always check compliance with
`az policy state list` scoped to what you're accountable for, not just the
policies you personally assigned.

## Exemptions vs. exclusions

- An **exclusion** (`--not-scopes` on assignment) removes a scope from the
  assignment entirely — the policy doesn't evaluate there at all.
- A **policy exemption** lets a specific resource be non-compliant with a
  specific assignment **for a documented, time-boxed reason**, while the
  assignment still applies (and still evaluates) everywhere else.

```bash
az policy exemption create \
  --name exempt-legacy-vm-encryption \
  --policy-assignment "/subscriptions/xxxx/providers/Microsoft.Authorization/policyAssignments/require-disk-encryption" \
  --exemption-category Waiver \
  --expires-on 2026-12-31T00:00:00Z \
  --scope "/subscriptions/xxxx/resourceGroups/rg-legacy/providers/Microsoft.Compute/virtualMachines/vm-legacy-app" \
  --display-name "Legacy app pending decommission Q4"
```

**Gotcha:** exclusions are invisible in compliance reporting (the resource
just never shows up as evaluated), while exemptions show up as
"Exempt" with a reason and expiry — prefer exemptions for anything you'll
need to justify in an audit; broad exclusions accumulate into governance
blind spots nobody remembers creating.

## RBAC at management group scope

```bash
az role assignment create \
  --assignee "group-platform-engineers@example.com" \
  --role "Contributor" \
  --scope "/providers/Microsoft.Management/managementGroups/alz-landingzones"
```

**Gotcha:** granting `Owner` or `Contributor` at a management group scope
grants it to **every subscription under that scope, present and future** —
a role assigned at `alz-landingzones` automatically applies to a brand new
subscription vended into `alz-corp` a year later, with nobody explicitly
re-granting it. This is powerful for platform teams and dangerous if
assigned too broadly; always scope role assignments to the narrowest level
that satisfies the actual need, and prefer custom roles over `Owner`.

## Custom roles

```bash
az role definition create --role-definition '{
  "Name": "Landing Zone Network Operator",
  "Description": "Manage VNets and peerings, no RBAC or policy changes",
  "Actions": [
    "Microsoft.Network/virtualNetworks/*",
    "Microsoft.Network/virtualNetworks/peer/action"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/write"
  ],
  "AssignableScopes": [
    "/providers/Microsoft.Management/managementGroups/alz-landingzones"
  ]
}'
```

`NotActions` **subtracts** from `Actions` for the purposes of what the role
grants, but does **not** work as a deny — if the same identity has a
different role assignment granting `Microsoft.Authorization/*/write`
through some other path, that grant still applies. `NotActions` only
narrows this specific role definition, it isn't a security boundary on its
own.

## Compliance reporting at scale

```bash
az policy state summarize --management-group alz

az policy state list \
  --management-group alz \
  --filter "ComplianceState eq 'NonCompliant'" \
  --query "[].{resource:resourceId, policy:policyDefinitionName}" \
  -o table
```

**Gotcha:** `az policy state summarize` reflects the **last evaluation
cycle** (policy evaluates on a roughly 24-hour cycle plus on resource
create/update events) — a resource fixed five minutes ago can still show
`NonCompliant` until the next cycle runs; trigger an on-demand scan with
`az policy state trigger-scan` when you need current results for a change
you just made, rather than assuming the dashboard is live.

## Governance mechanisms compared

| Mechanism | Scope | Purpose | Auditable? |
|---|---|---|---|
| Policy assignment | Any scope | Enforce/audit a rule | Yes — compliance state |
| Policy exemption | Specific resource | Document a justified exception | Yes — visible as Exempt |
| Assignment exclusion (`--not-scopes`) | Sub-scope | Remove scope from evaluation | No — invisible in reports |
| RBAC role assignment | Any scope | Grant access to act | Yes — via `az role assignment list` |
| Custom role | Any scope | Narrow a grant beyond built-ins | Yes, but `NotActions` isn't a deny |

## How It Actually Works

**Management groups** exist purely as a scoping node in ARM's authorization
hierarchy — creating one doesn't provision any resource, it inserts an
entry into your tenant's management-group tree that RBAC role assignments
and Azure Policy assignments can target, and ARM's authorization check
(from Level 1's capstone) walks *up* this tree from a resource through its
resource group, subscription, and every management group above it when
evaluating whether a caller's role assignment or a deny-effect policy
applies — this upward walk, not any per-resource configuration, is the
actual mechanism behind policy/RBAC inheritance. **Azure Policy's
`DeployIfNotExists` and `Modify` effects**, when assigned at a management
group, are evaluated per-resource at the resource's own scope during the
same compliance-scan cycle from Level 3's IaC module, but the *remediation*
deployment they trigger runs under a managed identity created for the
policy assignment at the management-group scope — which is why that
identity needs RBAC roles granted explicitly at each subscription it must
remediate resources in, even though the policy assignment itself lives
several levels above.

**Blueprints/deployment stacks vs. Policy vs. RBAC** are three different
enforcement points in that same request pipeline: RBAC gates *who* can call
an operation, Policy gates *what* a request's properties are allowed to be
(evaluated inline, before the resource provider processes it), and a
deployment stack (or Blueprint) additionally tracks a *set* of resources as
a unit so it can apply lifecycle protections (deny-delete on managed
resources) and detect drift against the originally deployed template — the
three mechanisms compose because they intercept the request lifecycle at
genuinely different stages, not because one supersedes another.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az policy assignment list-resources` | See what a specific assignment applies to. |
| `az policy exemption create --expires-on` | Grant a documented, time-boxed compliance waiver. |
| `az role assignment create --scope <mg-id>` | Grant RBAC inherited by everything under a management group. |
| `az role definition create` | Define a custom role narrower than built-ins. |
| `az policy state summarize --management-group` | Get an aggregate compliance view. |
| `az policy state trigger-scan` | Force an on-demand compliance re-evaluation. |

## Exercise

1. Assign a policy at a management group scope and confirm (via
   `list-resources`) it applies to a subscription two levels below.
2. Create a policy exemption with an expiry date for one resource, and
   contrast how it appears in `az policy state list` versus how an
   exclusion would (not appear at all).
3. Create a custom role with `NotActions` removing one permission, assign
   it at a management group, and verify what it actually restricts (and
   what it doesn't, if another role grants the same action).
4. Run `az policy state trigger-scan` after fixing a non-compliant resource
   and confirm the compliance state updates without waiting for the next
   cycle.
5. Clean up any test policy assignments, exemptions, and role assignments
   when finished.
