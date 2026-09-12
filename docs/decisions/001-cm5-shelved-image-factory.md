# CM5/Pi5 Shelved, CM4 Moved to Image Factory

## Context

`talos-rpi-builder` (a separate, related repo — a custom Talos Linux image
builder producing a unified `rpi_generic` image for CM4/CM5/Pi4/Pi5) carried
a custom build pipeline to work around upstream gaps: NVMe/PCIe boot and
unified CM4/CM5 support depend on an unmerged upstream PR
([siderolabs/sbc-raspberrypi#88](https://github.com/siderolabs/sbc-raspberrypi/pull/88)),
plus a handful of locally-carried DTB/kernel patches (see that repo's
CLAUDE.md for the full list).

CM5/Pi5 support hit an unresolved GRUB→kernel silent reboot loop that
persisted across multiple patch attempts.

## Decision

* **CM5/Pi5: shelved entirely.** Too buggy to justify continued investment
  right now. Revisit if/when upstream PR #88 merges or the CM5 issues get
  resolved properly upstream.
* **CM4: moved off the custom build pipeline, onto Talos's official Image
  Factory** (factory.talos.dev) instead:
  * Schematic ID: `9c17e26911d4ad2c1899ed4dc9f1e99753ce47cbd1d154898c8e3aae3b347e4f`
  * Stock `siderolabs/sbc-raspberrypi` `rpi_generic` overlay
  * `extraKernelArgs: console=ttyS0,115200`
  * Talos v1.14.0

## Rationale

CM4 doesn't need the custom pipeline's fixes — those exist specifically for
CM5/Pi5 NVMe boot and the D0 BCM2712 stepping. Using the stock Image Factory
build for CM4 removes an entire maintenance surface (patch regeneration on
every Talos/kernel bump, overlay rebuilds, GHCR publishing) for a device
class that doesn't need any of it.

## Status

`talos-rpi-builder`'s own build pipeline stays in place and was still bumped
to Talos v1.14.0 in this same work cycle (three merged PRs — see that repo's
CHANGELOG/PR history) — it remains valid should CM5/Pi5 work resume later,
but CM4 provisioning in this repo (`homelab`) now uses the Image Factory
image, not an artifact from that pipeline.

Open question at time of writing: whether the Image Factory image boots
cleanly on CM4 eMMC — an UNRELATED eMMC boot failure was hit on the
`talos-rpi-builder` pipeline's own v1.14.0 image first (see
[BAREMETAL.md](../BAREMETAL.md#lessons-learnt)); it's not yet confirmed
whether that symptom also affects the Image Factory image.
