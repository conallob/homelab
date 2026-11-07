---
tags:
  - homelab
  - learning
---
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Bare-metal Kubernetes homelab running **Talos Linux** on **Raspberry Pi Compute Module 4** (CM4) hardware. Six nodes: 1 control plane + 5 workers, all on ARM64.

See `cfg/talos/CLAUDE.md` for detailed Talos operational commands and architecture.

## Directory Structure

```
cfg/
  talos/          # Live Talos machine configs + talosctl client config + Cilium config (see cfg/talos/CLAUDE.md)
    20260919/     # Vestigial previous-generation configs and patches
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
  - `controlplane.yaml`, `worker.yaml` and `talosconfig` hold cluster secrets: they are gitignored and stored in the 1Password vault `Homelab` as `talos/<generation>/<file>`
- `bin/op-vault` — add, update and fetch gitignored sensitive files in the `Homelab` vault, e.g. `bin/op-vault get talos/20260920/talosconfig cfg/talos/talosconfig`
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

## Cilium

Cilium is the CNI, installed via Helm with kube-proxy replacement (Talos runs with `cluster.network.cni.name: none` and kube-proxy disabled). It reaches the API server via KubePrism on `localhost:7445`. LoadBalancer Services get VIPs from `192.168.7.0/24` via L2 announcements on `end0` (see `docs/decisions/002-cilium-l2-vips.md`).

```bash
# Upgrade Cilium (values drop SYS_MODULE, which Talos forbids)
helm upgrade cilium cilium/cilium --version 1.20.3 -n kube-system \
  -f cfg/talos/cilium-values.yaml

# VIP pool + L2 announcement policy
kubectl apply -f cfg/talos/cilium-lb.yaml
```

## Network Layout

| Purpose | IP / Range |
|---------|-----------|
| Lab subnet | `192.168.6.0/23` (domain: `lab.taku.ie`) |
| DHCP range | `192.168.6.10–192.168.6.254` |
| Control plane VIP | `192.168.6.250` |
| Console server / dnsmasq | `192.168.6.254` |
| Tinkerbell LoadBalancer | `192.168.6.127` |
| Cilium LoadBalancer VIPs (L2) | `192.168.7.1–192.168.7.254` |

VLAN 5 is used for the lab network on the Unifi switch.

## Provisioning Flow

Two provisioning paths exist (see `docs/decisions/000-provisioning.md` for rationale):

1. **Direct PXE (current):** dnsmasq on console server → `cfg/compute-blade.ipxe` → Talos kernel from `factory.talos.dev` → `talosctl apply-config`
2. **Tinkerbell (in progress):** Tinkerbell smee in-cluster handles DHCP/iPXE for new nodes, referencing `cfg/tinkerbell/hardware.yaml`

## EEPROM Boot Order

All CM4 and CM5 nodes use `BOOT_ORDER=0xf12` — network/PXE first (`2`), then SD/eMMC (`1`), then restart (`f`).

Key codes: `1` = SD/eMMC, `2` = network/PXE, `4` = USB, `6` = NVMe PCIe, `f` = restart. Note: `1` covers both SD card and eMMC (firmware tries eMMC when no SD card is present). `6` is NVMe, **not** eMMC.

Boot config files: `~/Documents/GitHub/usbboot/recovery/boot.conf` (CM4) and `~/Documents/GitHub/usbboot/recovery5/boot.conf` (CM5). After editing, regenerate the `.bin` with `./update-pieeprom.sh` before flashing.

## Modifying iPXE / dnsmasq

- `cfg/compute-blade.ipxe` — update the `TALOS_VERSION` variable at the top when upgrading Talos
- `cfg/dnsmasq.conf` — runs inside a podman container on the console server; restart the container after changes (see `docs/SOFTWARE.md`)
