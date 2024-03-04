# Security

## Security Primitives

### Secure Host

- Disable Password Auth
- SHH Based Auth Only

### Secure K8s

- kubeapi is at the centre of all kubernetes operations

You should control access to apisever by asking 2 questions:

- Who should access the cluster?
    - Authentication
        - Files - Username, Passwords
        - Files - Username, Tokens
        - Certificates
        - External Auth providers - LDAP
        - Service Accounts
- What could they do?
    - Authorization
        - RBAC Authorization
        - ABAC Authorization
        - Node Authorization
        - Webhook Mode

### TLS Certificates
All the communication across the cluster is secured by TLS encryption

### Communication between PODs

By default PODs are able to communicate with each other

## Authentication

```bash
kubectl create serviceaccount sa1
kubectl get serviceaccount
```

### Auth Mechanisms
How Does the kubeapi server authenticates 

- Static Password File
    - Create a user-details.csv file and pass it to kubeapi --basic-auth-file=user-details.csv flag, then kubeadm will restart kubeapi sever when it's updated
    - the csv file has columns as follows password,user and id you can have a forth column with value group
- Static Token File
    - the same as the static password file but instead of password you pass a token
- Certificates
- Identity Service

## TLS Certificates

A certificate is used to gurantee trust between 2 parties during a transaction, TLS ensures that the communication is secrue and the server is who it says it is

The problem with symemetric encryption is that the key used to encrypt data is sent along to the server, so the attack might be able to get the key to the data.

## TLS in K8s

Types of certificates:

- Server Certificates (Server)
- Root Certificates (CA)
- Client Certificates (Client)

The K8s cluster consists of a master and workers nodes, All interaction between the server and the client must be secured, communication between all the components of the K8s cluster must be secured, the 2 primary requirments are to have all the various server within the cluster to use server cettificates and the client to use client certificates to verify that they are who they say they are.

### Server Components

kube-api Server

- api server exposes a https service that other ccomponents and external user use to manage the cluster, so generate a certificate and a key pair
- apiserver.crt
- apiserver.key

ETCD Server

- etcdserver.crt
- etcdserver.key

Kubelet Server

- exposes a https endpoint for kube api to interact with the worker nodes
- kubelet.crt
- kubelet.key

### Client Components

Admin user

- admin.crt
- admin.key

Scheduler

- scheduler.crt
- scheduler.key

Kube-Controller manager

- controller-manager.crt
- controller-manager.key

Kube-proxy Server

- kube-proxy.crt
- kube-proxy.key

Kube-api server

- it talks to the kubelet and etcd so it can reuse its server certificate

## Generating TLS Certificates

### Certificate Authority

#### Genrating Key
```
openssl genrsa -out ca.key 2048
```

#### Genrating certificate signing request
```
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CS" -out ca.csr
```

#### Signing Certificate

```
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

### Admin User

#### Genrating Key
```
openssl genrsa -out kube-admin.key 2048
```

#### Genrating certificate signing request
```
openssl req -new -key kube-admin.key -subj "/CN=kube-admin/O=system:masters" -out kube-admin.csr
```

#### Signing Certificate

```
openssl x509 -req -in kube-admin.csr -signkey kube-admin.key -out kube-admin.crt
```

Kube Scheduler, Kube Controller and Kube Proxy names should prefixed with the system prefix since they are system components

### How to use certificates

you can use them instead of username and password when making an api call

```
curl https://kube-apiserver:6443/api/v1/pods \
--key admin.key \
--cert admin.crt \
--cacert ca.crt
```

Or you can put the in kube-config.yaml

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority: ca.crt
      server: https://kube-apiserver:6443
    name: kubernetes
kind: Config
users:
  - name: kubernetes-admin
    user:
      client-certificate: admin.crt
      client-key: admin.key
```

### Server side certificates

#### ETCD

We follow the same steps as previous, ETCD can be deployed as a cluster across multiple servers as in a HA environment, to secure communication between the different memebers within the cluster we must generate additional peer certificates, once peer certificates are generated specify them when starting the ETCD server

```yaml
- etcd
    - --key-file=
    - --cert-file=
    - --client-cert-auth=true
    - --trusted-ca-file=
    .
    .
    .
    - --peer-key-file=
    - --peer-cert-file=
    - --peer-client-cert-auth=true
    - --peer-trusted-ca-file=
    .
    .

```

#### Kube API Server

Genrating Key
```
openssl genrsa -out apiserver.key 2048
```

Genrating certificate signing request
```
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf
```
openss.cnf
```conf
[req]
req_extentions = v3_req
distinguished_name = req_distinguished_name
[ v3_req ]
basicConstraints = CA:FALSE
keyUSage = nonRepudiation,
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87

```

Signing Certificate

```
openssl x509 -req -in apiserver.csr -signkey apiserver.key -out apiserver.crt
```

```
ExecStart=/usr/local/bin/kube-apiserver
    --etcd-cafile=/var/lib/kubernetes/ca.pem
    --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt
    --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key
    .
    .
    .
    --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem
    --kubelet-client-certificate=/var/lib/kubernetes/apiserver-kubelet-client.crt
    --kubelet-client-key=/var/lib/kubernetes/apiserver-kubelet-client.key
    .
    .
    .
    --client-ca-file=/var/lib/kubernetes/ca.pem
    --tls-cert-file=/var/lib/kubernetes/apiserver.crt
    --tls-private-key-file=/var/lib/kubernetes/apiserver.key
```

#### Kubelet Server

The certificate for the kubelet server, it's a https sever responisble for managing the node.

you need a key pair for every node, how to name the certificates?

you will name them after the node as such node01, node02 ...

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTImeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet-node01.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"

```

## View Certificates Details

```
cat /etc/kubernetes/manifests/kube-apiserver.yaml
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

Inspect service logs

```
journalctl -u etcd.service -l
kubectl logs etcd-master -n kube-system
```

View Logs if kubectl is down

```
docker ps -a
docker logs 87fc
```

## Certificate API

What is the CA server and where is it located in a K8s setup

right now the master node is the also the CA server

K8s has a built in certificate API, with it you send the CSR to the admin, the admin create a k8s api object called CertificateSigningRequest Object, once object is create all CSR are viewed by Admins and they can review and approve them, this certificate can be shared with the user

```
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
cat jane.csr | base64
```
jane-csr.yaml
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  expirationSeconds: 600
  usages:
    - digital signature
    - key encipherment
    - server auth
  request:
    ${{jane.csr to base64}}
```

```
kubectl get csr
kubectl certificate approve jane
kubectl get csr jane -o yaml
echo ${{status.certificate}} | base64 --decode
```

All the certificate related operation are carried out by the controller manager, it hase controllers such as csr approving, csr signing

We know that if anayone has to sign certificates, they nade the ca root certificate and private key, the controller manager has 2 options where you can specify this

## Kube Config

instead of sending the certificate, key, ca everytime you use kubeapi

```bash
kubectl get pods \
--server my-kube-playground:6443
--client-key admin.key
--client-certificate admin.crt
--certificate-authority ca.crt
```

you can put these info in ~/.kube/config

```
kubectl get pods
```

the config file has 3 sections clusters, users, contexts

- users specify the users
- cluster specify cluster
- context matches each user to a cluster

```yaml
apiVersion: v1
kind: Config
current-context: my-kube-admin@my-kube-playground
clusters:
  - name: my-kube-playground
    cluster:
      certificate-authority: ca.crt
      server: https://my-kube-playground:6443
context:
  - name: my-kube-admin@my-kube-playground
    context:
      cluster: my-kube-playground
      user: my-kube-admin
      namespace: kube-system
users:
  - name: my-kube-admin
    user:
      client-certificate: admin.crt
      client-key: admin.key
```

how does kubectl know which context to use?

- you can use current-context field

```
kubectl config view
kubectl config view --kubeconfig=my-custom-config
```

How to update current context?

```
kubectl config use-context prod-user@production
```
- this command also modifies the kubeconfig file

There are more commands to rename, delete, update
```
kubectl config -h
```

What about namespaces?

- the context can take an additional field to user and cluster named namespaces

Certificates

- you can replace certificate-authority field in cluster with certificate-authority-data and add the certificate content converted to base64