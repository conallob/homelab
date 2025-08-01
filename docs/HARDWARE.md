# Homelab Hardware

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
|------------|-----------------|---------------------|-----------------------------|-------------------|
| lab-node-1 | Port 1          | `d8:3a:dd:9f:54:db`	| Compute Blade v1 MK4 - Dev | CM4008032         |
| lab-node-2 | Port 2          |	                    | Compute Blade v1 MK4 - Dev | CM4008032         |
| lab-node-3 | Port 3          | 	                   	| Compute Blade v1 MK4 - Dev | CM4008032         |
| lab-node-4 | Port 4          |	                   	| Compute Blade v1 MK4 - Dev | CM4008032         |
| lab-node-5 | Port 5          |	                    | Compute Blade v1 MK4 - Dev | CM4008032         |
| lab-node-6 | Port 6          |	                    | Compute Blade v1 MK4 - Dev | CM4008032         |


## Previous Deployment Details

| **Node**   | **Switch Port** | **MAC Address**     | **Board**     | **Compute Module** |
|------------|-----------------|----------------------|---------------|--------------------|
| lab-node-1 | Port 1          | `d8:3a:dd:9f:50:45`	| CM4-IO-BASE-B | CM4008032         |
| lab-node-2 | Port 2          | `d8:3a:dd:9f:51:bd`	| CM4-IO-BASE-B | CM4008032         |
| lab-node-3 | Port 3          | `d8:3a:dd:9f:54:0e`	| CM4-IO-BASE-B | CM4008032         |
| lab-node-4 | Port 4          | `d8:3a:dd:9f:54:db`	| CM4-IO-BASE-B | CM4008032         |
