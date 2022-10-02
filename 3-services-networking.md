# Services & Networking (20%)

## Table of contents
1. [Understand host networking configuration on the cluster nodes](#understand-host-networking-configuration-on-the-cluster-nodes)
1. [Understand connectivity between Pods](#understand-connectivity-between-pods)
1. [Understand ClusterIP, NodePort, LoadBalancer service types and endpoints](#understand-clusterip-nodeport-loadbalancer-service-types-and-endpoints)
1. [Know how to use Ingress controllers and Ingress resources](#know-how-to-use-ingress-controllers-and-ingress-resources)
1. [Know how to configure and use CoreDNS](#know-how-to-configure-and-use-coredns)

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

### Service without `selector` and Endpoints

Services most commonly abstract access to Pod thanks to the `selector`, but when used with Endpoints object without a selector, the Service can abstract other kinds of backends, including the ones that run outside the cluster. For example:
- You want to have an external database in production, but in your test environment your have your own StatefulSet
- You are migrating a workload to Kubernetes. While evaluating the approach, you run only a portion of your backends in Kubernetes.

For any of these scenarios, you can use a Service **without** a selector.

> **Note**  
> The endpoint IPs must not be: loopback (127.0.0.0/8 for IPv4, ::1/128 for IPv6), or link-local (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6).
> Endpoint IP addresses cannot be the cluster IPs or other Kubernetes Services.

- For this example, we are going to point to a Pod IP address. 
```bash
kubectl get pods -o wide

# Example
# NAME             READY   STATUS    RESTARTS     AGE     IP           NODE          NOMINATED NODE   READINESS GATES
# nginx-nodeport   1/1     Running   1 (3d ago)   6d23h   10.244.1.6   k8s-worker1   <none>           <none>
```

- Create a Service without a selector and the corresponding Endpoint
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-service-without-selector
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  # the name here should match the name of the Service
  name: sample-service-without-selector
subsets:
  - addresses:
      - ip: 10.244.1.6 # You cannot use the cluster IP address instead of any Kubernetes internal cluster IP .
    ports:
      - port: 80
```

- Check to see the details of the service created:
```bash
kubectl get services

# Output:
# kubectl get services
# NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
# kubernetes                        ClusterIP   10.96.0.1        <none>        443/TCP          9d
# nginx-service                     NodePort    10.104.240.162   <none>        8080:31451/TCP   6d23h
# sample-service-without-selector   ClusterIP   10.102.237.244   <none>        80/TCP           2m20s
```

- We can check to see if we reach the Pod:
```bash
curl 10.102.237.244

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...
```

> **Note**  
> If you need to point another Service within the Cluster, consider using an `ExternalName` Service type.
> An `ExternalName` Service is a special case of Service that **does not have selectors** and uses DNS names instead. 
</details>

## Know how to use Ingress controllers and Ingress resources
Reference:
- https://kubernetes.io/docs/concepts/services-networking/ingress/

<details>
<summary>Solution</summary>

Ingress is a Kubernetes object to help with load balancing, SSL termination and name-based virtual hosting.

An IngressController is reponsible for fulfilling the Ingress, usually with a load balancer.

An Ingress does not expose arbitrary ports or protocols. Exposing services other than HTTP and HTTPS to the internet typically uses a Service of `type=NodePort` or `type=LoadBalancer`

### Install IngressController - NGINX IngressController

- Install the NGINX ingress controller using the following YAML definition:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```

- Get the installed IngressClass:
```bash
kubectl get ingressclass

# Output:
# NAME    CONTROLLER             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx   <none>       2d7h
```


### Create an Ingress resource

- Create a Namespace to hold all resources:
```bash
kubectl create namespace sample-ingress
```

- Create an Ingress:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-deployment
  namespace: sample-ingress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-app
        image: nginx
        ports:
        - containerPort: 80
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-service
  namespace: sample-ingress
spec:
  type: ClusterIP
  selector:
    app: hello-app
  ports:
  - name: service-http
    protocol: TCP
    port: 80
    targetPort: http # you can name the port that matches the Pod or use the port number (80)
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-ingress
  namespace: sample-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /hello-app
        pathType: Prefix
        backend:
          service:
            name: ingress-service
            port:
              number: 80
```

- The definition will create few resources:
  - Deployment with a NGINX image running on port 80
  - Service to expose the NGINX Pods internally (`type=ClusterIP`)
  - Ingress resource to redirect external calls from `<EXTERNAL>/hello-app` to the Service

- Let's check the if the Ingress is working as expected:
```bash
kubectl get all -n sample-ingress

# Output:
# kubectl get all -n sample-ingress
# NAME                                      READY   STATUS    RESTARTS   AGE
# pod/ingress-deployment-6c77df49d9-4t9ln   1/1     Running   0          10m
# pod/ingress-deployment-6c77df49d9-5k42c   1/1     Running   0          10m
# 
# NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
# service/ingress-service   ClusterIP   10.104.153.192   <none>        80/TCP    21m
# 
# NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/ingress-deployment   2/2     2            2           20m
# 
# NAME                                            DESIRED   CURRENT   READY   AGE
# replicaset.apps/ingress-deployment-6c77df49d9   2         2         2       10m
```

- Lets have a look at the Ingress created:
```bash
kubectl describe ingress sample-ingress -n sample-ingress

# Output: 
# Name:             sample-ingress
# Labels:           <none>
# Namespace:        sample-ingress
# Address:
# Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
# Rules:
#   Host        Path  Backends
#   ----        ----  --------
#   *
#               /hello-app   ingress-service:80 (10.244.1.11:80,10.244.2.32:80)
# Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
# Events:
#   Type    Reason  Age                From                      Message
#   ----    ------  ----               ----                      -------
#   Normal  Sync    23m (x2 over 23m)  nginx-ingress-controller  Scheduled for sync# 
```

- Because we are not running on a cloud provider, we don't have a LoadBalancer created. We need to get the IngressController service IP address:
```bash
kubectl get all -n ingress-nginx

# Output:
# NAME                                            READY   STATUS    RESTARTS       AGE
# pod/ingress-nginx-controller-54d587fbc6-fn484   1/1     Running   1 (108m ago)   2d8h
# 
# NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
# service/ingress-nginx-controller             LoadBalancer   10.109.12.86     <pending>     80:32284/TCP,443:31392/TCP   2d8h
# service/ingress-nginx-controller-admission   ClusterIP      10.101.107.213   <none>        443/TCP                      2d8h
# 
# NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/ingress-nginx-controller   1/1     1            1           2d8h
# 
# NAME                                                  DESIRED   CURRENT   READY   AGE
# replicaset.apps/ingress-nginx-controller-54d587fbc6   1         1         1       2d8h
```

> **Note**  
> The Service `ingress-nginx-controller` does not have a EXTERNAL-IP.  
> We can use the NodePorts to make requests to the IngressController:
> - HTTP: 80:**32284**/TCP
> - HTTPS: 443:**31392**/TCP

- If we make calls to the ports described, we should be able to reach the Pods:
```bash
# HTTPS request
curl https://localhost:31392/hello-app --insecure

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...

# HTTP request
curl http://localhost:32284/hello-app

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...
```

- If we try to use URI prefix that does not exist, the IngressController returns a HTTP 404:
```bash
curl http://localhost:32284/invalid

# Output:
# <html>
# <head><title>404 Not Found</title></head>
# <body>
# <center><h1>404 Not Found</h1></center>
# <hr><center>nginx</center>
# </body>
# </html>
```

### Name based virtual hosting

Name-based virtual hosts support routing HTTP traffic to multiple host names at the same IP address.

We can achieve that by specifying a `{.spec.rules[].host}` on a Ingress.

- Lets start by creating a namespace for our resources
```bash
kubectl create namespace ingress-host-routing
```

- Create an Ingress: (ingress-host-routing.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: first-deployment
  namespace: ingress-host-routing
spec:
  replicas: 2
  selector:
    matchLabels:
      app: first
  template:
    metadata:
      labels:
        app: first
    spec:
      containers:
      - name: nginx
        image: nginxdemos/hello
        ports:
        - containerPort: 80
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: first-service
  namespace: ingress-host-routing
spec:
  type: ClusterIP
  selector:
    app: first
  ports:
  - name: first-service-http
    protocol: TCP
    port: 80
    targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: second-deployment
  namespace: ingress-host-routing
spec:
  replicas: 2
  selector:
    matchLabels:
      app: second
  template:
    metadata:
      labels:
        app: second
    spec:
      containers:
      - name: nginx
        image: nginxdemos/hello
        ports:
        - containerPort: 80
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: second-service
  namespace: ingress-host-routing
spec:
  type: ClusterIP
  selector:
    app: second
  ports:
  - name: second-service-http
    protocol: TCP
    port: 80
    targetPort: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-host-routing-ingress
  namespace: ingress-host-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: first.127.0.0.1.nip.io # using the master node ip address and a DNS resolution workaround to get to it
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: first-service
            port:
              number: 80
  - host: second.127.0.0.1.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: second-service
            port:
              number: 80
```

- The above definition create multiple resources:
  - A Deployment and Service to route requests from `first.127.0.0.1.nip.io` host
  - A Deployment and Service to route requests from `second.127.0.0.1.nip.io` host
  - A Ingress resource to map all the hosts and configure the IngressController to route to the correct services based on the host name.
    
  
- If we call the IngressController CLUSTER-IP:NODE-PORT we can test the newly created Ingress resource:
```bash
# On any node of the custer:
curl http://first.127.0.0.1.nip.io:32284/

# Output:
# ...
# <p><span>Server&nbsp;address:</span> <span>10.244.1.15:80</span></p>
# <p><span>Server&nbsp;name:</span> <span>first-deployment-684495f4d8-dwbpk</span></p>
# ...

# If we call the second:
curl http://second.127.0.0.1.nip.io:32284/

# Output
# ...
# <p><span>Server&nbsp;address:</span> <span>10.244.2.37:80</span></p>
# <p><span>Server&nbsp;name:</span> <span>second-deployment-5fbb848954-xsvll</span></p>
# ...
```

</details>

## Know how to configure and use CoreDNS
Reference:
- https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

<details>
<summary>Solution</summary>

### How to get CoreDNS ConfigMap

- To get CoreDNS `ConfigMap`:
```bash
kubectl get configmap coredns -n kube-system -o yaml

# Output: 
# apiVersion: v1
# data:
#   Corefile: |
#     .:53 {
#         errors
#         health {
#            lameduck 5s
#         }
#         ready
#         kubernetes cluster.local in-addr.arpa ip6.arpa {
#            pods insecure
#            fallthrough in-addr.arpa ip6.arpa
#            ttl 30
#         }
#         prometheus :9153
#         forward . /etc/resolv.conf {
#            max_concurrent 1000
#         }
#         cache 30
#         loop
#         reload
#         loadbalance
#     }
# kind: ConfigMap
# metadata:
#   creationTimestamp: "2022-07-04T11:11:08Z"
#   name: coredns
#   namespace: kube-system
#   resourceVersion: "228"
#   uid: fa62b821-c958-4adb-bb13-8885f62d54d9
```

- To edit CoreDNS `ConfigMap`:
```bash
# It's possible to use kubectl edit
kubectl edit configmap coredns -n kube-system

# It's also possible to export it and apply the changes again
kubectl get configmap coredns -n kube-system -o yaml > coredns-configmap.yaml

kubectl apply -f coredns-configmap.yaml
```

### Configuration of Stub-domain and upstream nameserver using CoreDNS


</details>