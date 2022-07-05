# Services & Networking (20%)

## Table of contents
1. [Understand host networking configuration on the cluster nodes](#understand-host-networking-configuration-on-the-cluster-nodes)
1. [Understand connectivity between Pods](#understand-connectivity-between-pods)
1. [Understand ClusterIP, NodePort, LoadBalancer service types and endpoints](#understand-clusterip-nodeport-loadbalancer-service-types-and-endpoints)

## Understand host networking configuration on the cluster nodes
Reference: 
- https://kubernetes.io/docs/concepts/services-networking/
- https://kubernetes.io/docs/concepts/cluster-administration/networking/

<details>
<summary>Solution</summary>

At the heart of the Kubernetes networking model, is that `every Pod gets assigned an IP address`. This means that you don't need to create links between Pods yourself or have to deal with mapping containers ports to host ports (much like running Docker).
This model creates a clean, backwards-compatible model where Pods can be treated like VMs or physical hosts from the perspective of port allocation, naming, service discovery, load balancing, application configuration and migration. 

Kubernetes imposes the following fundamental requirements on any networking implementation:
- Every Pod gets an IP address.
- Pods can communicate with all other Pods on any other node without NAT.
- Pods running on the host network can still communicate with all other pods on any other node without NAT.
- agents on a node (system daemons, kubelet) can communicate with all Pods on that node.

The idea of having Pods with IP address enable a low-friction porting of apps running on VMs into containers.

</details>

## Understand connectivity between Pods
Reference:
- https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking

<details>
<summary>Solution</summary>

The IP address exists at the Pod scope: containers within a Pod share their network namespaces. This means that containers within a Pod can communicate with each other using `localhost`. It also means that containers with a Pod needs to coordinate port usage. 

### Create Pod with multiple containers and communicate using `localhost`

- Create Pod definition
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod-networking
spec:
  containers:
    - name: webserver
      image: nginx
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'while true; do wget -O - http://localhost; sleep 10; done']
```

</details>

## Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
Reference:
- https://kubernetes.io/docs/concepts/services-networking/service/

<details>
<summary>Solution</summary>

### Create a service

- Create a Pod and Service and bind them (sample-pod-service.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc
        
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

> Note that the `Pod` label `app.kubernetes.io/name` is used as `selector` on the `Service`.  
> This way a `Service` knows how to redirect requests to a group of `Pods` (like a `Deployment` or `StatefulSet`)

- Let's get the service details:
```bash
kubectl get services

# Output:
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP    22h
# nginx-service   ClusterIP   10.97.253.251   <none>        80/TCP     13m
```

- We can try using `curl` to hit the Pod behind the service. Use the Service CLUSTER-IP and PORT associated with the service.
```bash
curl 10.97.253.251:80

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...
```

### ClusterIP



</details>