# Building a kubeadm cluster on Ubuntu 22.0.4 (Jammy Jellyfish)

Here I'm going to install a kubeadm cluster with one control node and two workers at v1.24. I'm not going to detail the provisioning of the ubuntu servers here, as the cluster installation *should* work whether using VirtualBox, a hypervisor (Hyper-V, VMware etc), or cloud instances, so this guide starts from the point where you have 3 ubuntu servers provisioned that can see each other across the network. I used Hyper-V for this build, and configured static IPs for the servers.

## Initial configuration of Operating System

Do the following to all three nodes, as root user (`sudo -i`)

### Boot time setup

I found that the cluster would not come up correctly with the Ubuntu 22.0.4 default cgroups-v2. This has to be changed at boot time. I also disabled IPv6 since I'm not using it.

1. Edit `/etc/default/grub` and set `GRUB_CMDLINE_LINUX_DEFAULT` as follows

```
GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=0 ipv6.disable=1"
```

2. Run

```bash
update-grub
```

### Kernel setup

Enable the network support required to run a cluster.

1. Edit `/etc/modules` and add the line `br_netfilter`
1. Create `/etc/sysctl.d/10-kubernetes.conf` and add the following lines

```
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
```

3. Disable swap

```bash
swapoff -a
vi /etc/fstab    # remove line for `/swap.img`
rm /swap.img
```

At this point, reboot the system so that the above changes take effect.

## Packages setup

Do this on all three nodes as root user

Install the packages we're going to need

```bash
{
    apt update
    apt install -y apt-transport-https ca-certificates curl
    curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    apt update
    apt-get install -y containerd kubelet kubeadm kubectl kubernetes-cni
    systemctl stop kubelet              # So it doesn't fill the journal with errors
    apt-mark hold kubelet kubeadm kubectl
    # Stop crictl from warrning about other CRI endpoints that don't exist
    crictl config \
        --set runtime-endpoint=unix:///run/containerd/containerd.sock \
        --set image-endpoint=unix:///run/containerd/containerd.sock
}
```


## Control Plane Node Setup

### Create Cluster Configuration File

Create `kubeadm-config.yaml` with the following content

```yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.24.3          # <- At time of writing. Change as appropriate
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
controllerManager:
  extraArgs:
    "node-cidr-mask-size": "24"
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

### Boot control plane

```bash
kubeadm init --config kubeadm-config.yaml
```

### Ensure all control plane pods have started

```bash
crictl ps
```

There should be 5 containers

* etcd
* kube-apiserver
* kube-proxy
* kube-controller-manager
* kube-scheduler


### Check we can see them with kubectl

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf  # <- You can add this to root's `.bashrc` to make it permanent.
kubectl get pods -n kube-system
```

### Install CNI

You may have noticed the CoreDNS pods were in pending state from the above command. This is due to not yet having a cluster network provider. I chose Weave for this example, as it's simple to install

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

After 30 seconds or so, the weave pod will start, and so will the CoreDNS pods

```bash
kubectl get pods -n kube-system
```

### Get the join token to use at the workers

```bash
kubeadm token create --print-join-command
```

## Worker Nodes Setup

1. Install the [packages](#packages-setup) if not already done so.
1. Execute the join command output by `kubeadm token create`
1. Wait up to a minute for the nodes to come online, then from control plane

```bash
kubectl get nodes -o wide
```