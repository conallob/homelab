---
tags:
  - hardware
  - homelab
  - kubernetes
  - k8s
  - learning
  - rpi
  - raspberry-pi
  - talos
---
# Changelog


## 2026-09-12

* Shelved CM5/Pi5 support in `talos-rpi-builder` after an unresolved GRUB→kernel
  silent reboot loop persisted across multiple fixes — see
  [decisions/001-cm5-shelved-image-factory.md](decisions/001-cm5-shelved-image-factory.md)
* Pivoted CM4 provisioning to Talos's official Image Factory
  (factory.talos.dev) instead of the custom build pipeline
* Bumped `talos-rpi-builder` to Talos v1.14.0 (three merged PRs fixing real
  upstream breakage: `hack/modules-arm64.txt` regen, a new strict `depmod`
  check requiring `/boot/System.map`, and two Pi-irrelevant modules
  (`hibmc-drm.ko`, `panfrost.ko`) that became fatal under the stricter check)
* Flashed the resulting v1.14.0 image to a CM4 (eMMC and NVMe) and hit a UART
  boot failure — RPi boot ROM reported `start4.elf`/`config.txt` "not found"
  on the eMMC's EFI partition despite the files being verified present when
  mounted on macOS; root cause not yet confirmed, see BAREMETAL.md
* Designed (not yet applied) a fleet-wide EEPROM `boot.conf` for the planned
  7-node cluster: `BOOT_ORDER=0xf261`, with node role expressed by which
  device carries a bootable Talos partition rather than by boot order —
  workers boot Talos from eMMC/SD (NVMe left blank for Longhorn/CSI storage),
  control-plane boots Talos from NVMe (eMMC left blank)

## 2026-04-28

* Reflashed talos onto every CM5 and CM4
* CM5 nodes are booting fine, CM4 nodes however appear to be stuck in a reboot loop
* Followed https://docs.siderolabs.com/talos/v1.13/getting-started/getting-started
  to create a new talos kubnernetes cluster

## 2026-04-27

* Forked https://github.com/talos-rpi5/talos-builder to create a Talos v1.12.x
  install image for the RPi 5
* Eventually got https://github.com/conallob/talos-builder/ building a v1.12.7
  image, which required a few iterations after running into boot issues
* Discovered Talos v1.13.0 was released today, and has official support for the RPi 5
* Nope, maybe not. v1.13.0 doesn't boot on the CM5s, 
* Let Claude run the `talosctl` steps to bootstrap the cluster. It went crazy and
  I had to reflash every CM4 and CM5 by hand


## 2025-11-23

* 3x CM5 with 16GB RAM + 64GB eMMC Flashed with Talos v.11.3

## 2025-11-06

* Upgraded all 7 nodes to Talos v1.11.5

## 2025-10-07

* Cilium CNI installed
* ArgoCD installed per https://argo-cd.readthedocs.io/en/stable/#quick-start
* Longhorn CSI installed per https://longhorn.io/docs/1.10.0/deploy/install/install-with-kubectl/

## 2025-10-06

* ComputeBlade CM4s erased and re-imaged with Talos
* Kubernetes cluster bootstrapped

## 2024-09-03

* Realised that ConsolePi image installer is 32-bit Raspberry Pi OS, causes issues pulling docker images

* Reinstall console server with vanilla Raspberry Pi OS + [ser2net](https://github.com/cminyard/ser2net)

## 2024-09-02

* Decided to move to a container based PXE install solution, instead of splitting config across Unifi DHCP configuration & TFTP server

* Researched [matchbox](https://matchbox.psdn.io/), which looks interesting, but is quite clunky around the edges to install
