---
tags:
  - homelab
  - learning
---
# Homelab Hardware

## Talos Image Schematics

| **Node Type** | **Overlay** | **Schematic ID** | **Notes** |
|---------------|-------------|------------------|-----------|
| Raspberry Pi 5 | `rpi_5` | `74e4edb7eb35db8cc5815ca6e2183eded5fd855e904d973e6de64f503e1c02df` | `console=ttyAMA0,115200` |
| CM4 | `rpi_generic` | `6b700850e84fbbaa67e3558d1d8599f336f36eb063b3a3763c97f7bc7b07a760` | `console=ttyAMA0,115200` + `console=tty1` — supersedes `9c17e269...` below, which hangs silently after GRUB (see [CHANGELOG](CHANGELOG.md) 2026-09-19) |
| ~~CM4 (superseded)~~ | `rpi_generic` | ~~`9c17e26911d4ad2c1899ed4dc9f1e99753ce47cbd1d154898c8e3aae3b347e4f`~~ | `console=ttyS0,115200` — **do not use**, hangs after "Booting Talos" on ComputeBlade CM4s (wrong UART device) |

## Current Configuration

* 1x [Unifi USW Pro Max 24](https://techspecs.ui.com/unifi/switching/usw-pro-max-24-poe)

* 6x [Raspberry Pi Compute Module 4 - CM4008032](https://www.raspberrypi.com/products/compute-module-4/?variant=raspberry-pi-cm4008032)

* 1x [ComputeBlade Dev](https://docs.computeblade.com/blade/)

* 5x [ComputeBlade Basic](https://docs.computeblade.com/blade/)
  
* 3x [ComputeBlade Smart Fan Units](https://docs.computeblade.com/fan-unit/)

* 1x [Raspberry Pi 4 Model B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/) with a [Waveshare PoE Hat E](https://www.waveshare.com/wiki/PoE_HAT_(E))

* 6x [Pimoroni USB-UART Cables](https://shop.pimoroni.com/products/usb-to-uart-serial-console-cable?variant=288389664)

* 1x [Waveshare USB-to-8CH TTL Adaptor](https://www.waveshare.com/usb-to-8ch-tt.htm)

## Previous Configuration

* 4x [Raspberry Pi Compute Module 4 - CM4008032](https://www.raspberrypi.com/products/compute-module-4/?variant=raspberry-pi-cm4008032)

* 4x [Waveshare CM4-IO-BASE-B](https://www.waveshare.com/wiki/CM4-IO-BASE-B)

* 1x [Raspberry Pi 4 Model B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/)

* 4x [Pimoroni USB-UART Cables](https://shop.pimoroni.com/products/usb-to-uart-serial-console-cable?variant=288389664)

* 1x [Waveshare USB-to-8CH TTL Adaptor](https://www.waveshare.com/usb-to-8ch-tt.htm)

* 5x [100/1000 PoE to USB-C PoE Adapters](https://www.aliexpress.com/item/1005005653835652.html?spm=a2g0o.order_list.order_list_main.5.56ec18029lfC09)

## Deployment Details

| **Node**   | **Switch Port** | **MAC Address**     | **Board**     | **Compute Module** |
|------------|-----------------|---------------------|-------------------------------|-------------------|
| lab-node-1 | Port 1          | `d8:3a:dd:9f:54:db`	| Compute Blade v1 MK4 - Dev   | CM4008032         |
| lab-node-2 | Port 2          | `d8:3a:dd:9f:51:5e`  | Compute Blade v1 MK4 - Basic | CM4008032         |
| lab-node-3 | Port 3          | `d8:3a:dd:9f:54:0e`	| Compute Blade v1 MK4 - Basic | CM4008032         |
| lab-node-4 | Port 4          | `d8:3a:dd:9f:50:fc`	| Compute Blade v1 MK4 - Basic | CM4008032         |
| lab-node-5 | Port 5          | `d8:3a:dd:9f:51:bd`  | Compute Blade v1 MK4 - Basic | CM4008032         |
| lab-node-6 | Port 6          | `d8:3a:dd:9f:50:45`  | Compute Blade v1 MK4 - Basic | CM4008032         |
| ctrl-plane-1 |               | `88:a2:9e:51:9c:76`  |                              | CM5016064         |
| ctrl-plane-2 |               | `88:a2:9e:51:a1:c7`  |                              | CM5016064         |
| ctrl-plane-3 |               | `88:a2:9e:51:a3:0a`  |                              | CM5016064         |
| ctrl-plane-temp |            |                      | Raspberry Pi 5 Model B 8GB   | RPi5 (temporary)  |

## Previous Deployment Details

| **Node**   | **Switch Port** | **MAC Address**     | **Board**     | **Compute Module** |
|------------|-----------------|----------------------|---------------|--------------------|
| lab-node-1 | Port 1          | `d8:3a:dd:9f:50:45`	| CM4-IO-BASE-B | CM4008032         |
| lab-node-2 | Port 2          | `d8:3a:dd:9f:51:bd`	| CM4-IO-BASE-B | CM4008032         |
| lab-node-3 | Port 3          | `d8:3a:dd:9f:54:0e`	| CM4-IO-BASE-B | CM4008032         |
| lab-node-4 | Port 4          | `d8:3a:dd:9f:54:db`	| CM4-IO-BASE-B | CM4008032         |
