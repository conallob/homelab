# Bootstrapping

## Installing the CM4 Modules for Kubernetes

1. Follow https://github.com/raspberrypi/usbboot?tab=readme-ov-file#building to get `rpiboot`
1. `wget https://factory.talos.dev/image/9c17e26911d4ad2c1899ed4dc9f1e99753ce47cbd1d154898c8e3aae3b347e4f/v1.10.4/metal-arm64.raw.xz`
1. 
```
sudo ./rpiboot -d mass-storage-gadget64
RPIBOOT: build-date 2025/07/02 pkg-version local efc5da60

Please fit the EMMC_DISABLE / nRPIBOOT jumper before connecting the power and USB cables to the target device.
If the device fails to connect then please see https://rpltd.co/rpiboot for debugging tips.

Loading: mass-storage-gadget64/bootfiles.bin
Using mass-storage-gadget64/bootfiles.bin
Waiting for BCM2835/6/7/2711/2712...

Sending bootcode.bin
Successful read 4 bytes
Waiting for BCM2835/6/7/2711/2712...

Second stage boot server
File read: mcb.bin
File read: memsys00.bin
File read: memsys01.bin
File read: memsys02.bin
File read: memsys03.bin
File read: memsys04.bin
File read: memsys05.bin
File read: memsys06.bin
File read: memsys07.bin
File read: memsys08.bin
File read: bootmain
Loading: mass-storage-gadget64/config.txt
File read: config.txt
Loading: mass-storage-gadget64/boot.img
File read: boot.img
Second stage boot server done
```
4.
```
sudo dd if=~/metal-arm64.raw bs=4M of=/dev/sda
311+1 records in
311+1 records out
1306525696 bytes (1.3 GB, 1.2 GiB) copied, 0.594591 s, 2.2 GB/s
```
