# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Bare-metal Kubernetes homelab running **Talos Linux** on **Raspberry Pi Compute Module 4** (CM4) hardware. Six nodes: 1 control plane + 5 workers, all on ARM64.

See `cfg/talos/CLAUDE.md` for detailed Talos operational commands and architecture.

## Directory Structure

```
cfg/
  talos/          # Talos machine configs + talosctl client config (see cfg/talos/CLAUDE.md)
  tinkerbell/     # Tinkerbell Helm values + Hardware CRDs for in-cluster bare-metal provisioning
  dnsmasq.conf    # DHCP/TFTP server config for PXE booting (runs on console server)
  compute-blade.ipxe  # iPXE boot menu served via TFTP; fetches Talos from factory.talos.dev
docs/
  HARDWARE.md     # Node inventory: MAC addresses, IPs, physical hardware specs
  SOFTWARE.md     # Console server setup (ser2net, dnsmasq, podman)
  BAREMETAL.md    # Provisioning history: what worked, what failed, and why
  BOOTSTRAPPING.md # Step-by-step cluster bootstrap procedure
  decisions/      # ADR-style records for key architectural choices
```

## Key Tools

- `talosctl` — manage Talos nodes; set `TALOSCONFIG=cfg/talos/talosconfig`
- `kubectl` — Kubernetes cluster management
- `helm` — deploys Cilium CNI and Tinkerbell
- `rpiboot` — flash eMMC on CM4 via USB (see `docs/BOOTSTRAPPING.md`)

## Tinkerbell

Tinkerbell is deployed **inside the cluster** (not as a standalone service) via Helm. Only the `smee` component is enabled — it handles DHCP and iPXE for bare-metal nodes.

```bash
# Deploy / upgrade Tinkerbell
helm upgrade --install tinkerbell oci://ghcr.io/tinkerbell/charts/tinkerbell \
  --version <version> -f cfg/tinkerbell/values.yaml -n tinkerbell --create-namespace

# Apply Hardware CRDs after deploy
kubectl apply -f cfg/tinkerbell/hardware.yaml
```

- LoadBalancer IP: `192.168.6.127` (configured in `cfg/tinkerbell/values.yaml`)
- Hardware definitions in `cfg/tinkerbell/hardware.yaml` map MAC addresses → DHCP IPs → iPXE URLs

## Network Layout

| Purpose | IP / Range |
|---------|-----------|
| Lab subnet | `192.168.6.0/24` (domain: `lab.taku.ie`) |
| DHCP range | `192.168.6.10–192.168.6.128` |
| Control plane VIP | `192.168.6.250` |
| Console server / dnsmasq | `192.168.6.254` |
| Tinkerbell LoadBalancer | `192.168.6.127` |

VLAN 5 is used for the lab network on the Unifi switch.

## Provisioning Flow

Two provisioning paths exist (see `docs/decisions/000-provisioning.md` for rationale):

1. **Direct PXE (current):** dnsmasq on console server → `cfg/compute-blade.ipxe` → Talos kernel from `factory.talos.dev` → `talosctl apply-config`
2. **Tinkerbell (in progress):** Tinkerbell smee in-cluster handles DHCP/iPXE for new nodes, referencing `cfg/tinkerbell/hardware.yaml`

## Modifying iPXE / dnsmasq

- `cfg/compute-blade.ipxe` — update the `TALOS_VERSION` variable at the top when upgrading Talos
- `cfg/dnsmasq.conf` — runs inside a podman container on the console server; restart the container after changes (see `docs/SOFTWARE.md`)
