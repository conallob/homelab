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


## 2026-09-19

* Root-caused the CM4 boot hang first seen on `lab-node-1`, `lab-node-3`, and
  `lab-node-5`: GRUB prints `Booting 'A - Talos v1.14.0'` then goes
  completely silent (no kernel output at all), followed eventually by a full
  firmware-stage reset — the same visible symptom the CM5 GRUB->kernel hang
  had, but a different, now-confirmed root cause for CM4.
* Cause: the Image Factory schematic in use
  (`9c17e26911d4ad2c1899ed4dc9f1e99753ce47cbd1d154898c8e3aae3b347e4f`) set
  `extraKernelArgs: console=ttyS0,115200` — `ttyS0` (the BCM mini-UART) is
  the wrong console device on these ComputeBlade CM4 boards. Firmware/U-Boot/
  GRUB output all worked fine (they use their own console routing,
  independent of the Linux `console=` kernel arg), so the GRUB menu
  rendering correctly proved the UART wire/baud rate were fine — it just
  didn't prove the *kernel's* console matched. Once the kernel took over
  UART ownership its output never reached the wire, looking exactly like a
  dead hang.
* Fix: switch to `console=ttyAMA0,115200` (the PL011 UART). New working
  schematic: `6b700850e84fbbaa67e3558d1d8599f336f36eb063b3a3763c97f7bc7b07a760`
  (`extraKernelArgs: [console=ttyAMA0,115200, console=tty1]`, otherwise
  identical — stock `rpi_generic` overlay, Talos v1.14.0). Confirmed working
  on two boards, both reaching maintenance mode and becoming network
  reachable.
* Note: the `bcmgenet ... Link is Down` message seen right after boot is a
  **red herring**, not a fault — normal ~4-11s autonegotiation delay before
  `Link is Up` appears.
* Useful debug technique found along the way: at the GRUB menu, press `e` to
  edit the boot entry's `linux` line and change kernel args live, then
  Ctrl+X to boot — a fully reversible one-boot-only test (GRUB reloads its
  saved config next boot) that let the fix be validated on a second board
  without reflashing anything.
* Constraint driving the next step: only `lab-node-1` (ComputeBlade **Dev**
  board) has rpiboot/nRPIBOOT jumper logic for easy reflashing.
  `lab-node-2` through `lab-node-6` are ComputeBlade **Basic** boards, making
  physical reflashing far more tedious. Planned workaround instead of
  reflashing all 6: build an 8-port UART console server (using the existing
  Waveshare USB-to-8CH TTL Adaptor), GRUB-edit each Basic board's console arg
  to reach maintenance mode, then push a machine config via
  `talosctl apply-config` containing
  `machine.install.extraKernelArgs: [console=ttyAMA0,115200]` — since
  `apply-config` against a maintenance-mode node runs Talos's *initial*
  install, this bakes the fix into that node's own GRUB config on disk from
  the first install, with no reflash and no repeated manual edits needed.
* Checked whether Talos v1.14.0 fixes anything CM5/Pi5-relevant while
  investigating: no — the only CM5/Pi5-adjacent activity is in unreleased
  `siderolabs/pkgs` commits (`BCM2712_MIP` kernel config, macb EEE/TX-stall),
  which the (now-shelved) custom kernel pipeline bypassed anyway via the
  Raspberry Pi vendor kernel.
* Considered whether this same `console=ttyAMA0` fix might also resolve the
  still-unresolved CM5 GRUB->kernel hang — checked, and no: the CM5-facing
  build already defaulted to `console=ttyAMA0,115200`
  (`talos-rpi-builder`'s Makefile `EXTRA_KERNEL_ARGS` default), so the CM5
  hang has a different, still-unknown root cause. CM5/Pi5 work remains
  shelved.

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
