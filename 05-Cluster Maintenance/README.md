# Cluster Maintenance

## Operating System Upgrade

The default pod-eviction-timeout is 5 minutes, so if a node is down for more than 5 minutes, pod are terminated from the node, K8s considers it as dead

When the node comes back online, it comes online as empty without any pods

There is a safer way to make a quick upgrade and reboot; you can drain the node

```bash
kubectl drain node-01
```
the pod then are moved to another node in the cluster, the node is now marked as unschedulable.

when the node comes back online, it's still unschedulable, you can uncordon it by

```bash
kubectl uncordon node-01
```
pods moved to the other node don't automatically fall back to the original node.

there is also a command that marks the node unschedulable but does not drain the node

```bash
kubectl cordon node-01
```

## K8s releases and versions
```bash
kubectl get nodes
```
K8s version contains three parts major, minor and patch

The download package (tar.gz) has all the control plane components in it all of them have the same version, some of them have different version umber like the ETCD cluster and the CoreDNS which are different projects

## Cluster Upgrade Process in K8s

The kube-api server should have the latest version and no other component should have a version higher than it. the controller manager and the kube-scheduler can be 1 minor version lower, the kubelet and kube-proxy can be 2 minor versions lower, however kubectl can be a minor version higher or lower

Kubernetes only supports the latest 3 version

It's recommended to upgrade your cluster one minor version at a time

### Upgrading with kubeadm

Upgrading your cluster involves 2 steps, upgrading the master node then upgrading the worker nodes.

You can either upgrade one node at a time or upgrade the nodes one by one. the first option however introduces downtime.

There is a 3rd option to add new nodes with upgraded version and remove the older nodes.

```bash
kubeadm upgrade plan
```

however kubeadm does not upgrade kubelets, also kubeadm follows the Kubernetes components versions

```bash
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade apply v1.12.0
```

The next step is to upgrade kubelet on master node

```bash
kubectl get nodes
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet
```
The next step is to upgrade the worker nodes

```bash
kubectl drain node-1
apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet
kubectl uncordon node-1
```