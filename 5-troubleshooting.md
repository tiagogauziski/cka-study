# Troubleshooting (30%)

## Table of contents
1. [Evaluate cluster and node logging](#evaluate-cluster-and-node-logging)
1. [Understand how to monitor applications](#understand-how-to-monitor-applications)
1. [Manage container stdout & stderr logs](#manage-container-stdout--stderr-logs)
1. [Troubleshoot application failure](#troubleshoot-application-failure)
1. [Troubleshoot cluster component failure](#troubleshoot-cluster-component-failure)

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
- https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/
- https://github.com/kubernetes-sigs/metrics-server

<details>
<summary>Solution</summary>

Monitoring is a broad topic, so we will cover in two different steps:
- Monitoring application health
- Metrics

### Monitoring application health

Application monitoring can be done with `livenessProbes`, `readinessProbes` and `startupProbes`:
- `livenessProbe`  
Liveness probes allow you to customise the default detection mechanism and make it more sophisticated.    
By default, Kubernetes will only consider a container to "down" and apply the restart policy if the container process stops.
> By default, Kubernetes will decide whether to restart the container based on the status of container's PID 1 process.  
> The first process to run on a container assumes PID 1. 

- `readinessProbe`
Indicates whether the container is ready to respond to requests. If the readiness probe fails, the Endpoint controller (related to Services) removes the Pod's IP address from the endpoints of all Services that match the Pod.
The default state of readiness before the initial delay is `Failure`. If a container does not provide a readiness probe, the default state is `Success`.

- `startupProbe`
Indicates whether the aplication within the container is started. All other probes are disabled if a startup probe is provided, until it succeeds. If the startup probe fails, the kubelet kill the container, and the container is subjected to it's restart policy. 
> Similar to `livenessProbe`, however, while liveness probe run constantly, startup probes run at the container startup and stop running once it succeed.  
> Useful for legacy applications with long startup times.

> Note: check the hands-on steps on the previous topics:
> [Investigate the default `livenessProbe` behavior](2-workloads-scheduling.md#investigate-the-default-livenessprobe-behavior)
> [Configure a `livenessProbe` with `exec` command](2-workloads-scheduling.md#configure-a-livenessprobe-with-exec-command)
> [Configure a `startupProbe` with `exec` command](2-workloads-scheduling.md#configure-a-startupprobe-with-exec-command)
> [Configure a `readinessProbe` with `exec` command](2-workloads-scheduling.md#configure-a-readinessprobe-with-exec-command)

### Metrics

There are two ways of setup metrics on Kubernetes:
 - Use `metrics-server`: it provides a lightweight, short-term, in-memory way of collecting CPU and memory metrics from Pods and Nodes to be used for scaling or resource limits enforcement.
 - Use Prometheus to monitor Kubernetes resource to provide a full metrics pipeline, and it you access to richer metrics.

 This example will explore the use of `metrics-server` for quick access of the cluster metrics.

 - Install `metrics-server` by running applying the following YAML manifest:
 ```bash
# One of the requirements for metrics server is the following:
# Kubelet certificate needs to be signed by cluster Certificate Authority (or disable certificate validation by passing --kubelet-insecure-tls to Metrics Server)
# So, in order to make it work on our cluster, we need to add `--kubelet-insecure-tls` argument into the deployment resource.
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -O metrics-server.yaml

# Edit the file and include the additional argument on the Deployment 
vim metrics-server.yaml
# containers:
#      - args:
#        - --cert-dir=/tmp
#        - --secure-port=4443
#        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
#        - --kubelet-use-node-status-port
#        - --metric-resolution=15s
#        - --kubelet-insecure-tls
#        image: k8s.gcr.io/metrics-server/metrics-server:v0.6.2

# Apply the resources
kubectl apply -f metrics-server.yaml

# Get nodes metrics
kubectl top nodes

# NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# k8s-control   51m          2%     822Mi           21%
# k8s-worker1   12m          0%     487Mi           12%
# k8s-worker2   11m          0%     625Mi           16%
 ```
</details>


## Manage container stdout & stderr logs
Reference: 
- https://kubernetes.io/docs/concepts/cluster-administration/logging/
- https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-running-pods
- https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-deployments-and-services

<details>
<summary>Solution</summary>

The easiest way is to execute `kubectl logs` to access pods or deployments logs.

### Access `Pod` logs using `kubectl logs`
To access container stdout and stderr logs, you can use `kubectl logs` command:
```bash
# Lets create a nginx pod
kubectl run nginx --image=nginx

# Get stdout logs from pod (single container)
kubectl logs pod/nginx

# Delete the pod
kubectl delete pod nginx
```

It's also possible to access logs from different container inside the same pod (multi container scenario):
```bash
# Create a pod with multiple containers
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-counter
spec:
  containers:
  - name: counter-1
    image: busybox:1.28
    args: [/bin/sh, -c,
            'i=0; while true; do echo "(counter-1) \$i: \$(date)"; i=\$((i+1)); sleep 10; done']
  - name: counter-2
    image: busybox:1.28
    args: [/bin/sh, -c,
            'i=0; while true; do echo "(counter-2) \$i: \$(date)"; i=\$((i+1)); sleep 10; done']
EOF

# Get container logs from container `counter-1`
kubectl logs pod/multi-container-counter --container counter-1

# Output:
# (counter-1) 0: Sun Apr 16 04:23:29 UTC 2023
# (counter-1) 1: Sun Apr 16 04:23:39 UTC 2023
# (counter-1) 2: Sun Apr 16 04:23:49 UTC 2023

# Get container logs from container `counter-2`
kubectl logs pod/multi-container-counter --container counter-2

# Output
# (counter-2) 0: Sun Apr 16 04:23:29 UTC 2023
# (counter-2) 1: Sun Apr 16 04:23:39 UTC 2023
# (counter-2) 2: Sun Apr 16 04:23:49 UTC 2023

# Delete the deployment
kubectl delete pod multi-container-counter
```

### Access `Deployment` logs using `kubectl logs`

```bash
# Create a deployment with multiple containers
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container-logs-deployment
spec:
  replicas: 3
  selector:
    matchLabels: 
      app: multi-container-logs
  template:
    metadata:
      labels:
        app: multi-container-logs
    spec:
      containers:
      - name: counter-1
        image: busybox:1.28
        args: [/bin/sh, -c,
                'i=0; while true; do echo "(\$POD_NAME) (counter-1) \$i: \$(date)"; i=\$((i+1)); sleep 10; done']
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
      - name: counter-2
        image: busybox:1.28
        args: [/bin/sh, -c,
                'i=0; while true; do echo "(\$POD_NAME) (counter-2) \$i: \$(date)"; i=\$((i+1)); sleep 10; done']
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
EOF

# Get all logs from all containers running under the deployment (using label selector)
kubectl logs -l app=multi-container-logs --all-containers

# Output
# (multi-container-logs-deployment-754fbf6dd9-2dr7n) (counter-1) 0: Sun Apr 16 05:05:41 UTC 2023
# (multi-container-logs-deployment-754fbf6dd9-2dr7n) (counter-2) 0: Sun Apr 16 05:05:42 UTC 2023
# (multi-container-logs-deployment-754fbf6dd9-92z9w) (counter-1) 0: Sun Apr 16 05:05:41 UTC 2023
# (multi-container-logs-deployment-754fbf6dd9-92z9w) (counter-2) 0: Sun Apr 16 05:05:42 UTC 2023
# (multi-container-logs-deployment-754fbf6dd9-9qhsd) (counter-1) 0: Sun Apr 16 05:05:41 UTC 2023
# (multi-container-logs-deployment-754fbf6dd9-9qhsd) (counter-2) 0: Sun Apr 16 05:05:42 UTC 2023

# Delete the deployment
kubectl delete deployment multi-container-logs-deployment
```

</details>


## Troubleshoot application failure
Reference:
- https://kubernetes.io/docs/tasks/debug/debug-application/
- https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/
- https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/
<details>

### Debuging Pods
The first step in troubleshooting is triage:
- What is the problem?
- Is your Pods running?
- What is the Pod state? Is it crashing?
- Is there any relevant logs? 

#### My pod stays pending
If a Pod is stuck in `Pending` state, it means it cannot be scheduled onto a node. Generally this is because there are insufficient resources that is preventing scheduling.
Check the output from `kubectl describe pod ...` for more information.
Common reasons for `Pending`
- **You don't have enough resources**: There is not enough Memory or CPU available on your cluster. You can adjust the resource requests or add a new node to your cluster.
- **You are using hostPort**: When you bind a Pod to a hostPort, there are limited number of places that the Pod can be scheduled. You should use `Services` to expose the Pod's ports. If you do require hostPort then you are limited by the number of nodes on your cluster.

```bash
# A pod requesting too much resources:
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: greedy-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 100
EOF

# Lets check the pod status
kubectl get pod

# Output:
# NAME         READY   STATUS    RESTARTS   AGE
# greedy-pod   0/1     Pending   0          8s

kubectl describe pod greedy-pod

# Output:
# Name:             greedy-pod
# Namespace:        default
# Priority:         0
# Service Account:  default
# Node:             <none>
# Labels:           <none>
# Annotations:      <none>
# Status:           Pending
# ...
# Events:
#  Type     Reason            Age   From               Message
#  ----     ------            ----  ----               -------
#  Warning  FailedScheduling  48s   default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient cpu. preemption: 0/3 nodes are available: # 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod..

# To get hostPort "Pending" status error, we will need to create a Deployment with more replicas than Pods in our cluster
# In this example, we have 2 worker nodes, so we need 3 replicas
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostportdeployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hostport
  template:
    metadata:
      labels:
        app: hostport
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - name: http
          hostPort: 8080
          containerPort: 80
EOF

# Lets query all Pods
kubectl get pods -o wide

# Output
# NAME                                  READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
# hostportdeployment-5ff54468c5-j4f5g   0/1     Pending   0          18s   <none>         <none>        <none>           <none>
# hostportdeployment-5ff54468c5-mv67q   1/1     Running   0          18s   10.244.2.166   k8s-worker2   <none>           <none>
# hostportdeployment-5ff54468c5-xgnfc   1/1     Running   0          18s   10.244.1.72    k8s-worker1   <none>           <none>

# Let's check why one of the Pods is marked as "Pending"
kubectl describe pod hostportdeployment-5ff54468c5-j4f5gH

# Output
# Name:             hostportdeployment-5ff54468c5-j4f5g
# Namespace:        default
# Priority:         0
# Service Account:  default
# Node:             <none>
# Labels:           app=hostport
#                   pod-template-hash=5ff54468c5
# Annotations:      <none>
# Status:           Pending
# ...
# Events:
#   Type     Reason            Age    From               Message
#   ----     ------            ----   ----               -------
#   Warning  FailedScheduling  4m30s  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't have free ports for the requested pod ports. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod..
```

#### My pod is running but not working correctly
```bash
# Lets create a Pod with a incorrect command (sheep does not exist)
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: incorrect-command-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'echo "test"; sheep 10']
EOF

# Check Pod status
kubectl get pod -o wide

# Output
# NAME                    READY   STATUS             RESTARTS      AGE   IP            NODE          NOMINATED NODE   READINESS GATES
# incorrect-command-pod   0/1     CrashLoopBackOff   3 (24s ago)   83s   10.244.1.74   k8s-worker1   <none>           <none>

# Check pod logs
kubectl logs incorrect-command-pod

# Output
# test
# sh: sheep: not found
```

#### Determine termination reason of a Pod
Termination messages provides way to store fatal error events to a location which could be retrieved by tools like kubectl or dashboards. By default, Kubernetes expects the termination logs to available on `/dev/termination-log` file.
```bash
# Create Pod on the namespace to test termination log messages
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: termination-log-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 10 && echo 'Sleep termination' > /dev/termination-log"]
EOF

# Get Pod
kubectl describe pod termination-log-pod


# Output 
# State without termination log message
# ...
#    State:          Terminated
#      Reason:       Completed
#      Exit Code:    0
#      Started:      Sun, 23 Apr 2023 07:41:26 +0000
#      Finished:     Sun, 23 Apr 2023 07:41:36 +0000

# State with termination log message
# ...
#    State:      Terminated
#      Reason:   Completed
#      Message:  Sleep termination
#
#      Exit Code:  0
#      Started:    Sun, 23 Apr 2023 07:45:10 +0000
#      Finished:   Sun, 23 Apr 2023 07:45:20 +0000
```

It's also possible to change the `terminationMessagePolicy` field on the container. The default value is set to `File` which means the message with be extracted from the file. By setting it to `FallbackToLogsOnError`, you tell Kubernetes to use the last chunk of container log output if the termination message file is empty and the container exited with an error.

#### Dedugging Pods with `kubectl exec` or `kubectl debug` commands
- If the container image contains debugging utilities, you can run `kubectl exec`: 
```bash
# Create simple nginx Pod
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Extract contents from index.html file
kubectl exec nginx-pod -- cat /usr/share/nginx/html/index.html

# Output
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# If we execute bash on remove, we get root privileges, and we can modify NGINX index page
kubectl exec --stdin --tty nginx-pod -- /bin/bash

# Execute:
# $ echo 'Change from kubectl exec' > /usr/share/nginx/html/index.html
# $ exit

# Let's check nginx serving page
kubectl get pod nginx-pod -o wide

# Output
# NAME        READY   STATUS    RESTARTS   AGE    IP             NODE          NOMINATED NODE   READINESS GATES
# nginx-pod   1/1     Running   0          109m   10.244.2.167   k8s-worker2   <none>           <none>

# cURL the Pod's IP
curl 10.244.2.167

# Output:
# Change from kubectl exec
```
- From Kubernetes 1.25 onwards, it's possible to use `kubectl debug` commands to create **ephemeral containers**, useful for situations when the main container is failing and the image does not include debugging utilities.
```bash
# Create a pod based on a pause container, which does not include any debugging utilities.
kubectl run ephemeral-demo --image=registry.k8s.io/pause:3.1 --restart=Never

# Try `kubectl exec` on the pod
kubectl exec -it ephemeral-demo -- sh

# Output
# error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "8feb76799c2cb1e2db635d7e087df0e7b3504ffe6176dfbfc125e863b58953da": OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH: unknown

# Create ephemeral container to debug main container
kubectl debug -it ephemeral-demo --image=busybox:1.28 --target=ephemeral-demo

# It's possible to see all process being run on target container
# Execute:
$ ps

# Output
# PID   USER     TIME  COMMAND
#     1 root      0:00 /pause
#    14 root      0:00 sh
#    20 root      0:00 ps

```
- It's possible to use `kubectl debug` to troubleshoot nodes as well:
```bash
kubectl debug node/k8s-worker1 -it --image=ubuntu

# The host filesystem will be mapped to /host path
ls /host

# The container also share the same network namespace 
apt update && apt install net-tools -y

# List all network interfaces
ifconfig
```

</details>

## Troubleshoot cluster component failure
Reference:
- https://kubernetes.io/docs/tasks/debug/debug-cluster/
<details>

</details>