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


## 2026-11-10

* Added `cfg/talos/patch-logging.yaml`: nodes send Talos service logs
  (`json_lines` over TCP to `127.0.0.1:6170`) to the vector DaemonSet in
  `homelab-apps` (`apps/vector`), which forwards them, along with container
  logs, to GCP Cloud Logging. Apply with
  `talosctl patch mc --nodes <ip> --patch @cfg/talos/patch-logging.yaml`
  once vector is running, since nothing listens on that port before then
  (Talos retries, so the order isn't critical).

## 2026-11-09

* Stood up External Secrets Operator (ESO) + a self-hosted 1Password Connect
  server in `homelab-apps`, backed by the `Homelab` vault on
  `taku-ie.1password.com` — see
  [conallob/homelab-apps#6](https://github.com/conallob/homelab-apps/pull/6)
  and [conallob/homelab-apps#7](https://github.com/conallob/homelab-apps/pull/7).
  cert-manager's `cloudflare-api-token` Secret now syncs from 1Password via
  an `ExternalSecret` instead of a hand-run `kubectl create secret`, as the
  first real consumer.
* Separately, audited the Cloudflare account for every CNAME pointing at
  `metis.asclepian.ie` (7 found, across `asclepian.ie`, `conall.net`, and
  `taku.ie`) ahead of migrating that VM's vhosts elsewhere, and deleted 3
  confirmed-dead CNAMEs turned up by the same sweep (`nagios2`/`nagios3`
  .asclepian.ie, `download.infocad.ie`).
* **The real debugging story:** the `onepassword-connect` Deployment sat in
  `ContainerCreating` for 21+ hours, always citing a missing Secret named
  `op-credentials` — the upstream chart's hardcoded default — never the
  `onepassword-connect` Secret our `values.yaml` was supposed to select via
  `connect.credentialsName`. Row of dead ends before the real cause:
  - First suspected the hand-created bootstrap Secret had the wrong name or
    field key (it didn't — confirmed correct both times).
  - Then suspected Argo CD sync staleness or an Argo Rollouts field-manager
    fight over the Deployment's pod spec (ruled out: `--show-managed-fields`
    showed only `argocd-controller` and `kube-controller-manager` touching
    the object; a `workloadRef` Rollout doesn't even carry its own
    `spec.template`, so it was never in the running).
  - **Actual cause, found only once `get.helm.sh` was allowed through this
    session's egress filter and a real `helm template` became possible**:
    `--set connect.credentialsName=x` had *zero* effect on the rendered
    manifest — proof the override path itself was wrong, not a sync issue.
    Dumping the fully-merged `.Values` via a throwaway debug template
    revealed why: the upstream `connect-helm-charts` "connect" subchart's
    *own* `values.yaml` namespaces its Connect-server config under a
    top-level `connect:` key (sibling to its `operator:` key, for the
    optional Kubernetes Operator). Since our dependency alias is *also*
    `connect`, the real override path from our parent chart is
    `connect.connect.credentialsName` — ours was one level too shallow and
    silently landed next to the real default instead of overriding it.
  - Lesson: when a Helm subchart override appears to do nothing, don't trust
    static reading of the subchart's templates alone — `helm template --set
    <path>=<sentinel>` against the *real* fetched chart is the fastest way
    to prove whether a values path is even reaching the template, before
    chasing GitOps-layer explanations (sync lag, controller conflicts) that
    turned out to be red herrings here.
  - Also discovered along the way: `1Password/connect-helm` (the chart repo
    URL used throughout 1Password's own docs search results) no longer
    exists — the project renamed to `1Password/connect-helm-charts`
    sometime before this session, with the chart now at `charts/connect`.
* Fixed in
  [conallob/homelab-apps#7](https://github.com/conallob/homelab-apps/pull/7);
  confirmed working end-to-end after merge:
  `kubectl get clustersecretstore onepassword -n external-secrets` → `Valid`
  / `Ready: True`, and
  `kubectl -n cert-manager get externalsecret cloudflare-api-token` →
  `SecretSynced` / `Ready: True`.
* Added `apps/node-exporter` (DaemonSet) and `apps/kube-state-metrics` to
  `homelab-apps` — see
  [conallob/homelab-apps#8](https://github.com/conallob/homelab-apps/pull/8).
  `node-exporter`'s image tag is pinned explicitly (`image.tag`), rather
  than left to default to `v<Chart.AppVersion>`, so the binary version
  doesn't silently drift on every chart bump.
* `node-exporter`'s DaemonSet sat at `DESIRED: 4, CURRENT: 0` for hours —
  never actually creating a single pod. Root cause: the cluster enforces
  Pod Security Admission `baseline` by default, which forbids
  `hostNetwork`/`hostPID`/hostPath volumes/hostPort, all of which
  `node_exporter` legitimately needs. Fixed by labeling the namespace
  `pod-security.kubernetes.io/enforce: privileged` explicitly — the
  standard Kubernetes mechanism for opting one namespace out of a
  cluster-wide default, rather than weakening it globally. See
  [conallob/homelab-apps#9](https://github.com/conallob/homelab-apps/pull/9).
  This pattern (PSA `baseline` silently blocking a legitimately
  privileged workload, with no pods — not even failing ones — to
  `kubectl describe`) is worth remembering for the next host-level
  exporter or agent added to this cluster.
* Rewrote Tinkerbell end to end — twice. First pass
  ([conallob/homelab#5](https://github.com/conallob/homelab/pull/5)) fixed
  three real bugs sitting in the previously-committed, never-actually-used
  `cfg/tinkerbell/{values,hardware}.yaml`: a `nodeSelector` pinned to a
  node name from a previous cluster generation, a MAC reassigned to a
  different node after the 2026-09 blade swap (`HARDWARE.md` had already
  documented the swap; the Tinkerbell config just never got updated to
  match), and a `Hardware` entry duplicating both the `metadata.name` and
  `mac` of another — copy-paste damage. Also migrated off the deprecated
  per-component `stack` chart to the current unified `tinkerbell` chart
  (v0.25.0), configured `proxy` DHCP mode (the Unifi router keeps owning
  real leases; Tinkerbell only supplies PXE boot options as a second
  responder), and pinned smee to `lab-node-1` via `nodeSelector` — its
  macvlan DHCP listener needs to be on the CM4s' L2 segment, since a
  Cilium-announced LoadBalancer Service can't deliver raw broadcast DHCP
  traffic into a pod the way it can unicast traffic to a VIP. Verified
  every one of these by actually cloning `tinkerbell/tinkerbell` and
  running `helm template` against the real chart, not by reading the
  values schema and guessing — caught the chart's default `kube-vip`
  `loadBalancerClass` (this cluster uses Cilium's LB-IPAM, no kube-vip
  installed) and the `artifactsFileServer` requirement for HookOS (unused
  here — every node chainloads straight to Talos via its own `Hardware`
  object) this way.
* Second pass: moved Tinkerbell again, this time from a manual
  `helm install` documented in this repo into `homelab-apps` as
  `apps/tinkerbell`, once it became clear nothing was actually running
  that install command — Argo CD had no idea Tinkerbell was supposed to
  exist. See
  [conallob/homelab-apps#10](https://github.com/conallob/homelab-apps/pull/10)
  and [conallob/homelab#6](https://github.com/conallob/homelab/pull/6) (the
  cleanup removing the now-duplicate `cfg/tinkerbell/` from this repo).
  Rendering the real chart again caught a second PSA `baseline` rejection,
  same shape as `node-exporter`'s: Tinkerbell's Deployment runs
  `hostPID: true` plus a `privileged: true` init container (for the same
  macvlan setup), needing the identical namespace-level `privileged`
  exemption.
* Also added `Hardware` objects for the 3 CM5 control-plane nodes
  (`ctrl-1`..`ctrl-3`, `192.168.6.200`–`.202`), but with `allowPXE: false`
  and no `netboot.ipxe.url` — CM5 is a different SoC (BCM2712 vs the CM4s'
  BCM2711) and CM5/Pi5 Talos support was shelved as too buggy elsewhere
  this session (unresolved GRUB→kernel hang); there's no validated image
  to point them at yet. While wiring these up, caught a real Helm
  templating bug of our own: `{{ .allowPXE | default true }}` silently
  turned every `false` back into `true`, because Sprig's `default` treats
  Go's bool zero-value as "empty" and substitutes over it. All three
  `ctrl-*` entries rendered `allowPXE: true` on the first pass despite
  `false` being set in the data. Fixed by setting the field explicitly on
  every entry instead of relying on a fallback — worth remembering
  whenever a Helm template takes a boolean value with a `default`.

## 2026-11-07

* All nodes back to `Ready` after the flannel → Cilium migration. Root cause:
  Cilium 1.20.3 was Helm-installed on 2026-10-21 with no values, so agents
  requested `SYS_MODULE` (dropped by Talos) and, once kube-proxy was removed,
  couldn't reach the API server via `10.96.0.1`. `lab-node-1`/`lab-node-2`
  only looked `Ready` because of a stale `10-flannel.conflist`.
* Fix: removed `KubeFlannelCNIConfig` and disabled `KubeProxyConfig` in the
  Talos config (`cilium-patch.yaml`), deleted the `kube-proxy`/flannel
  objects, installed Gateway API v1.6.3 CRDs, and upgraded Cilium with
  `cilium-values.yaml` (KubePrism, kube-proxy replacement, Talos caps).
* Removed the ghost control-plane node `talos-rk5-3nq` (not an etcd member).
* Lab VLAN widened to `192.168.6.0/23`; `192.168.7.0/24` is now the Cilium
  LoadBalancer VIP netblock, announced via L2 on `end0`. See
  `decisions/002-cilium-l2-vips.md`.

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
