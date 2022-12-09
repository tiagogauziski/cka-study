# Troubleshooting (30%)

## Table of contents
1. [Evaluate cluster and node logging](#evaluate-cluster-and-node-logging)
1. [Understand how to monitor applications](#understand-how-to-monitor-applications)

## Evaluate cluster and node logging
Reference: 
- https://kubernetes.io/docs/concepts/cluster-administration/logging/

<details>
<summary>Solution</summary>

In Kubernetes, cluster and node logging are available on different locations, dependending on how your cluster has been deployed.

```bash
# Use journalctl to access kubelet or containerd logs
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

- Note that kubectl logs access `stdout` and `stderr` through `/var/log` folder on control/worker nodes:
```bash
# List contents of log folder on control node
ls /var/log/pods/

# Output:
# kube-system_coredns-64897985d-87wzw_0906a726-9fc9-4b32-8e34-d61cf41ffef7/
# kube-system_coredns-64897985d-z9spm_259bcd2e-c513-4c43-83c0-4bd16114af85/
# kube-system_etcd-k8s-control_38e46b4483e181276898d979d6540151/
# kube-system_kube-apiserver-k8s-control_c711f6b4d98c1ffa7b9fcc5355b54483/
# kube-system_kube-controller-manager-k8s-control_a39a22e07bab370e11c1f2eabc082694/
# kube-system_kube-flannel-ds-fdq5r_d67970ff-7af8-43c9-9ce6-9181b6b2d08c/
# kube-system_kube-proxy-t4wt8_77135467-0d6f-4081-95c4-0f9e6cb9d082/
# kube-system_kube-scheduler-k8s-control_8e66e38685d6e7cabb6d343c447e597b/
```
</details>

## Understand how to monitor applications
Reference: 
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

<details>
<summary>Solution</summary>

</details>