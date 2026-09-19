---
tags:
  - homelab
  - learning
---
# Software

* [x] Kubernetes
* [x] Container Network Interface (CNI)
* [ ] Container Storage Interface (CSI)
* [ ] Ingress
* [x] ArgoCD Operator
* [ ] Secrets Operator


## Console Server

**Planned (2026-09-19):** an 8-port UART console server, using the existing
[Waveshare USB-to-8CH TTL Adaptor](https://www.raspberrypi.com/products/compute-module-4/?variant=raspberry-pi-cm4008032)
(see [HARDWARE.md](HARDWARE.md)), to work around the fact that only
`lab-node-1` (the ComputeBlade **Dev** board) has rpiboot/nRPIBOOT jumper
logic for easy reflashing — `lab-node-2` through `lab-node-6` are
ComputeBlade **Basic** boards, which makes physically reflashing all of them
much more tedious.

Workflow once built: for each Basic-board node, power on, catch it at the
GRUB menu, press `e` to edit the boot entry's `linux` line and fix
`console=ttyS0,115200` → `console=ttyAMA0,115200` (see CHANGELOG 2026-09-19
for why), Ctrl+X to boot into maintenance mode, then `talosctl apply-config`
with `machine.install.extraKernelArgs: [console=ttyAMA0,115200]` in the
pushed config. Since `apply-config` against a maintenance-mode node runs
Talos's *initial* install (not a later "upgrade cycle"), the corrected
console arg gets baked into that node's own GRUB config on disk from the
first install — no reflash needed, and no repeated manual GRUB edits on
future reboots.

<details>
  <summary>Previously</summary>

### Current (as of last revision before the 2026-09-19 8-port plan)

* [Raspberry Pi OS 64-bit Lite](https://www.raspberrypi.com/software/operating-systems/)

* [ser2net](https://github.com/cminyard/ser2net)

### Previously

* [ConsolePi](https://github.com/Pack3tL0ss/ConsolePi)

* maas.io

## Nodes

## Bare Metal Provisioning


```
podman run --rm --cap-add=NET_ADMIN,NET_RAW --net=host quay.io/poseidon/dnsmasq:v0.5.0-41-g0212fd2 \
  -d -q \
  --dhcp-range=192.168.6.1,proxy,255.255.255.0 \
  --enable-tftp --tftp-root=/var/lib/tftpboot \
  --dhcp-userclass=set:ipxe,iPXE \
  --pxe-service=tag:#ipxe,x86PC,"PXE chainload to iPXE",undionly.kpxe \
  --pxe-service=tag:ipxe,x86PC,"iPXE",http://pxe.lab.taku.ie:8080/boot.ipxe \
  --pxe-service=tag:#ipxe,X86-64_EFI,"PXE chainload to iPXE UEFI",ipxe.efi \
  --pxe-service=tag:ipxe,X86-64_EFI,"iPXE UEFI",http:///pxe.lab.taku.ie:8080/boot.ipxe \
  --log-queries \
  --log-dhcp
```

</details>

## Talos Factory Semantic IDs

* CM5 - `c2da9242b133f6f82bb686864b6d7de6a42d7599bb4e40ac07aa5b49827796a9`
* CM4 - `6b700850e84fbbaa67e3558d1d8599f336f36eb063b3a3763c97f7bc7b07a760`
  (supersedes `9c17e26911d4ad2c1899ed4dc9f1e99753ce47cbd1d154898c8e3aae3b347e4f`,
  which hangs after "Booting Talos" — see [HARDWARE.md](HARDWARE.md) and
  CHANGELOG 2026-09-19)
