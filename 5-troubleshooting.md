# Troubleshooting (30%)

## Table of contents
1. [Evaluate cluster and node logging](#evaluate-cluster-and-node-logging)

## Evaluate cluster and node logging
Reference: 
- https://kubernetes.io/docs/concepts/cluster-administration/logging/

<details>
<summary>Solution</summary>

In Kubernetes, cluster and node logging are available on different locations, dependending on how your cluster has been deployed.

```bash
# Use journalctl to access kubelet logs
sudo journalctl -u kubelet -n 10
sudo journalctl -u containerd -n 10

# In order to access Kubernetes components, lets get the name of the pods so we access their logs later
kubectl get pods -n kube-system

# Output:
# etcd-k8s-control                      1/1     Running   35 (33m ago)    153d
# kube-apiserver-k8s-control            1/1     Running   113 (33m ago)   153d
# kube-controller-manager-k8s-control   1/1     Running   91 (33m ago)    153d
# kube-scheduler-k8s-control            1/1     Running   91 (33m ago)    153d

# Now we have the pods name, we can access their logs:
kubectl logs -n kube-system etcd-k8s-control
kubectl logs -n kube-system kube-apiserver-k8s-control
kubectl logs -n kube-system kube-controller-manager-k8s-control
kubectl logs -n kube-system kube-scheduler-k8s-control
```

- Note that kubectl logs access `stdout` and `stderr` t

</details>