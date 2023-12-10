# Mumshad CKA Course Notes

## Cluster Architecture

### Master Node (Control Plane)

#### ETCD Cluster

Highly available key value store database that stores information about the cluster

#### kube-scheduler

-  Identifies the right node to put the container on

#### Controller-manager

##### Node controller
- take care of nodes, responsible of onboarding new nodes to the cluster
##### Replication controller
- make sure that the desired number of containers are running at all time at a replication group
controller-manager

#### kube-apiserver

- responsible for orchestrating all operation within the cluster, it's the primary management component, it exposes the k8s api that allow users to make changes and manage the k8s cluster

#### All the components for the master node can be run on containers


### Worker Nodes

#### container runtime

- could be rkt, containerd or docker

#### kubelet

- it's the captain of the ship, it's a agent the runs on each node of the cluster, it listens for instructions from apiserver and deploys and destroys containers on the node as desired

#### kube-proxy

- allows communication between containers on nodes

## Docker vs ContainerD

### Docker

- at first k8s only supported docker, the k8s introduced cri which allowed other container runtimes to run but only if they adhere to the OCI (open container initiative), consists of imagespec (how an image should be built) and runtimespec (defined on how any container runtime should be developed)

### Dockershim

- K8s introduced dockershim to continue to support docker outside of the runtime interface

- support for dockershim was removed in 1.24

### containerd

- it's a part of docker but now it's a separate project, you can install containd without the other docker components

- it comes with a cli (ctr), which is not very user friendly and supports only limited features
```
ctr images pull {image}
ctr run {image}
```

- a better alternative is nerdctl

### nerdctl

- provide a docker-like cli for containerd
- supports docker compose
- supports the newest features in containerd (encrypted container images, lazy pulling, p2p image distribution, image signing and verifying, namespaces in k8s)

```
nerdctl run --name redis redis:alpine
nerdctl run --name nginx -p 80:80 -d nginx
```

### crictl

- provides a cli for cri compatible container runtimes
- installed separately
- used to inspect and debug container runtimes (not to create containers ideally)
- works across different runtimes

```
crictl pull {image}
crictl images
crictl ps -a
crictl exec -i -t {image_id} ls
crictl logs {image_id}
crictl pods
```

- mostly the same commands as docker with slight differences

- prior to version 1.24 the crictl connected to runtime endpoints:
    unix:///var/run/dockershim.sock
    unix:///run/containerd/containerd.sock
    unix:///run/crio/crio.sock
    unix:///var/run/cri-dockerd.sock
- starting 1.24
    unix:///run/containerd/containerd.sock
,    unix:///run/crio/crio.sock
    e theunix:///var/run.cri-dockerd.sock

- users are encouraged to set the runtime endpoints manually

```
crictl --runtime-endpoint
export CONTAINER_RUNTIME_ENDPOINT
```

## ETCD Introduction

### What is ETCD

- It's a distributed reliable key-value store, that is simple, secure and fast

- stores info as pages, changes to a file doesn't affect the other

-  to install etcd:
```
curl -L https://github.com/etcd-io/etcd/releases/.......
tar xzvf etcd-v3.3.11-linux-amd64.tar.gz
./etcd
```

- etcd runs on port 2379
- a default client that comes with etcd is etcd control client

```
./etcdctl set key1 value1
./etcdctl get key1
```

#### History of ETCD

- started on AUG 2013
- CNCF incubation NOV 2018
- important to know the difference between v2 and v3

- how to know which version of etcd you have
```
./etcdctl --version
```

- to change etcdctl version
```
export ETCDCTL_API=3
```

- etcd v3 commands
```
./ectdctl put key1 value1
./ectdctl get key1
```

## etcd in k8s

- the etc store stores info about nodes, pods, configs, secrets, accounts, roles, bindings and other

### manual setup

- etcd listens on --advertise-client-urls https://${INTERNAL_IP}:2379

### kubeadm setup

```
kubectl get pods -n kube-system
```

- it's deployed a pod

### to list all the keys stored by etcd

```
kubectl exec etcd-master -n kube-system etcdctl get / --prefix -keys-only
```

- you'll have all the k8s components:
    register ->
        minions
        pods
        replicasets
        deployments
        roles
        secrets

- in a HA environment, you'll have multiple master nodes, you have to configure the ETCD --initial-cluster flag is where you shall specify the different etc instances

## kube-apiserver

### Architecture

- kube-apiserver authenticate user and validate request then pass on to the ETCD cluster to retrieve data, then the etcd is updated, the scheduler identifies the node to update, then communicates that to the apiserver and updates it to the ETCD cluster, the api server then passes that request to the kubelet to update the underlying node with the new pod, the kubelet then communicates with the apiserver with the updates it did, then the apiserver updates the ETCD cluster with the updates done by the kubelet

#### how to view apiserver

```
kubectl get pod -n kube-system
```

- kube-apiserver-master, you can see the apiserver option using this command (kubeadm)

```
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

- in a normal kubernetes setup

```
cat /etc/systemd/system/kuber-apisever.service
```

- you can also see the running process of kube-apiserver using this command

```
ps -aux | grep kube-apiserver
```

## Kube-Controller-Manager

- it's a process that continuously monitors the state of the components of the system
- Node controller monitors the status of the nodes through the api server, it checks the health of the node every 5 seconds, it also has 40 seconds of grace period until it marks the node as unreachable.

- after the node is marked unreachable, it gives it 5 minutes to comeback up, if it does it removes the pods provisioned on that node and provisions them to another healthy node

- Replication Controller, it responsible for monitor the status of replica sets and ensures that the desired number of pods are available within a replicaset, if a pod dies it provisions another one

### kubeadm

- Contoller manager is auto deployed as a pod in the master node

- you can navigate the configurations of the controller manager through this command

```
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

- in a non kubeadm setup

```
cat /etc/systemd/system/kube-controller-manager.service
```

- you can also see the running process and the effective options through:

```
ps -aux | grep kube-controller-manager 
```

## kube-scheduler

- it's only responsible for deciding which pod goes on which node, it does not actually put the pod on the node that's the job of the kubelet (captain of the ship)

- the scheduler finds the best node to put the new pod to

- the scheduler first filter the nodes that are not fit for the new pod

- then it ranks the nodes and decides which one to go with

- there are many strategies for scheduler ranking (resource requirements and limits, taints and tolerations, node selectors/affinity)

## kubelet

- they are the captains of the ship (node)

- it registers the node with a k8s cluster

- create pods through container runtime

- it continues to monitor the node and the pods and reports it status to the api server

### how to install kubelet

- kubeadm does not automatically deploy a kubelet

- you shall always manually install the kubelet

## kube-proxy

- for nodes to communicate with each other through a pod network

- node communicate with each other through services, but a service is not an actual thing, it's a virtual component that only lives in k8s memory.

- kube-proxy is a process that runs on every node on a k8s cluster, it looks for new services and each time it finds a new service it creates the appropriate rules on the each node to forward traffic to those services to underlying the pods, one way it does this is through ip table rules, it creates an ip table rule on each node in the cluster to forward traffic to the service ip to the ip of the pod

### installing kube-proxy

- can be downloaded as a binary and run it as a service

- it's already deployed as a part of kubeadm, in fact it's deployed as a daemonset, so single pod is always deployed on every node of the cluster

## Pods

- k8s does not deploy the containers directly on the nodes, however it's encapsulated in a pod then deployed on a node, a pod is the smallest object in k8s

- pods always have 1 to 1 relation with containers

- However you can have multiple containers of different kinds, for example sidekiqs, they can share the same storage, and they can reference each other as localhost

### kubectl command

- it deploys a docker container using a pod

```
kubectl run nginx --image nginx
kubectl get pods
```

## Pods with YAML

- a k8s config file always contains 4 fields (apiVersion, kind, metadata, spec)

- kind refers to the kind of the object we're trying to create (Pod, ReplicaSet, Deployment)

- metadata is data above the object, it's a dictionary consists of (name:string, labels:key-value-pairs)

- spec is the specification of the object, it's different for every kind of object, it's a dictionary

```
apiVersion: v1
kind: Pod
metadata:
    name: my-app
    labels:
        env: prod
        type: backend
        stack: crm
spec:
    containers:
        - name: nginx-container
          image: nginx
```

```
kubectl create -f pod-definition.yml
```

- once you create it use the following command to list the pods in the cluster

```
kubectl get pods
```

- to get details about the pod

```
kubectl describe pod my-app
```

