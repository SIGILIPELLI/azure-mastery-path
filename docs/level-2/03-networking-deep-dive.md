# 03 · Networking Deep Dive (Load Balancer, App Gateway)

[Level 1, Module 05](../level-1/05-virtual-networks.md) covered VNets,
subnets, NSGs, and route tables. This module covers the layer that sits in
front of your apps: **Azure Load Balancer** (Layer 4, TCP/UDP), **Application
Gateway** (Layer 7, HTTP/HTTPS, with an optional Web Application Firewall),
and **private endpoints**, which pull PaaS services onto your VNet instead
of the public internet.

## Load Balancer (Layer 4)

An **Azure Load Balancer** distributes TCP/UDP traffic across a pool of
backend instances (VMs, VM Scale Sets) based on a 5-tuple hash — it doesn't
look inside the packet, so it can't route on URL path or hostname.

```bash
az group create --name rg-net-deep --location eastus

az network lb create \
  --resource-group rg-net-deep \
  --name lb-demo \
  --sku Standard \
  --public-ip-address pip-lb-demo \
  --frontend-ip-name frontend \
  --backend-pool-name backend-pool

az network lb probe create \
  --resource-group rg-net-deep \
  --lb-name lb-demo \
  --name http-probe \
  --protocol Tcp \
  --port 80

az network lb rule create \
  --resource-group rg-net-deep \
  --lb-name lb-demo \
  --name http-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name frontend \
  --backend-pool-name backend-pool \
  --probe-name http-probe
```

The **health probe** decides which pool members receive traffic — a VM that
fails its probe (say, its web server crashed but the VM is still running)
is silently pulled from rotation until it starts passing again.

**Gotcha:** `Basic` SKU load balancers are being retired (no new SLA, no
new features) — always create `Standard` SKU, which also requires backend
VMs to have `Standard` SKU public IPs if they have one directly, and denies
all inbound traffic by default unless an NSG explicitly allows it (Basic
implicitly allowed it).

## Application Gateway (Layer 7)

An **Application Gateway** understands HTTP: it can route based on URL path
or hostname, terminate TLS, and — with the WAF SKU — inspect and block
requests matching OWASP rule sets before they reach your backend.

```bash
az network application-gateway create \
  --resource-group rg-net-deep \
  --name appgw-demo \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name vnet-appgw \
  --subnet subnet-appgw \
  --public-ip-address pip-appgw \
  --servers 10.0.1.4 10.0.1.5
```

App Gateway v2 requires its own **dedicated subnet** (no other resource
types allowed in it) and needs at least a `/24` if you want headroom for
autoscaling instances. Path-based routing splits traffic by URL prefix:

```bash
az network application-gateway url-path-map create \
  --resource-group rg-net-deep \
  --gateway-name appgw-demo \
  --name path-map \
  --paths /api/* \
  --address-pool api-pool \
  --default-address-pool web-pool \
  --default-http-settings appGatewayBackendHttpSettings \
  --http-settings appGatewayBackendHttpSettings \
  --http-listener appGatewayHttpListener
```

Requests to `/api/*` go to `api-pool`; everything else falls through to the
`web-pool` default — one gateway, one public IP, fronting multiple backend
services.

**Gotcha:** enabling WAF (`WAF_v2` SKU) in **Prevention** mode blocks
matched requests outright; **Detection** mode only logs them. Always launch
a new WAF in Detection mode first, review the logs for false positives
against real traffic for a few days, then flip to Prevention — going
straight to Prevention on day one is a common cause of "why is my app
suddenly returning 403s" incidents.

## Load Balancer vs. Application Gateway vs. Front Door

| | Load Balancer | Application Gateway | Front Door |
|---|---|---|---|
| **Layer** | 4 (TCP/UDP) | 7 (HTTP/HTTPS) | 7 (HTTP/HTTPS), global |
| **Scope** | Regional | Regional | Global (anycast) |
| **Routing** | 5-tuple hash | Path/host-based | Path/host-based |
| **WAF** | No | Yes (optional) | Yes (optional) |
| **Use case** | Raw TCP, non-HTTP, lowest latency | Regional HTTP with WAF/path routing | Multi-region HTTP, CDN-style edge |

## Private endpoints

A **private endpoint** attaches a private IP (from your own VNet) to a PaaS
resource — a storage account, SQL server, Key Vault — so traffic reaches it
over the Azure backbone, never the public internet, and the public
endpoint can be locked down entirely.

```bash
az network private-endpoint create \
  --resource-group rg-net-deep \
  --name pe-storage \
  --vnet-name vnet-appgw \
  --subnet subnet-privatelink \
  --private-connection-resource-id $(az storage account show --name ststoragedeep --resource-group rg-storage-deep --query id -o tsv) \
  --group-id blob \
  --connection-name conn-storage
```

Creating the endpoint doesn't automatically redirect DNS — without a
**Private DNS Zone** linked to the VNet resolving
`<account>.blob.core.windows.net` to the private IP, clients still resolve
to the public IP and either still work over the internet or fail if you've
also disabled public access. Wire that up:

```bash
az network private-dns zone create \
  --resource-group rg-net-deep \
  --name privatelink.blob.core.windows.net

az network private-dns link vnet create \
  --resource-group rg-net-deep \
  --zone-name privatelink.blob.core.windows.net \
  --name dns-link-appgw \
  --virtual-network vnet-appgw \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group rg-net-deep \
  --endpoint-name pe-storage \
  --name zone-group \
  --private-dns-zone privatelink.blob.core.windows.net \
  --zone-name blob
```

**Gotcha:** this is the single most common thing people forget with private
endpoints — the endpoint exists, connects fine from inside the VNet by IP,
but any client resolving the hostname normally still gets the public IP
until the private DNS zone is created, linked, and populated. Test with
`nslookup <account>.blob.core.windows.net` from inside the VNet and confirm
it resolves to the `10.x` private IP, not a public one.

## How It Actually Works

**User-Defined Routes (UDRs)** override Azure's default system routes by
being programmed into the same VFP layer described in Module 5 — a route
table attached to a subnet inserts entries into every VM's effective route
table there, so traffic destined for an address you've routed through an
NVA (say, `0.0.0.0/0 → 10.0.1.4`) is forwarded to that next hop's private IP
at the SDN layer before the packet leaves the sending host, with no
involvement from the NVA's own OS routing until the packet actually arrives
there. **Azure Load Balancer** (Layer 4) works by programming the same VFP
rules to rewrite the destination IP/port of an inbound packet from the
load balancer's frontend IP to a chosen backend instance's private IP — for
each new flow, it selects a backend using a 5-tuple hash and then
Direct-Server-Return-style forwards subsequent packets in that same flow
without re-hashing, all done at the host networking layer without the
packet ever passing through a separate load-balancing appliance VM.

**Private Endpoints** work by injecting a NIC with a private IP from your
VNet directly into the PaaS service's own network path via **Azure Private
Link**: DNS for the PaaS resource (e.g. a storage account's
`.blob.core.windows.net`) is overridden by a **Private DNS Zone** linked to
your VNet so the name now resolves to that private IP instead of the
public one, and traffic to it never leaves the Microsoft backbone or
traverses the public internet — this is a genuinely different data path
from a Service Endpoint, which instead just tags outbound traffic on your
VNet's existing route so the resource provider's firewall can identify and
allow it, while the traffic still egresses to the service's public IP
range. That distinction — new private IP + DNS override vs. tagged
traffic on the existing public path — is the actual mechanism behind why
Private Endpoints work across peered VNets and on-prem via ExpressRoute/VPN
while Service Endpoints do not.

## Cheat sheet

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

## Exercise

1. Create a Standard SKU load balancer with a health probe on port 80 and a
   forwarding rule, pointed at two test VMs in a backend pool.
2. Create a WAF_v2 Application Gateway in **Detection** mode with a
   path-based rule sending `/api/*` to one pool and everything else to
   another.
3. Create a private endpoint for a storage account, then create and link a
   private DNS zone so the account's hostname resolves to the private IP
   from inside the VNet.
4. Confirm with `nslookup` (from a VM inside the VNet) that the storage
   account hostname resolves privately, not publicly.
5. Delete the resource group when finished.
