# Services & Networking (20%)

## Table of contents
1. [Understand host networking configuration on the cluster nodes](#understand-host-networking-configuration-on-the-cluster-nodes)

## Understand host networking configuration on the cluster nodes
Reference: 
- https://kubernetes.io/docs/concepts/services-networking/

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

The IP address exists at the Pod scope: containers within a Pod share their network namespaces. This means that containers within a Pod can communicate with each other using `localhost`. It also means that containers with a Pod needs to coordinate port usage. 

</details>