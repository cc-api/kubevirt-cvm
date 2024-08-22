# KubeVirt-CVM Deployment Guide

This deployment guide shows how to boot Intel TDX guest (TD) with KubeVirt. The TD runs in Kubernetes and managed by KubeVirt.

It supports to deploy on Ubuntu 24.04 and Ubuntu 23.10. The follows will use Ubuntu 24.04.

## Prerequisite

1. Make sure you have [Supported Hardware](https://github.com/canonical/tdx/tree/mantic-23.10?tab=readme-ov-file#supported-hardware).
2. Setup TDX host following [guide](https://github.com/canonical/tdx/tree/mantic-23.10?tab=readme-ov-file#4-setup-tdx-host).
3. Install Kubernetes on the host following [official guide](https://kubernetes.io/docs/setup/). Or you can use below commands to setup an experimental environment quickly.
    ```
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ```

## Install KubeVirt

```bash
# ubuntu 23.10
kubectl create -f ./manifests/kubevirt-operator.yaml
kubectl create -f ./manifests/kubevirt-cr.yaml

# centos stream 9
kubectl create -f ./manifests/kubevirt-operator-centos.yaml
kubectl create -f ./manifests/kubevirt-cr.yaml
```

- Check the components if the installation is successful
```
kubectl get all -n kubevirt
```

Check the kubevirt components `virt-api`, `virt-controller`, `virt-handler` if are running.

## Prepare CVM image

1. Fetch guest image
    - CentOS Stream 9: Download the Red Hat Enterprise Linux 9.4 KVM Guest Image.
    - Ubuntu 23.10: Follow [guide](https://github.com/canonical/tdx/tree/mantic-23.10?tab=readme-ov-file#5-setup-td-guest) to create a TDX guest image.
2. Convert the image to raw format. The image will be used in the next step.

    ```
    qemu-img convert -f qcow2 -O raw tdx-guest-ubuntu-23.10-generic.qcow2 tdx-guest-ubuntu-23.10-generic.img
    ```

## Boot TD

Update image path in VMI yaml file [vmi-hostpath.yaml](./manifests/vmi-hostpath.yaml). Set it to the path of your guest image.


```
  volumes:
  - hostDisk:
      capacity: 10Gi
      path: <your image path>
      type: DiskOrCreate
    name: host-disk
```

The default TDVM configuration is 4vCPUs/16GMemory, and it can also be configured in the yaml file.

Create VMI with below command.

```bash
kubectl create -f ./manifests/vmi-hostpath.yaml
```

After deployment, the status of VMI should be `Running`.

```bash
kubectl get vmi
```

```console
NAME            AGE   PHASE     IP              NODENAME       READY                                                                                                    
vmi-ubuntu-td   36s   Running   172.10.13.190   css-spr-prc1   True 
```

When the status of vmi changes to **True**, it's time to login. Use `virtctl` to login the VM.

```bash
# console mode
./virtctl console vmi-ubuntu-td

# ssh mode
./virtctl ssh tdx@vmi-ubuntu-td
```

## Check if tdx enabled

Run the following command in the TD guest.
```bash
dmesg | grep -i tdx
```
If the dmesg not contains such message, means TDX is not enabled.
```console
[    0.000000] tdx: Guest detected
```
