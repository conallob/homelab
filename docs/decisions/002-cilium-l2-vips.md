---
tags:
  - homelab
  - learning
---
# Cilium L2 Announcements for LoadBalancer VIPs

## Context

Cilium replaced flannel + kube-proxy as the CNI. Gateway API Gateways and
other `LoadBalancer` Services need an IP reachable from the rest of the
network, and nothing in the cluster handed one out: Services sat at
`<pending>`.

The original proposal was a separate `192.168.5.0/24` netblock. That range
was off-subnet: nodes only had addresses in `192.168.6.0/24` (gateway
`192.168.6.1`), so nothing would ARP for `192.168.5.x` on the lab VLAN and
L2 announcements could not reach it.

## Options

1. **BGP to the UniFi gateway** — Cilium BGP control plane advertises VIP
   `/32`s; works for any off-subnet range, gives ECMP. Requires FRR BGP
   config on the UniFi gateway.
2. **L2 announcements on an on-link range** — Cilium answers ARP for VIPs
   on the node interface. No routing protocol, but VIPs must be on-link for
   the lab VLAN.
3. **Carve VIPs from `192.168.6.0/24`** — L2 with no network change, but
   a smaller range shared with DHCP.

## Decision

Option 2. The lab VLAN (VLAN 5) was widened to `192.168.6.0/23` on UniFi,
with DHCP restricted to `192.168.6.10–192.168.6.254`. The upper half,
`192.168.7.0/24`, is the VIP netblock:

* `CiliumLoadBalancerIPPool` `lab-vips`: `192.168.7.1–192.168.7.254`
  (`192.168.7.255` is the `/23` broadcast address)
* `CiliumL2AnnouncementPolicy` `lab-vips`: LoadBalancer IPs announced on
  `^end0$`
* Helm values: `l2announcements.enabled: true`, `devices: end0`

Config lives in `cfg/talos/20260920/cilium-values.yaml` and
`cfg/talos/20260920/cilium-lb.yaml`.

## Consequences

* Each VIP is answered by a single node at a time, chosen by a
  `cilium-l2announce-<namespace>-<service>` Lease in `kube-system`. No load
  spreading across nodes; failover follows the lease (seconds, not
  instant).
* Verified 2026-11-07: a test Service was allocated `192.168.7.1`, and
  `192.168.1.239` (a different subnet) opened a TCP connection to it through
  the router.
* Nodes keep their `/24` DHCP leases until renewal. VIPs still work
  (Cilium answers ARP regardless of the node prefix); node-to-VIP traffic
  hairpins through the gateway until then.
* `cfg/dnsmasq.conf` still hands out `255.255.255.0`; update it if that
  dnsmasq serves the lab.
* Tinkerbell's `192.168.6.127` is outside the pool and must move to a
  `192.168.7.x` address before it can be assigned.
* BGP remains an option if load spreading or faster failover is needed.
