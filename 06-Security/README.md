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

A certificate is used to guarantee trust between 2 parties during a transaction, TLS ensures that the communication is secure and the server is who it says it is

The problem with symmetric encryption is that the key used to encrypt data is sent along to the server, so the attack might be able to get the key to the data.

## TLS in K8s

Types of certificates:

- Server Certificates (Server)
- Root Certificates (CA)
- Client Certificates (Client)

The K8s cluster consists of a master and workers nodes, All interaction between the server and the client must be secured, communication between all the components of the K8s cluster must be secured, the 2 primary requirements are to have all the various server within the cluster to use server certificates and the client to use client certificates to verify that they are who they say they are.

### Server Components

kube-api Server

- api server exposes a https service that other components and external user use to manage the cluster, so generate a certificate and a key pair
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

#### Generating Key
```
openssl genrsa -out ca.key 2048
```

#### Generating certificate signing request
```
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CS" -out ca.csr
```

#### Signing Certificate

```
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

### Admin User

#### Generating Key
```
openssl genrsa -out kube-admin.key 2048
```

#### Generating certificate signing request
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

We follow the same steps as previous, ETCD can be deployed as a cluster across multiple servers as in a HA environment, to secure communication between the different members within the cluster we must generate additional peer certificates, once peer certificates are generated specify them when starting the ETCD server

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

Generating Key
```
openssl genrsa -out apiserver.key 2048
```

Genrating certificate signing request
```
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf
```
openssl.cnf
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

The certificate for the kubelet server, it's a https sever responsible for managing the node.

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

All the certificate related operation are carried out by the controller manager, it has controllers such as csr approving, csr signing

We know that if anyone has to sign certificates, they need the ca root certificate and private key, the controller manager has 2 options where you can specify this

## Kube Config

instead of sending the certificate, key, ca every time you use kubeapi

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

## API Groups

What is the K8s API?

- We learned about kubeapi-server, whatever operation we've done so far with the cluster, we've been interacting with the apiserver wither with the kubectl utility or by rest

```
curl https://kube-master:6443/version
curl https://kube-master:6443/api/v1/pods
```

### Core Groups APIs

- /api
  - /v1
    - namespaces
    - pods
    - rc
    - events
    - endpoints
    - nodes
    - bindings
    - PV
    - PVC
    - configmaps
    - secrets
    - services

### Named Group APIs

- /apis
  - /apps
    - /v1
      - /deployments
        - list
        - get
        - create
        - delete
        - update
        - watch
      - /replicasets
      - /statefulsets
  - /extentions
  - /networking.k8s.io
    - /v1
      - /networkpolicies
  - /storage.k8s.io
  - /authentication.k8s.io
  - /certificates.k8s.io
    - /v1
      - /certificatesigningrequests

### Kubectl Proxy

user -> kubectl proxy -> kube apiserver
```
kubectl proxy
curl http://localhost:8001 -k
```

- kube proxy != kubectl proxy

## Authorization

As an admin you able to preform any operation on the cluster, but soon other will have access on the cluster like developers and 3rd party apps like monitoring apps or CI applications.

### Authorization Mechanisms

- Node
- ABAC
- RBAC
- Webhook

The Kubeapi server is accessed by kubelets, these requests are handled by a special authorizer called node authorizer, kubelet certificate should have a prefix system:node:node01 for example

#### ABAC

External access to kubeapi, for example dev-user can view, create and delete pods. you do this by creating a policy file by passing a json format like this

```json
{
  "kind": "Policy",
  "spec":{
    "user":"dev-user",
    "namespace":"*",
    "resource":"pods",
    "apiGroup":"*"
  }
}
```

```json
{
  "kind": "Policy",
  "spec":{
    "group":"developers",
    "namespace":"*",
    "resource":"pods",
    "apiGroup":"*"
  }
}
```

This can be tedious to manage as will have to pass this json to the kubeapi every time you need to grant access

#### RBAC

RBAC make this much easier, instead of associating a user directly with a set of permission, we define a role

for example we create a role for developers then we associate all the developers with that role same for security engineers

#### Webhook

What if we want to outsource our authorization mechanism?

For instance Open Policy Agent is a third party tool that help with admission control and authorization we can have k8s make an api call to open policy agent and with the info about the user and access requirements, then have OPA decide whether if the user should be permitted or no

#### Always Allow and Always Deny


## RBAC

How do we create a role?
- We do that by creating a role object

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
```

Each rule has 3 sections apiGroup, resources, verbs

The Next step is to link the user to that role, for this we create another object called role binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  - kind: Role
    name: developer
    apiGRoup: rbac.authorization.k8s.io
```

The role bindings fall under scope of name spaces

To View the created roles
```bash
kubectl get roles
```

To View rolebindings
```bash
kubectl get rolebindings
```

To View more details about the role
```bash
kubectl describe role developer
```

To View more details about the rolebinding
```bash
kubectl describe rolebinding devuser-developer-binding
```

How to know if you have access to a resource in a cluster?

```bash
kubectl auth can-i create deployments
kubectl auth can-1 delete nodes
```

What if you want to impersonate another user and check their access?

```bash
kubectl auth can-i create deployments --as dev-user
kubectl auth can-1 create pods --as dev-user
```

You can also specify the namespace in the command


```bash
kubectl auth can-1 create pods --as dev-user --namespace test
```

You can restrict access to specific resources to specific resources by adding resourceNames

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
    resourceNames: ["blue-pod", "green-pod"]
```

## Cluster Roles

Cluster roles are like roles but for cluster-scoped resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list","get","create", "delete"]
```

To link this role to a user you create a cluster role binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
  - kind: User
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  - kind: Role
    name: cluster-administrator
    apiGRoup: rbac.authorization.k8s.io
```

You can create cluster roles for namespace-scoped resource but the access of these resources won't be binded to the namespace but rather all the resources in the cluster

## Service Accounts

The concept of service account is related to another security concepts in k8s such s authentication, authorization, RBAC, etc

There are 2 types of accounts in k8s

- User Accounts
- Service Account

```bash
kubectl create serviceaccount dashboard-sa
kubectl get serviceaccount
kubectl describe serviceaccount dashboard-sa
```

When a service account is created a token is created and saved in a secret

```bash
kubectl describe secret dashboard-sa-token-kbbdm
```

If the dashboard application is deployed in the k8s cluster you can mount the secret token as a volume to the pod and then the application can read it

A default service account is automatically create within a namespace

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-k8s-dashboard
spec:
  containers:
    - name: my-k8s-dashboard
      image: my-k8s-dashboard
  serviceAccountName: dashboard-sa
```
You can choose not to mount the default service account

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-k8s-dashboard
spec:
  containers:
    - name: my-k8s-dashboard
      image: my-k8s-dashboard
  automountServiceAccountToken: false
```

In Version 1.24 when creating a service account, it no longer creates a secret or a token access secret, so you must run this command

```bash
kubectl create token dashboard-sa
```
to generate a token for the service account and it will print that token on the screen, it will have a default expiry of 1 hour

If you want to create a non-expiring token you can create a secret object like this

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecrettoken
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
```

## Image Security

```yaml
image: nginx
```

This image reference docker library which is docker.io/library/nginx, where library is the user account docker.io is the registry

To use an image from our private registry

```bash
kubectl create secret docker-registry regcred \ 
--docker-server=privat-registry.io  \
--docker-username=username  \
--docker-password=password  \
--docker-email=registry-user@org.com
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/app/internal-app
  imagePullSecrets:
    - name: regcred
```

## Security Contexts

```bash
docker run --user=1001 ubuntu sleep 3600
docker run --cap-add MAC_ADMIN ubuntu sleep 3600
docker run --privileged ubuntu sleep 3600
```

These can be configured in k8s as well, you can choose to configure the security settings on the pod or the container level, when on the pod level the setting will apply to all containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1001
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep","3600"]
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep","3600"]
      securityContext:
        runAsUser: 1001
        capabilities:
          add: ["MAC_ADMIN"]
```

## Network Policies

### Traffic

You have a 3 tier application, where the frontend server listens on port 80 and sends data to the backend server on port 5000, then the backend server sends and receives the data to the database server on port 3306

There are 2 types of traffic, ingress and egress

So the frontend server has ingress on port 80 and egress on port 5000, backend on port 5000 and 3306 respectively and the database only have ingress on port 3306

### Network Security

Assume we have 3 nodes with different pods deployed in each node, by default k8s have an always allow rule that allows any pod to communicated with any other pod within the vpc of the cluster

what if you want the frontend server to not be able to communicate with the db server, there is where you implement a network policy

A network policy is an object in k8s namespace just like pods and replicasets.

Once a policy is created it only allows the rules defined and blocks otherwise

We can use the same technique to link replicasets or services to a pod, which is labels and selectors.

We label the pod to use the same label on the port selector field in the network policy then we build our rule.

Pod Definition

```yaml
.
.
.
labels:
  role: db
.
.
.
.
```
 Network Policy Definition
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchlabels:
      role: db
  policyType:
  - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
      ports:
        - protocol: TCP
          port: 3306
```

## Developing network policies

For example we want to block any traffic coming in or unless it's through port 3306 from the api pod

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyType:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
      ports:
        - protocol: TCP
          port: 3306
```

what if there are multiple api pods in multiple namespaces
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyType:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
        namespaceSelector:
          matchLabels:
            name: prod
      ports:
        - protocol: TCP
          port: 3306
```

what if we have an external server that needs to connect to our database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyType:
    - Ingress
    - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            name: api-pod
        namespaceSelector:
          matchLabels:
            name: prod
      - ipBlock:
          cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 3306
  egress:
    - to:
      - ipBlock:
          cidr: 192.168.5.10/32
      ports:
        - protocol: TCP
          port: 80
```