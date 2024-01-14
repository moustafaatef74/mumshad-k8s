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
