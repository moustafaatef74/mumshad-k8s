# Application Lifecycle Management

## Deployments (Updates and Rollbacks)

When you create a deployment it triggers a rollout whicch create a new deployment revision, in the future a when the container version is updated to a new one a new rollout is triggered and a new deployment revision is created.

```bash
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deplyment/myapp-deployment
```

### Deployment Strategies

- Recreate Strategy (Downtime included)

- Rolling Update Strategy (No Downtime) (Default)

```bash
kubectl apply -f deployment-definition.yaml
kubectl set image deployment/myapp-deployment \
nginx-container=nginx:1.9.1
```

```bash
kubectl describe deployment myapp-deployment
```

### Upgrades

The deployment object creates new replicaSet and starts deploying the container and at the same time taking doen the old containers in the old replicaSet

### Rollbacks

```bash
kubectl rollout undo deployment/myapp-deployment
```

## Configure Application (Application Commands)

```bash
docker run ubuntu
docker ps
docker ps -a
```

Container is not meant to run an operating system, it exits since it does not have a task running

a command is what defines what the task for the container

```bash
docker run ubuntu-sleeper
```

```dockerfile
FROM Ubuntu
CMD ["sleep","5"]
```

an entry point is where you can pass parameters to a command through the docker run

```bash
docker run ubuntu-sleeper 10
```

```dockerfile
FROM Ubuntu
ENTRYPOINT ["sleep"]
```

you can combine both to allow for more flexible commands to pass a default value if no value is passed

```bash
docker run ubuntu-sleeper 10
```

```dockerfile
FROM Ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

You can also modify the entrypoint command at runtime using --entrypoint flag

```bash
docker run --entrypoint echo ubuntu-sleeper 10
```

## Commands and arguments in K8s Pod

```yaml
apiVersion: v1
kind: pod
metadata:
    name: ubuntu-sleeper-pod
spec:
    containers:
        - name: ubuntu-sleeper
          image: ubuntu-sleeper
          command: ["echo"] #entrypoint
          args: ["10"] #command
```

## ENV Vars in K8s

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: ubuntu-sleeper-pod
spec:
    containers:
        - name: ubuntu-sleeper
          image: ubuntu-sleeper
          ports:
            - containerPort: 8080
        env:
            - name: APP_COLOR
              value: pink
            - name: APP_NAME
              valueFrom:
                configMapKeyRef:
            - name: APP_ENV
              valueFrom:
                secretKeyRef:
```

## ConfigMaps

Config Maps are used to pass configuration data to pods when created

```bash
kubectl create configmap \
<config-name> --from-literal=<key>=<value>
```

```yaml
apiVersion: v1
kind: configMap
metadata:
    name: app-config
data:
    APP_COLOR: blue
    APP_MODE: prod
```

to view configMaps

```
kubectl get configmaps
kubectl describe configmaps
```

### ConfigMaps in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: simple-webapp-color
spec:
    containers:
        - name: simple-webapp-color
          image: simple-webapp-color
          ports:
            - containerPort: 8080
            envFrom:
                - configMapRef:
                    name: app-config
```

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: simple-webapp-color
spec:
    containers:
        - name: simple-webapp-color
          image: simple-webapp-color
          ports:
            - containerPort: 8080
        env:    
            - name: APP_COLOR
              valueFrom:
                configMapKeyRef:
                    name: app-config
                    key: APP_COLOR

```

## K8s Secrets

```bash
kubectl create secret generic \
<secret-name> --from-literal=<key>=<value>
```

```YAML
aviVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: mysql
  DB_USER: root
  DB_PASSWORD: pswd1234
```

### Configuring Secrets with Pods

ENV

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      envFrom:
        - secretRef:
            name: app-secret
```
SINGLE ENV

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
```

VOLUME

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      volumes:
        - name: app-secret-volume
          secret:
              secretName: app-secret
```
### Things to keep in mind

- Secrets are not encrypted they are encoded
    - Don't check-in Secret object to SCM along with code
- Secrets are not encrypted in ETCD
    - Enable Encryption at rest
- Anyone is able to create pods/deployments in the same namespace can access the secrets
    - Configure least-privilege access to Secrets - RBAC
- Consider third party secret store providers AWS Provider, Azure Provider, GCP Provider, Vault Provider

## Multi-container Pods

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
    - name: log-agent
      image: log-agent
```