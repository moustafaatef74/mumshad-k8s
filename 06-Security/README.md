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

