# Scheduling

### Manual Scheduling

How does a scheduler works?

- Every pod has a field called nodeName which is by default is not set, the scheduler goes through all nodes and looks for the nodes that do not have the nodeName field set, those are the candidate for the scheduler, it then identifies the right node for the pod by running the scheduling algorithm, once identified, it schedules the pod on the node by setting the nodeName property to the name of the node by creating a binding object.

If there is no scheduler what can you do?

- The pod created won't be automatically assigned to a node, but you can always set the nodeName field inside the pod definition, the pod then get assigned to the specified node.
- You can only specify the nodeName at creation time, another way to assign nodeName to an existing pod is to create a binding object and send a Post request to the pod's binding API with the data set to the binding object object in a JSON format, so you must convert the YAML file to JSON

```YAML
apiVersion: v1
kind: Binding
metadata:
    name: nginx
target:
    apiVersion: v1
    kind: node
    name: node02
```
```bash
curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1","kind":"Binding","metadata":{"name":"nginx"},"target":{"apiVersion":"v1","kind":"node","name":"node02"}} http://$SERVER/api/v1/namespace/default/pods/$PODNAME/binding/
```

### Labels and Selectors

It's a standard method to group things together.

Labels; properties attached to items (class, color).
Selectors; help you filter these items.

#### How are labels and selectors used in K8s?

- You can group objects by their type (Pods, Deployments, Services, ReplicaSets)
- By Application (App1, App2, App3)
- By functionality (Frontend, Backend, Caching, DB, Auth)

#### How do you specify labels in K8s?

- inside metadata create a section called labels under that add as many labels as you like

- after creating you object you can select you object using this kubectl command

```bash
kubectl get pods --selector app=App1
```

This is one use but K8s use selectors and labels internally to connect different objects together.
- Like in replica sets we use selector to connect the replicaset to the pod to match the labels we find on the pod.

### Annotations

It's used for recording details for informatory purpose like emails, build versions, contact details, etc

### Taints and Tolerations

it's used to set restrictions on what pods can be scheduled on a node.

```bash
kubectl taint nodes node-name key=value:taint-effect
kubectl taint nodes node1 key=blue:NoSchedule
```

```YAML
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
    tolerations:
        - key: "app"
          operator: "Equal"
          value: "blue"
          effect: "NoSchedule"
```
\*\* __*All these values should be in double quotes*__ \*\*

NoExecute taint effect: it evicts non tolerant nodes out

### Node Selectors

You have 3 nodes, 2 are smaller with low hardware resources and one is larger with higher resources, you have different kinds of workloads running in your cluster, you would like to dedicate high resource workloads to the larger node as that is the only node that will not run out of resources. to solve this you can set limitation on pods so that they only run on particular nodes, there are 2 ways to do this:

- You can use node selectors, where you define the pod file as follows:
```YAML
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
spec:
    containers:
        - name: data-processor
          image: data-processor
    nodeSelector:
        size: Large
```
- To label the node you can use this command:

```bash
kubectl label nodes <node-name> <label-key>=<label-value>
```

- nodeSelector solved our problem but there are limitations

- for example we would like to place the pod on any node that is not large or place it on medium or small nodes, the other option is node affinity

### Node Affinity

The primary feature of node affinity is to ensure that pods are hosted on specific nodes

- We can't provide advanced expression with nodeSelector (and, or, not)


```YAML
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
spec:
    containers:
        - name: data-processor
          image: data-processor
    affinity:
        nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            #preferredDuringSchedulingIgnoredDuringExecution
            #requiredDuringSchedulingRequiredDuringExecution (planned)
                nodeSelectorTerms:
                    - matchExpressions:
                        - key: size
                          operator: In #NotIn #Exists
                          values:
                            - Large
                            - Medium
```

### Difference between Taints and Tolerations vs nodeAffinity

-  a combination of taints and tolerations is used to ensure that no node will have any other pods that desired (limitation with node affinity) or that pods will end up on node with no taints (limitation with taints and tolerations)

### Resource requirements

- scheduler decided which node should the pod be placed on to, the scheduler takes into consideration the amount of resource required by a pod and those available on a node.

- if nodes have no sufficient resources available, the scheduler avoids placing pods on these nodes, and instead places the pods on nodes with sufficient resources, if there is sufficient resources available on any on the nodes, then the scheduler avoids scheduling theses pods and you will see the pods in pending state, and if you see events using kubectl describe pods command you will see insufficient resources (CPU, Memory).

#### Resource Request

- you can specify the resources used by a pod by specifying the CPU and Memory and this is known as resource request for a container, so the minimum amount of cpu and memory requested by the container.

- to do this in the pod definition YAML

```YAML
apiVersion: v1
kind: Pod
metadata:
    name: simple-webapp
    labels:
        name: simple-webapp
spec:
    containers:
        - name: simple-webapp
          image: simple-webapp
          ports:
              - containerPort: 8080
          resources:
              requests:
                  memory: "4Gi" # 1 Gi -> 1024 Mi -> 1024 Ki
                              # 1 G -> 1000 M -> 1000 -> 1000 K
                  cpu: 2 # 1 cpu -> 1000m
              limits:
                  memory: "4Gi"
                  cpu: 4
```
- a container can use more memory resources than its limit, so if the pod tries to consume more memory constantly then the pod will be terminated with OOM (out of memory) error.

- by default K8s does not have CPU or Memory request or limit set, so every pod can consume as much resources from the node and suffocate other pods in the node.

#### How CPU requests and limits work?

- lets say there are 2 pods competing for cpu on the cluster, when I say pod I mean a container within a pod, without a resource or a limit set, one pod can consume all the resources on a node depriving the other pod from resources.

- the ideal scenario is to set requests but not limits to allow pods, to consume resources when available.

- same goes for memory, but we can not throttle memory so the only way to free memory is to kill the pod consuming it

#### LimitRange

```YAML
apiVersion: v1
kind: LimitRange
metadata:
    name: resource-constraint
spec:
    limits:
        - default: simple-webapp
              cpu: 500m
          defaultRequest:
              cpu: 500m
          max:
              cpu: "1"
          min:
              cpu: 100m
          type: Container
```

- Limit Range only affects pods when created, it will not affect existing pods

#### Resource Quotas

- it's a namespace level objects that is used to set hard limits for requests and limits.

```YAML
apiVersion: v1
kind: ResourceQuota
metadata:
    name: my-resource-quota
spec:
    hard:
        requests.cpu: 4
        requests.memory: 4Gi
        limits.cpu: 10
        limits.memory: 10Gi
```

### Daemon Sets

- Daemon sets are like replicaSets as in it helps you deploy multiple instances of of pods but it runs one copy of your pod in each node of the cluster, whenever a new node is added to the cluster a replica of this pod is automatically added to this node and when the node is remove the pods is automatically removed

- use cases: deploying log routers and monitoring agents, kube-proxy, networking like weave-net

- DaemonSet Definition

```YAML
apiVersion: apps/v1
kind: DaemonSet
metadata:
    name: monitoring-daemon
spec:
    selector:
        matchLabels:
            app: monitoring-agent
    template:
        metadata:
            labels:
                app: monitoring-agent
        spec:
            containers:
                - name: monitoring-agent
                  image: newrelic/nri-kubernetes
```
```bash
kubectl get daemonsets
kubectl describe daemonset monitoring-daemen
```

#### How does it work?

- before 1.12, one each pod set the nodeName property before creating in the pod specification and when created the land on the respective node.

- from 1.12 and onwards, the daemon set uses the default scheduler and the node affinity rules to schedule pods on nodes

### Static Pods

- is there anything that a kubelet can do if it was the only single node with no cluster with no any other node?

- you can configure the kubelet to read the pod definition files from a dir on the server designated to store info about pods ("/etc/kubernetes/manifests"), the kubelets periodically reads these files and create the pods on the host, not only it creates the pod but it ensures that it stays alive. if the app crashed the kubelet restarts it, if you make a change to the files, kubelet recreate the pods with the changes. if you remove a file from this directory, kubelet deletes the pod.

- these pods are known as static pods, you can only create pods that way you can not create replicasets, deployments or service by using this method

- the dir that kubelet runs pods from is configured through --pod-manifest-path in the kubelet.service

- you can pass a path to file and in that file configure staticPodPath: /etc/kubernetes/manifest

```YAML
staticPodPath: /etc/kubernetes/manifest
```


- you can see the pods running using the ```docker ps``` command

### Multiple Schedulers

- You can have multiple scheduler at a time, you can have your own custom scheduler.

- when you have multiple scheduler each one must have a unique name.
- to deploy the scheduler you must download the k8s scheduler binary
- then  inside the scheduler service  we point the configuration point to the configuration yaml file 
```YAML
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
    - schedulerName: my-scheduler-1
```

- You can also deploy a custom scheduler as a pod

*my-custom-scheduler.yaml*
```YAML
apiVersion: v1
kind: Pod
metadata:
    name: my-custom-scheduler
    namespace: kube-system
spec:
    containers:
        - command:
              - kube-scheduler
                --address=127.0.0.1
                --kubeconfig=/etc/kubernetes/scheduler.conf
                --config=/etc/kubernetes/my-scheduler-config.yaml
          image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
          name: kube-scheduler
```

*my-scheduler-config.yaml*
```YAML
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
    - schedulerName: my-scheduler-1
leaderElection:
    leaderElect: true # used when you have multiple copies of the scheduler running on different master nodes in a HA setup
    resourceNamespace: kube-system
    resourceName: lock-object-my-scheduler
```

- how to use you custom scheduler

```YAML
apiversion: v1
kind: pod
metadata:
    name: nginx
spec:
    containers:
        - image: nginx
          name: nginx
    schedulerName: my-custom-scheduler
```

- how to know which scheduler picked up my pod creation?

```bash
kubectl get events -o wide
kubectl logs my-custom-scheduler --name-space=kube-system
```

### Scheduling Profiles

- Scheduling has multiple steps to do before binding the pod to a node

- first the pod go through a scheduling queue (PrioritySort) -> queueSort
- then it goes through filtering the nodes (NodeResourcesFit, NodeName, NodeUnschedulable) -> preFilter, filter, postFilter
- then it goes to scoring the nodes (NodeResourcesFit, ImageLocality) -> preScore, score, reserve
- then lastly binding the pod to node (DefaultBinder) -> permit, preBind, bind, postBind

