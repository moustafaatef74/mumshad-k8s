# Storage

## Docker Storage

How Storage works with containers?

- There are 2 concepts storage drivers and volume drivers

When you install docker on a system it create a folder structure /var/lib/docker it have multiple folders under it called aufs, containers, image, volumes, etc.

There is where docker stores all of it's data by default by that it I mean images, containers running on the docker host

### Docker layered architecture

```Dockerfile
FROM Ubuntu
RUN apt-get update && apt-get -y install python
RUN pip install flask flask-mysql
COPY . /opt/source-code
ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```

```bash
docker build Dockerfile -t mmumshad/my-custom-app
```

Layers:
- Layer 1: Base Ubuntu Layer
- Layer 2: Changes in apt packages
- Layer 3: Changes in pip packages
- Layer 4: Source Code
- Layer 5: Update Entrypoint

If we want to keep the data in the container layer we can add a persistent volume

```bash
docker run -v data_volume:/var/lib/mysql mysql
```

this will mount /var/lib/docker/volumes/data_volume to the path /var/lib/mysql inside the container

another way is to bind mount

```bash
docker run -v /data/mysql:/var/lib/mysql mysql
```

using -v is old using --mount is the preferred mode as it's more verbose

```bash
docker run \
--mount type=bind, source=/data/mysql , target=/var/lib/mysql \
mysql
```

Storage Drivers:
- AUFS
- ZFS
- BTRFS
- Device Mapper
- Overlay
- Overlay2


## Volume Driver Plugins in Docker

We discussed storage drivers before, remember volumes are not handled by storage drivers, the default volume driver plugin is local, the local volume plugin helps create a volume on the docker host and store its data under the var/lib/docker/volumes directory

There are many volume driver plugins that allows you to create a volume on third party solutions like azure file storage, RexRay, Digital Ocean Block Storage, Flocker, gce-docker, GlusterFS, NetApp, Portworx, Convoy, VMware vSphere Storage

Some of these volume drivers support different storage provider

For example RexRay can be used to provision storage on EBS, S3

When  you run docker container you can use a specific volume driver such as RexRay EBS to provision a volume on AWS EBS
```bash
docker run -it \
--name mysql \
--volume-driver rexray/ebs \
--mount src=ebs-vol, target=/var/lib/mysql \
mysql
```

## Container Storage Interface (CSI)

In the past K8s used docker alone and all the code to work with docker was embedded within the k8s source code, with other container runtimes coming in such as RKT and CRI-O it was essential to extend support to work with different container runtimes and noy be dependant on k8s source code

And that's how container runtime interface came to be, the CRI is a standard that defines how an orchestration runtime like K8s would communicate with container runtimes like docker, so in the future if any new container runtime interface is developed they can simply follow the CRI standards

Similarly to extend working with network solutions the container network interface (CNI) was introduced, Now any new networking vendors could simply develop plugins based on the CNI standard and make their solution work with K8s

Also Container Storage Interface was developed to support multiple storage solutions to work with K8s, CSI is not a k8s specific standard, it's meant to be universal and if implemented allow any container orchestration tool to work with any storage vendor with a supported plugin

Currently Kubernetes, cloud foundry, mesos are on board with CSI

So here is how CSI looks like, it defines a set of RPC or remote procedure calls, that will be called by the container orchestrator and these must be implemented by the storage drivers

For example, CSI says when a pod is created and requires a volume the container orchestrator should call the create volume RPC and pass a set of details such as the volume name.

The storage driver should implement the RPC and handle the request and provision a new volume on the the storage array and return the result of the operation, similarly container container orchestrator should call the delete volume RPC and the storage driver should implement the code to decommission the volume from the volume array.

## Volumes

To persist data processed by the container we attach a volume to the container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
    - image: alpine
      name: alpine
      command: ["/bin/sh","-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
      volumeMounts:
        - mountPath: /opt
          name: data-volume
  volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
```

To configure EBS as volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
    - image: alpine
      name: alpine
      command: ["/bin/sh","-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
      volumeMounts:
        - mountPath: /opt
          name: data-volume
  volumes:
    - name: data-volume
      awsElasticBlockStore:
        volumeID: <volume-id>
        fsType: ext4
```

## Persistent Volumes

you would like to manage storage more centrally. You would like it to be configured in a way that an administrator can create a large pool of storage and then have users carve out pieces from it as required.

### Persistent Volume Definition

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

## Persistent Volume Claims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl get persistentvolumeclaim
kubectl delete persistentvolumeclaim myclaim
```
But what happens to the underlying persistent volume when the claim is deleted?

- By default, it is set to retain. Meaning the persistent volume will remain until it is manually deleted by the administrator. It is not available for reuse by any other claims.

```yaml
persistentVolumeReclaimPolicy: Retain/Delete/Recycle
```

### How to use PVC in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

## Storage Classes

Before creating a PV in your kubernetes cluster using a third part cloud provider for example AWS or Google cloud, you must create this storage on your cloud provider with the same name as the PV.

You can also define a provisioner that creates storage on GCP by creating a storage class object

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

For the PVC to use the storage class you need to add storageClassName

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: google-storage
  resources:
    requests:
      storage: 500Mi
```