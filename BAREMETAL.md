# Bare Metal Configuration


## DHCP vis dnsmasq

```
podman run -d -it --rm  \
  --name dnsmasq \
  --net=host \
  -e "DNS1=192.168.1.3" -e "DNS2=192.168.6.1" \
  -v /etc/dnsmasq.conf:/etc/dnsmasq.conf \
  --cap-add=NET_ADMIN,NET_RAW \
  ghcr.io/dockur/dnsmasq:2.91
```
