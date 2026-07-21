# 05 · Virtual Networks Basics

A **Virtual Network (VNet)** is your private, isolated slice of Azure's
network — resources you put inside it can talk to each other privately, and
you control what comes in or goes out. Where [Module 03](03-virtual-machines.md)
let `az vm create` build a VNet for you as a convenience default, this
module builds one explicitly so you understand the pieces: VNets, subnets,
NSGs, and route tables.

## VNets and address spaces

A VNet is defined by an **address space** in CIDR notation — a private IP
range that exists only within Azure's network fabric (these don't need to
be globally unique, only unique within networks you plan to connect
together).

```bash
az group create --name rg-vnet-demo --location eastus

az network vnet create \
  --resource-group rg-vnet-demo \
  --name vnet-demo \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-web \
  --subnet-prefix 10.0.1.0/24
```

`10.0.0.0/16` gives the VNet roughly 65,000 addresses; the `/24` subnet
carved out of it gives that subnet 256 addresses (251 usable — Azure
reserves 5 per subnet for its own networking functions).

The private ranges reserved for this kind of use (RFC 1918) are
`10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16` — pick non-overlapping
blocks from these if you'll ever connect multiple VNets together (via
peering or a VPN gateway, covered in [Level 3](../level-3/01-advanced-networking.md)).

## Subnets

A VNet is subdivided into **subnets** — smaller ranges used to segment
resources by role (a "web" subnet, a "database" subnet, etc.) so you can
apply different security rules and route tables to each.

```bash
# Add another subnet to the same VNet
az network vnet subnet create \
  --resource-group rg-vnet-demo \
  --vnet-name vnet-demo \
  --name subnet-data \
  --address-prefix 10.0.2.0/24

# List subnets in a VNet
az network vnet subnet list \
  --resource-group rg-vnet-demo \
  --vnet-name vnet-demo \
  --output table
```

Some Azure services (Azure SQL, Storage, Key Vault, ...) support **service
endpoints** or **private endpoints** on a subnet, letting resources reach
those services over the private Azure backbone instead of the public
internet — covered in [Level 2](../level-2/03-networking-deep-dive.md).

## Network Security Groups on subnets

[Module 03](03-virtual-machines.md) attached an NSG to a VM's network
interface directly; NSGs can also attach to a **subnet**, applying to every
resource inside it at once — the more common pattern for anything beyond a
single VM.

```bash
az network nsg create --resource-group rg-vnet-demo --name nsg-web

az network nsg rule create \
  --resource-group rg-vnet-demo \
  --nsg-name nsg-web \
  --name allow-https \
  --priority 200 \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow

az network nsg rule create \
  --resource-group rg-vnet-demo \
  --nsg-name nsg-web \
  --name deny-all-inbound \
  --priority 4096 \
  --destination-port-ranges "*" \
  --protocol "*" \
  --access Deny

# Associate the NSG with the subnet
az network vnet subnet update \
  --resource-group rg-vnet-demo \
  --vnet-name vnet-demo \
  --name subnet-web \
  --network-security-group nsg-web
```

When both a subnet NSG and a NIC NSG are in play, **both** must allow a
packet for it to pass — either one denying it blocks the traffic. Azure
already has a lower-priority implicit "deny all" and "allow VNet-internal
traffic" set of default rules; the explicit `deny-all-inbound` rule above
just makes the intent visible in the rule list.

## Route tables

By default, Azure automatically routes traffic between subnets in the same
VNet, out to the internet, and to a few Azure-internal ranges — these are
the **system routes**. A **route table** lets you override that, e.g. to
force all outbound traffic through a firewall appliance or VPN gateway
instead of straight to the internet.

```bash
az network route-table create \
  --resource-group rg-vnet-demo \
  --name rt-web

az network route-table route create \
  --resource-group rg-vnet-demo \
  --route-table-name rt-web \
  --name force-tunnel \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.2.4

az network vnet subnet update \
  --resource-group rg-vnet-demo \
  --vnet-name vnet-demo \
  --name subnet-web \
  --route-table rt-web
```

This route says "anything leaving `subnet-web` destined anywhere
(`0.0.0.0/0`) goes to the virtual appliance at `10.0.2.4` first" — the
building block behind hub-and-spoke topologies with a central firewall,
covered further in [Level 3, Module 01](../level-3/01-advanced-networking.md).

## Putting it together: a VM inside your VNet

```bash
az vm create \
  --resource-group rg-vnet-demo \
  --name vm-web \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name vnet-demo \
  --subnet subnet-web \
  --nsg ""
```

Passing `--nsg ""` tells `az vm create` *not* to generate its own NIC-level
NSG, since the subnet-level `nsg-web` already governs traffic — avoiding
duplicate, possibly conflicting rule sets.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az network vnet create --address-prefix --subnet-name --subnet-prefix` | Create a VNet with an initial subnet. |
| `az network vnet subnet create` | Add another subnet to an existing VNet. |
| `az network vnet subnet list -o table` | List a VNet's subnets. |
| `az network nsg create` / `nsg rule create` | Create an NSG and its rules. |
| `az network vnet subnet update --network-security-group` | Attach an NSG to a subnet. |
| `az network route-table create` / `route create` | Create a route table and a custom route. |
| `az network vnet subnet update --route-table` | Attach a route table to a subnet. |

## Exercise

1. Create a VNet `vnet-lab` with address space `10.1.0.0/16` and two
   subnets: `subnet-app` (`10.1.1.0/24`) and `subnet-db` (`10.1.2.0/24`).
2. Create an NSG that allows inbound 22 and 443 and denies everything else,
   and attach it to `subnet-app`.
3. List the effective rules with
   `az network nsg rule list --nsg-name <name> --output table` and confirm
   the priorities and actions look right.
4. Delete the resource group when finished.
