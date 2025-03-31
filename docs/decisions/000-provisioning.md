# Provisioning

## Options

* tinkerbell.org
* mass.io
* iPXE and netboot.xyz

## Decision

## MAAS

Decided to just use maas.io, even though it's not containerized or Kubernetes future facing. Can always be replaced with Tinkerbell in the future

## Discounted

### tinkerbell.org

Looks interesting, but attempts to run a VM configuration to bootstrap provded to be challenging

## iPXE netboot.xyz

Looks interesting, but fell into the rabbit hole of whether the RPi 4 and in particular the CM4 can PXE vs UEFI boot, whether it can detect a PXE server, etc
