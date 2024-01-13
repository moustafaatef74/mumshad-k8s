# Logging & Monitoring

## Monitoring K8s Cluster Components

Monitoring

- Node Level Metrics (Number of nodes, how many of them are healhy, preformance, cpu memeory, network, disk utilization)
- Pod Level Metrics

K8s does not come equipped with with a full feautured monitoring solution but there are some open source solutions like (metric server, prometheus, elastic stack)

Heapster was one the original projects the enabled monitoring and analysis for k8s, however it's now depricated

Metrics Server is now in use, you can have one metric server per cluster, it's an in memory solution and does not store data on disk.

### How are the metrics collected?

- the Kubelet has a subcomponent named cadvisor which is responisble for retreiving metrics from pods and exposing them through the kubelet api to make the metrics available for the metrics server.

### How to install metrics server?

```bash
git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git

kubectl create -f ./kubernetes-metrics-server
```

## Application Logs

```bash
kubectl logs -f event-simulation-pod
kubectl logs -f event-simulation-pod event-simulator
```

