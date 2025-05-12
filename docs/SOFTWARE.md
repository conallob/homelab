# Software

## Console Server

### Current

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
