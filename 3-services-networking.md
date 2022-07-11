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

### CLusterIP

- The default Service type.
- Expose the service on a cluster-internal IP. (The service will get an IP address - check it using `kubectl get services`)
- Not reachable from outside the cluster.

- Create a Pod and Service and bind them (sample-service-clusterip.yaml)
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

### NodePort

If the you set the Service `type` field to `NodePort`, Kubernetes will allocate a port from a range specified by `--service-node-port-range` flag on `/etc/kubernetes/manifests/kube-apiserver.yaml` (default: 30000-32767). 

When you create a Service as `NodePort`, the Service will report the allocated port in it's `.spec.ports[*].nodePort` field.
Each node on the cluster proxies that port into your service. 

- Create a `NodePort` Service
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nodeport
  labels:
    app.kubernetes.io/name: proxy-nodeport
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: proxy-nodeport
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 8080
    targetPort: 80
```

- Let's get the service details.
```bash
kubectl get services

# Output
# NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
# kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP          2d22h
# nginx-service   NodePort    10.104.240.162   <none>        8080:31451/TCP   5s

# Note the nodePort is set to 31451.
```

- If we make a call to multiple nodes of the cluster on the same port, it will forward the request to the service.
```bash
kubectl get nodes

# Output:
# NAME          STATUS   ROLES                  AGE     VERSION
# k8s-control   Ready    control-plane,master   2d22h   v1.23.5
# k8s-worker1   Ready    <none>                 2d22h   v1.23.5
# k8s-worker2   Ready    <none>                 2d22h   v1.23.5

# If we curl a node:
curl k8s-control:31451

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# If we curl another node:
curl k8s-worker1:31451

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...
```

### LoadBalancer

On a cloud provider that supports external load balancer (like AWS ELB/ALB/NLB or Azure Load Balancer/Application Gateway), the Service type `LoadBalancer` will provision a external load balancer. The creation of the external resource happens asynchronously and information about the provisioned balancer is published in the Service's `{.status.loadBalancer}` field. 

### ExternalName

Services of type `ExternalName` map a Service to a DNS name and does not use `selector` to redirect to a Pod. You specify these Services with `{.spec.externalName}` field.

- Create a `ExternalName` service (sample-service-externalname.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: development
spec:
  type: ExternalName
  externalName: example.com
```

- Lets check our Service
```bash
kubectl get service -n development

# Note: no CLUSTER-IP, only EXTERNAL-IP.
# Output:
# NAME         TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# my-service   ExternalName   <none>       example.com   <none>    17m
```

- The Service will create a CNAME DNS entry to redirect to `example.com`. We can check it by making some calls from within a Pod:
```bash
# Lets run a Pod and open a shell so we can run some commands
kubectl run -i --tty alpine --image=alpine --restart=Never --rm -- sh

# We will need curl to run test our Service
apk --update add curl

# We then call curl to the service. 
# Because the service will redirect to another URI, we need to add the `Host` header with the correct value. 
# We also need to skip TLS validation (--insecure) as we are calling one URI but tempering with the Host.
curl -H "Host: example.com" https://my-service.development.svc.cluster.local --insecure

# Output
# <!doctype html>
# <html>
# <head>
#     <title>Example Domain</title>
# ...
```

</details>