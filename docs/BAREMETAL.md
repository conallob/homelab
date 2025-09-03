# Bare Metal Configuration

## Working Configuration

1. Install talos.dev onto `dev` blade via USB key
2. Create single node k8s cluster, enabling control-plane nodes as workers, per
3. Install cilium.io as the CNI per
4. Install tinkerbell, using a mix of:
  1. https://tinkerbell.org/docs/setup/install/#tldr to install a modern tinkerbell version
  1. https://docs.computeblade.com/blade/advanced-guides/pxe-booting#installing-tinkerbell-on-a-single-node-kubernetes-installation to configure the hardware
  1. https://www.talos.dev/v1.11/kubernetes-guides/configuration/pod-security/#override-the-pod-security-admission-configuration to finally resolve the `PodSecurity` admission warnings + timeouts


<details>
<summary>Aborterd Configuration Attempts</summary>

### DHCP

* DHCP configured on Unifi UDM-Pro SE

* dnsmasq additonal config snippet on Unifi UDM-Pro SE:

```dnsmasq.conf
cat /run/dnsmasq.dhcp.conf.d/Lab-dhcp-pxe-options.conf

# Select the Lab vlan
interface=br5

pxe-service=0,Raspberry Pi Boot
dhcp-match=set:arm64,option:client-arch,b
dhcp-boot=tag:arm64,netboot.xyz-arm64.efi,lab.taku.ie,192.168.1.3
dhcp-match=set:amd32,option:client-arch,2
dhcp-match=set:amd32,option:client-arch,6
dhcp-boot=tag:amd32,netboot.xyz.efi,lab.taku.ie,192.168.1.3
dhcp-match=set:amd64,option:client-arch,7
dhcp-match=set:amd64,option:client-arch,8
dhcp-match=set:amd64,option:client-arch,9
dhcp-boot=tag:amd64,netboot.xyz.efi,lab.taku.ie,192.168.1.3
```

### netboot.xyz

* Installed as a Home Assistant add-on, thanks to
  https://github.com/FaserF/hassio-addons/tree/master/netboot-xyz
* Updated to use UEFI bootloader from the latest netboot.xyz release
* Install https://github.com/raspberrypi/firmware and
  https://github.com/pftf/RPi4

### DHCP vis dnsmasq

* Unifi UDM-Pro SE configured as DHCP relay, with dnsmasq installed on `cmdnode`

```shell
podman run -d -it --rm  \
  --name dnsmasq \
  --net=host \
  -e "DNS1=192.168.1.3" -e "DNS2=192.168.6.1" \
  -v /etc/dnsmasq.conf:/etc/dnsmasq.conf \
  --cap-add=NET_ADMIN,NET_RAW \
  ghcr.io/dockur/dnsmasq:2.91
```

### netboot.xyz

Running on `cmdnode` with:

```shell
podman run -d --rm --name=netbootxyz  \
    --net=host -v /mnt/config:/config \
    -v /mnt/assets:/assets \
    --restart unless-stopped \
    ghcr.io/netbootxyz/netbootxyz:latest
```

</details>

## Lessons Learnt

* Raspberry Pi PXE defaults to checking for CPU S/N, not Network MAC address. Changing from MAC to S/N allowed
* Flashing Talos Intstaller onto the eMMC of the Dev ComputeBlade was ultimately a waste of time and caused a distraction

## Additional Notes

* https://luke.mallon.ie/posts/2025-05/compute-blade-pxe-boot/
* https://gist.github.com/Roxedus/08a1351e4c092b7527a8830dd3456d7b
