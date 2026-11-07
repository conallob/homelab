# Bootstrapping

## Fix the Boot Order on each CM4 and CM5

`BOOT_ORDER=0xf21` = PXE first, eMMC fallback, loop (nibbles read right-to-left: `1`=eMMC, `2`=network/PXE, `f`=restart sequence). All nodes use onboard eMMC for the OS install; NVMe (CM4 only) is excluded from the boot order intentionally.

### CM4

1. Physically move the CM4 from the basic to DEV blade
2. Fit the nRPIBOOT jumper, connect USB slave port, power on
3. `cd usbboot/` and set `BOOT_ORDER=0xf21` in `recovery/boot.conf`
4. Flash: `sudo ./rpiboot -d recovery/`
5. Move CM4 back to its original blade

### CM5

> **Note:** Pre-built `rpiboot` binaries do not support BCM2712 (CM5) — must build from source.
> See https://github.com/raspberrypi/usbboot#building

1. Fit the nRPIBOOT jumper, connect USB slave port, power on
2. `cd usbboot/` and set `BOOT_ORDER=0xf21` in `recovery5/boot.conf`
3. Flash: `sudo ./rpiboot -d recovery5/`

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
1. If necessary, use `parted` or `Disk Utility` to erase partitions
1.
```
sudo dd if=~/metal-arm64.raw bs=4M of=/dev/sda
311+1 records in
311+1 records out
1306525696 bytes (1.3 GB, 1.2 GiB) copied, 0.594591 s, 2.2 GB/s
```

## Add Nodes to Cluster

Following https://www.talos.dev/v1.11/introduction/prodnotes/

1. Transfer CM4 back to the basic board it came from
2. Boot up, then add
3. Generate Talos `secrets.yaml`, `controlplane.yaml` and `workers.yaml` using
```
CONTROL_PLANE_IP=("<control-plane-ip-1>" "<control-plane-ip-2>" "<control-plane-ip-3>")
WORKER_IP=("<worker-ip-1>" "<worker-ip-2>" "<worker-ip-3>")
export CLUSTER_NAME=<your_cluster_name>
export YOUR_ENDPOINT=<your_endpoint>
talosctl gen secrets -o secrets.yaml
talosctl gen config --with-secrets secrets.yaml $CLUSTER_NAME https://$YOUR_ENDPOINT:6443
talosctl get disks --insecure --nodes <node-ip-address>
# Configure Control Plane Nodes
for ip in "${CONTROL_PLANE_IP[@]}"; do
  echo "=== Applying configuration to node $ip ==="
  talosctl apply-config --insecure \
    --nodes $ip \
    --file controlplane.yaml
  echo "Configuration applied to $ip"
  echo ""
done
# Configure Workers
for ip in "${WORKER_IP[@]}"; do
  echo "=== Applying configuration to node $ip ==="
  talosctl apply-config --insecure \
    --nodes $ip \
    --file worker.yaml
  echo "Configuration applied to $ip"
  echo ""
done
talosctl config endpoint <control_plane_IP_1> <control_plane_IP_2> <control_plane_IP_3>
talosctl kubeconfig --nodes <control-plane-IP>
```

## Install CNI

Install Cilium via Helm per https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium, without `kube-proxy` and with `gateway-api`.
The Talos machine config must already set `cluster.network.cni.name: none` and disable kube-proxy (see `cfg/talos/20260920/cilium-patch.yaml`).

> [!warning]
> Never install the chart without `cilium-values.yaml`. The chart defaults request `SYS_MODULE`, which Talos drops, and try to reach the API server via the `10.96.0.1` ClusterIP, which doesn't route without kube-proxy. Every agent crashloops and all nodes go `NotReady`.

1. Install the Gateway API v1.6.3 standard CRDs (must exist before Cilium starts)
```
BASE=https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard
for crd in gatewayclasses gateways httproutes referencegrants grpcroutes backendtlspolicies tlsroutes; do
  kubectl apply --server-side -f $BASE/gateway.networking.k8s.io_${crd}.yaml
done
```

2. Install Cilium with the Talos values: KubePrism (`localhost:7445`) as the API endpoint, kube-proxy replacement, Talos-safe capabilities, Gateway API and L2 announcements on `end0`
```
helm repo add cilium https://helm.cilium.io/ && helm repo update cilium
helm upgrade --install cilium cilium/cilium --version 1.20.3 -n kube-system \
  -f cfg/talos/20260920/cilium-values.yaml
```

3. Create the LoadBalancer VIP pool (`192.168.7.0/24`) and L2 announcement policy. See `docs/decisions/002-cilium-l2-vips.md`
```
kubectl apply -f cfg/talos/20260920/cilium-lb.yaml
kubectl get ciliumloadbalancerippools   # IPS AVAILABLE should be 254
```

<details>
  <summary>Previously</summary>

## Install Tinkerbell

Install tinkerbell per https://docs.computeblade.com/blade/advanced-guides/pxe-booting

TODO(conallob): Document tinkerbell steps
```
kubectl get pods -n tinkerbell -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE            NOMINATED NODE   READINESS GATES
hookos-569c8c9df4-4vmpn   2/2     Running   0          25m   10.244.0.239   talos-t13-h3c   <none>           <none>
...
# Get the pod CIDRs to set as trusted proxies
TRUSTED_PROXIES="0.0.0.0/0"

# Set the LoadBalancer IP for Tinkerbell services
LB_IP=192.168.6.127

# Set the artifacts file server URL for HookOS
ARTIFACTS_FILE_SERVER=http://10.244.0.239:7173

# Specify the Tinkerbell Helm chart version, here we use the latest release.
TINKERBELL_CHART_VERSION=v0.20.1
```
```
helm install tinkerbell oci://ghcr.io/tinkerbell/charts/tinkerbell \
  --version $TINKERBELL_CHART_VERSION \
  --create-namespace \
  --namespace tinkerbell \
  --wait \
  --set "trustedProxies={${TRUSTED_PROXIES}}" \
  --set "publicIP=$LB_IP" \
--set "artifactsFileServer=$ARTIFACTS_FILE_SERVER"
```

</details>

