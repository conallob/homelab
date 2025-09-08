# Bootstrapping

## Fix the Boot Order on each CM4

1. Physically move the CM4 from the basic to DEV blade
2. Boot into bootloader using `nRPIBOOT` button
3. `cd usbboot/` and add `BOOT_ORDER=0xf16472` to `recovery/boot.conf`
4. Flash recovery firmware, `sudo ./rpiboot -d recovery/`

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

## Add Nodes to Cluster

1. Transfer CM4 back to the basic board it came from
2. Boot up, then add
3. Generate Talos `secrets.yaml`, `controlplane.yaml` and `workers.yaml` using
```
talosctl gen secrets -o secrets.yaml
talosctl gen config --with-secrets secrets.yaml <cluster-name> <cluster-endpoint>
talosctl apply-config --insecure \
    --nodes [NODE IP] \
    --file controlplane.yaml
```

## Install CNI

Install Calico per https://www.talos.dev/v1.10/kubernetes-guides/network/deploying-cilium/#method-1-helm-install, specifically:

1. Install Cilium from helm, without `kube-proxy` and with `gateway-api`
```
   helm install \                                    ✔  1084  20:53:20
    cilium \
    cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true
   ```

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


