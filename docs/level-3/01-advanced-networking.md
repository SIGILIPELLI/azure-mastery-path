# 01 · Advanced Networking (VPN Gateway, ExpressRoute)

[Level 2, Module 03](../level-2/03-networking-deep-dive.md) covered Load
Balancer, Application Gateway, and private endpoints — all ways to route
traffic *within* Azure or from the public internet into a VNet. This module
covers connecting your **on-premises network** to Azure: site-to-site VPN
over the public internet, and ExpressRoute private circuits that never touch
it, plus the hub-and-spoke topology most enterprises land on once they have
more than one VNet.

## Site-to-site VPN Gateway

A **VPN Gateway** terminates an IPsec/IKE tunnel between Azure and an
on-premises VPN device (or another cloud). It needs its own dedicated
`GatewaySubnet` (minimum `/27`, no other resources allowed in it):

```bash
az group create --name rg-hub-net --location eastus

az network vnet subnet create \
  --resource-group rg-hub-net \
  --vnet-name vnet-hub \
  --name GatewaySubnet \
  --address-prefix 10.0.255.0/27

az network public-ip create \
  --resource-group rg-hub-net \
  --name pip-vpngw \
  --sku Standard \
  --allocation-method Static

az network vnet-gateway create \
  --resource-group rg-hub-net \
  --name vpngw-hub \
  --vnet vnet-hub \
  --public-ip-address pip-vpngw \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2 \
  --no-wait
```

Gateway provisioning takes **30-45 minutes** — `--no-wait` returns
immediately and you poll with `az network vnet-gateway show`. Once it's up,
define the on-prem side as a **local network gateway** and connect the two:

```bash
az network local-gateway create \
  --resource-group rg-hub-net \
  --name lgw-onprem \
  --gateway-ip-address 203.0.113.5 \
  --local-address-prefixes 192.168.0.0/16

az network vpn-connection create \
  --resource-group rg-hub-net \
  --name cn-hub-to-onprem \
  --vnet-gateway1 vpngw-hub \
  --local-gateway2 lgw-onprem \
  --shared-key "ReplaceWithARealSecret1!" \
  --enable-bgp false
```

```text
$ az network vpn-connection show -g rg-hub-net -n cn-hub-to-onprem --query connectionStatus -o tsv
Connected
```

**Gotcha:** `VpnGw1`/`Basic` SKUs cap throughput and don't support
zone-redundancy; production designs generally start at `VpnGw2` (or the
`AZ` variants for zone redundancy) since resizing later means a gateway
rebuild with a maintenance window. The pre-shared key must match exactly on
both ends — a mismatch fails silently with the tunnel state stuck at
`Connecting`, never surfacing as an explicit auth error.

## ExpressRoute

**ExpressRoute** is a private, dedicated circuit from a connectivity
provider's edge into Microsoft's network — traffic never traverses the
public internet, giving predictable latency and higher throughput than any
VPN, at a materially higher recurring cost that scales with the chosen
bandwidth tier.

```bash
az network express-route create \
  --resource-group rg-hub-net \
  --name er-circuit-primary \
  --provider "Equinix" \
  --peering-location "Washington DC" \
  --bandwidth 1000 \
  --sku-family MeteredData \
  --sku-tier Standard
```

The circuit only becomes billable/active once the connectivity provider
provisions their side and you confirm — check `serviceProviderProvisioningState`:

```bash
az network express-route show \
  --resource-group rg-hub-net \
  --name er-circuit-primary \
  --query serviceProviderProvisioningState -o tsv
```

After the provider marks it `Provisioned`, attach an ExpressRoute gateway
(a different SKU family than the VPN gateway, but can share the same
`GatewaySubnet`) and create the connection:

```bash
az network vnet-gateway create \
  --resource-group rg-hub-net \
  --name ergw-hub \
  --vnet vnet-hub \
  --gateway-type ExpressRoute \
  --sku Standard

az network vpn-connection create \
  --resource-group rg-hub-net \
  --name cn-hub-to-er \
  --vnet-gateway1 ergw-hub \
  --express-route-circuit2 er-circuit-primary
```

**Gotcha:** ExpressRoute alone does not encrypt traffic — it's private, not
encrypted by default. Regulated workloads typically add **IPsec over
ExpressRoute** (a VPN tunnel riding on the private circuit) or rely on
application-layer TLS. Also, an ExpressRoute circuit without a configured
**VPN Gateway as a backup path** has no automatic failover if the circuit
drops — most production designs pair ExpressRoute (primary) with a
site-to-site VPN (backup, lower priority via BGP weight).

## Hub-and-spoke topology

Rather than connecting every VNet to on-premises individually, enterprises
centralize shared services (firewall, VPN/ExpressRoute gateway, DNS) in a
**hub** VNet and peer application **spoke** VNets to it:

```bash
az network vnet create \
  --resource-group rg-hub-net \
  --name vnet-spoke-app1 \
  --address-prefix 10.1.0.0/16 \
  --subnet-name snet-app1 \
  --subnet-prefix 10.1.0.0/24

az network vnet peering create \
  --resource-group rg-hub-net \
  --name hub-to-spoke1 \
  --vnet-name vnet-hub \
  --remote-vnet vnet-spoke-app1 \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --allow-gateway-transit

az network vnet peering create \
  --resource-group rg-hub-net \
  --name spoke1-to-hub \
  --vnet-name vnet-spoke-app1 \
  --remote-vnet vnet-hub \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --use-remote-gateways
```

**Gotcha:** VNet peering is **not transitive** — spoke1 peered to hub and
spoke2 peered to hub does *not* let spoke1 talk to spoke2 directly; traffic
must route through a network appliance (Azure Firewall or an NVA) in the
hub, or you configure user-defined routes (UDRs) forcing spoke-to-spoke
traffic through it. Also `--allow-gateway-transit` on the hub side and
`--use-remote-gateways` on the spoke side must both be set, or spokes can't
reach on-premises through the hub's gateway at all — a very common
first-peering mistake that manifests as spokes losing on-prem connectivity
even though the peering itself shows `Connected`.

## Cheat sheet

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

## Exercise

1. Create a hub VNet with a `GatewaySubnet` and provision a `VpnGw2`
   gateway (use `--no-wait` and poll `az network vnet-gateway show`).
2. Define a local network gateway representing a fake on-premises range and
   create a site-to-site connection; check `connectionStatus`.
3. Create a spoke VNet and peer it to the hub with gateway transit enabled
   in both directions.
4. Explain in your own words why two spokes peered to the same hub still
   can't reach each other without an NVA or Azure Firewall in the path.
5. Delete the resource group when finished.
