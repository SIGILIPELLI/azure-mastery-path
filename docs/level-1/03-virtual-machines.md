# 03 · Virtual Machines

Azure **Virtual Machines (VMs)** are the IaaS building block — a full
Linux or Windows operating system running on Microsoft's hardware, billed
by the second while it's running. This module creates a Linux VM, connects
to it over SSH, and covers the networking rules (NSGs) that control what
traffic can reach it.

## VM sizes

Every VM has a **size** (also called a SKU) that determines its vCPUs, RAM,
and disk/network performance tier. Sizes are grouped into families by
workload:

| Family | Example size | Good for |
|---|---|---|
| **B-series** (burstable) | `Standard_B1s` | Free-tier eligible, low steady-state CPU with burst credit — ideal for learning/dev. |
| **D-series** (general purpose) | `Standard_D2s_v5` | Balanced CPU/memory for typical web apps and services. |
| **F-series** (compute optimized) | `Standard_F2s_v2` | CPU-heavy workloads like batch processing. |
| **E-series** (memory optimized) | `Standard_E2s_v5` | In-memory caches, larger databases. |

`Standard_B1s` (1 vCPU, 1 GiB RAM) is covered by the Always-Free tier's 750
hours/month of B1s Linux or Windows VM usage — use it for every exercise in
this course.

```bash
# List available VM sizes in a region
az vm list-sizes --location eastus --output table
```

## Create a Linux VM

```bash
az group create --name rg-vm-demo --location eastus

az vm create \
  --resource-group rg-vm-demo \
  --name vm-demo \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys
```

- `--image Ubuntu2204` picks an Ubuntu 22.04 LTS marketplace image (run
  `az vm image list --output table --all` to browse others).
- `--generate-ssh-keys` creates (or reuses) an SSH key pair at
  `~/.ssh/id_rsa` and installs the public key on the VM — no password
  authentication needed.
- This single command also creates, for free, a virtual network, subnet,
  public IP, and network security group as defaults, so a bare VM is
  reachable immediately. In [Module 05](05-virtual-networks.md) you'll
  create these explicitly instead of relying on the defaults.

The command prints JSON on completion including the VM's public IP:

```json
{
  "publicIpAddress": "20.xx.xx.xx",
  "resourceGroup": "rg-vm-demo",
  ...
}
```

## Connect over SSH

```bash
ssh azureuser@20.xx.xx.xx

# Or let the CLI look the IP up for you:
az vm show --resource-group rg-vm-demo --name vm-demo \
  --show-details --query publicIps --output tsv
```

Windows VMs use RDP (Remote Desktop Protocol) instead — port 3389 rather
than SSH's 22 — connected to via the Remote Desktop client using the
admin username/password you set at creation (`--admin-password`, since RDP
doesn't use SSH keys).

## Network Security Groups (NSGs)

An **NSG** is a stateful firewall of allow/deny rules, evaluated by
priority, attached to a subnet and/or a VM's network interface. The default
VM creation above attaches an NSG that allows inbound SSH (port 22) from
anywhere and denies everything else inbound by default.

```bash
# Show the NSG created for your VM
az network nsg list --resource-group rg-vm-demo --output table

# List its rules
az network nsg rule list \
  --resource-group rg-vm-demo \
  --nsg-name vm-demoNSG \
  --output table

# Open port 80 for a web server you'll install
az network nsg rule create \
  --resource-group rg-vm-demo \
  --nsg-name vm-demoNSG \
  --name allow-http \
  --priority 320 \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow
```

Rules are evaluated in ascending priority order (lower number = higher
precedence) and the first match wins — leave gaps between priorities (100,
200, 300...) so you can insert new rules later without renumbering.

!!! warning "Restrict SSH/RDP in anything beyond a lab"
    The default SSH rule allows **any** source IP. For real use, restrict
    `--source-address-prefixes` to your own IP (`curl ifconfig.me` to find
    it) instead of leaving port 22/3389 open to the entire internet, which
    is a constant scanning target for automated attacks.

```bash
# Tighten the default SSH rule to just your IP
az network nsg rule update \
  --resource-group rg-vm-demo \
  --nsg-name vm-demoNSG \
  --name default-allow-ssh \
  --source-address-prefixes "$(curl -s ifconfig.me)/32"
```

## Managing the VM

```bash
# Stop (deallocate) the VM -- stops billing for compute, keeps the disk
az vm deallocate --resource-group rg-vm-demo --name vm-demo

# Start it again
az vm start --resource-group rg-vm-demo --name vm-demo

# Restart
az vm restart --resource-group rg-vm-demo --name vm-demo

# Resize (must be stopped first if the new size isn't available on the
# current hardware cluster)
az vm resize --resource-group rg-vm-demo --name vm-demo --size Standard_B2s

# See running state
az vm get-instance-view --resource-group rg-vm-demo --name vm-demo \
  --query instanceView.statuses --output table
```

`az vm stop` only shuts down the OS but Azure keeps billing the compute
allocation; use `az vm deallocate` to actually release the hardware
reservation and stop compute charges (the disk still incurs its own small
storage charge either way).

## Cleanup

```bash
az group delete --name rg-vm-demo --yes --no-wait
```

Deleting the resource group removes the VM, its disk, public IP, NIC, and
NSG together — the fastest way to make sure nothing keeps billing.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az vm list-sizes --location <region>` | List available VM sizes in a region. |
| `az vm create -g -n --image --size --admin-username --generate-ssh-keys` | Create a Linux VM with SSH key auth. |
| `az vm show --show-details --query publicIps -o tsv` | Get a VM's public IP. |
| `az network nsg rule list -g --nsg-name` | List NSG rules. |
| `az network nsg rule create ... --priority --destination-port-ranges --access` | Add an NSG rule. |
| `az vm deallocate -g -n` | Stop the VM and release compute billing. |
| `az vm start` / `az vm restart` | Start / restart a VM. |
| `az vm resize -g -n --size` | Change a VM's size. |

## Exercise

1. Create a resource group `rg-vm-lab` and a `Standard_B1s` Ubuntu VM in it.
2. SSH into the VM and run `sudo apt update && sudo apt install -y nginx`
   to install a web server.
3. Add an NSG rule opening port 80, then visit `http://<public-ip>/` in a
   browser to see the nginx welcome page.
4. Tighten the SSH rule to your own IP address only.
5. `az vm deallocate` the VM, confirm its status shows "deallocated" with
   `az vm get-instance-view`, then delete the resource group.
