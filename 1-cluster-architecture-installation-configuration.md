# Cluster Architecture, Installation & Configuration (25%)

## Table of contents
1. [Manage role based access control (RBAC)](#manage-role-based-access-control-rbac)
1. [Bonus: How to trobleshoot permission problems](#bonus-how-to-trobleshoot-permission-problems)
1. [Use Kubeadm to install a basic cluster](#use-kubeadm-to-install-a-basic-cluster)
1. [Manage a highly-available Kubernetes cluster](#manage-a-highly-available-kubernetes-cluster)
1. [Provision underlying infrastructure to deploy a Kubernetes cluster](#provision-underlying-infrastructure-to-deploy-a-kubernetes-cluster)
1. [Implement etcd backup and restore](#implement-etcd-backup-and-restore)

## Manage role based access control (RBAC)
Reference: 
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-example
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#clusterrole-example

<details>
<summary>Solution</summary>
Role-based access (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within your organization.

It allows fine grain access permissions to users, groups or service accounts within your cluster.

`Role` and `ClusterRole` contains rules that represents a set of permissions (e.g. get a list of pods of a namamespace, or get a list all the nodes of a cluster)

`RoleBinding` and `ClusterRoleBinding` associates permissions defined in a Role/ClusterRole to a list of subjects (users, groups or service accounts).

### Create a Role and associate it with a RoleBinding
1. Create a namespace to be used in the Role and RoleBinding
```
kubectl create namespace development
```

1. Create a `Role` and `RoleBinding` to a specific namespace 
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "tiago" to read pods in the "development" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  namespace: development
  name: read-pods
subjects:
# You can specify more than one "subject"
- kind: User
  name: tiago # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

1. It's also possible to execute the same command above in an imperative way
```
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods

kubectl create rolebinding read-pods --user=tiago --role=pod-reader --namespace=development
```

### Create a ClusterRole and associate it with a ClusterRoleBinding
1. Create the `ClusterRole` and `ClusterRoleBinding` to a cluster-wide resource
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: nodes-reader
rules:
- apiGroups: [""]
  # at the HTTP level, the name of the resource for accessing Node
  # objects is "nodes"
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nodes-reader
subjects:
- kind: User
  name: tiago
roleRef:
  kind: ClusterRole
  name: nodes-reader
  apiGroup: rbac.authorization.k8s.io
```

1. It's also possible to execute the same command above in an imperative way
```
kubectl create clusterrole nodes-reader --verb=get --verb=list --verb=watch --resource=nodes

kubectl create clusterrolebinding nodes-reader --user=tiago --clusterrole=nodes-reader
```

</details>

## Bonus: How to trobleshoot permission problems
Reference: 
- https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access
- https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation

<details>
<summary>Solution</summary>

When requiring to troubleshoot permission issues with User/Group/ServiceAccount, it's possible to use the following `kubectl auth can-i` command to help you:
```bash
kubectl auth can-i <VERB> <RESOURCE> --as=<USER/SERVICEACCOUNT>

# Example
kubectl auth can-i list pods --as=tiago --namespace=development
# Should retun: yes

kubectl auth can-i list pods --as=john --namespace=development
# Should retun: no
```

You can list all permissions a user might have:
```bash
# List specific namespace permissions for a user
kubectl auth can-i --list --as=tiago --namespace=development

# List specific namespace permissions for a user
kubectl auth can-i --list --as=tiago --namespace=development
```

Cluster administrators also can run `kubectl` commands impersonating other users using parameter `--as=<USER/SERVICEACCOUNT>`:
```bash
# Should return list of pods
kubectl get pods --namespace development --as=tiago

# Should return an error
kubectl get pods --namespace development --as=john
# Output: Error from server (Forbidden): pods is forbidden: User "john" cannot list resource "pods" in API group "" in the namespace "development"
```
</details>

## Use Kubeadm to install a basic cluster
Reference:
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- https://v1-22.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

<details>
<summary>Solution</summary>

For creating a cluster using `kubeadm`, you basically have to use the reference material, first installing a container runtime, then installing and configuring the other components: `kubeadm`, `kubelet` and `kubectl` (optional) 

The following steps are an summary of the steps described on the reference section.

### (Control/Worker) Disable swap
```bash
# Disable swap immediately
sudo swapoff -a

# Open /etc/fstab with vim and comment swap
sudo vim /etc/fstab
## comment line by adding a # in front of line
## /swap.img       none    swap    sw      0       0
## #/swap.img       none    swap    sw      0       0
## :wq to save and quit
```

### (Control/Worker) Install container runtime: `containerd`
- Apply common settings for Kubernetes nodes on Linux
```bash
# Load `overlay` and `bt_netfilter` Linux modules during startup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load them immediately 
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

- Install `containerd` container runtime:
```bash
# Install `containerd` using package manager
sudo apt-get update && sudo apt-get install -y containerd

# Create containerd configuration folder
sudo mkdir -p /etc/containerd

# Generate default containerd configuration and save to the newly created default file
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Restart containerd to ensure new configuration file usage
sudo systemctl restart containerd
```

### (Control/Worker) Install Kubernetes components
```bash
# Update the apt package index and install packages needed to use the Kubernetes apt repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download the Google Cloud public signing key
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# Add the Kubernetes apt repository
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
sudo apt-get update
# You can install a version specific by adding the version on every component
# sudo apt-get install -y kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Control plane setup (single node)
1. Initialize the cluster
```bash
# Because we are going to use flannel as CNI, using the default pod CIDR from flannel
# See: https://github.com/flannel-io/flannel#deploying-flannel-manually
sudo kubeadm init --pod-network-cidr 10.244.0.0/16
```

1. Configure `kubectl` on control plane node:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Worker node setup
1. The control plane node `kubeadm init` should have created a `kubeadm join` command that contains the credentials to add a worker node. If required, you can recreate the token, as it has a expiration date on it.
```bash
kubeadm token create --print-join-command
```

1. Use the output from the previous command to join worker nodes into the cluster:
```bash
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
</details>

## Manage a highly-available Kubernetes cluster
Reference:
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

<details>
<summary>Solution</summary>

If you want to provision a highly available Kubernetes cluster, there are few considerations to make before you preceed, as described on the kubernetes.io reference above (for example: stacked etcd, external etcd, load balancer)

The following steps are using a stacked etcd configuration, and using another instance as load balancer running NGINX

This tutorial uses the following configuration:
- 1x load balancer instance
- 3x control plane instances
- 2x worker instances

Configure the control plane and worker instances using steps from the previous tutorial:
1. [Disable swap](#controlworker-disable-swap)
1. [Install container runtime: `containerd`](#controlworker-install-container-runtime-containerd)
1. [Install Kubernetes components](#controlworker-install-kubernetes-components)

### Setup load balancer
```bash
# Install nginx using package manager
sudo apt-get install -y nginx

# Enable nginx daemon
sudo systemctl enable nginx

# Create folder for the load balancer configuration
sudo mkdir -p /etc/nginx/tcpconf.d

# Edit nginx configuration to include our custom configuration
sudo vi /etc/nginx/nginx.conf

# Add the following to the end of nginx.conf:
# include /etc/nginx/tcpconf.d/*;

# Set up some environment variables for the lead balancer config file:
# The commands below is considering the control planes IP address, 
# but it can be configured to use hostname if you have it configured on your DNS
CONTROL_PLANE_1=<controller 1 private ip/DNS adress>
CONTROL_PLANE_2=<controller 2 private ip/DNS adress>
CONTROL_PLANE_3=<controller 3 private ip/DNS adress>

# Create the load balancer nginx config file:
cat << EOF | sudo tee /etc/nginx/tcpconf.d/kubernetes.conf
stream {
    upstream kubernetes {
        server $CONTROL_PLANE_1:6443;
        server $CONTROL_PLANE_2:6443;
        server $CONTROL_PLANE_3:6443;
    }

    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}
EOF

# Reload the nginx configuration
sudo nginx -s reload

# You can verify that the load balancer is working like so (once control planes configured):
curl -k https://localhost:6443/version
```

### Control plane setup (multi node)
Run the following command on one of the control plane nodes:
```bash
# Replace LOAD_BALANCER_DNS with the DNS address (or an IP address) from the instance configured on the last step
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:6443" --upload-certs
```

The output should look similar to:
```
You can now join any number of control-plane node by running the following command on each as a root:
    kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

Use the printed commands to join any control plane or worker node into the cluster.
</details>


## Provision underlying infrastructure to deploy a Kubernetes cluster
Reference:
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## Perform a version upgrade on a Kubernetes cluster using `kubeadm`
Reference:
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

<details>
<summary>Solution</summary>

For upgrading a cluster, you should use the reference material and follow the steps. 

The following steps are based on a highly available cluster described on the previous objective.

### (Control) For the first control plane upgrade
```bash
# replace x in 1.23.x-00 with the latest patch version
sudo apt-mark unhold kubeadm && \
sudo apt-get update && \
sudo apt-get install -y kubeadm=1.23.6-00 && \
sudo apt-mark hold kubeadm

# verify the upgrade plan
sudo kubeadm upgrade plan

# upgrade the node
sudo kubeadm upgrade node
```

### (Control) For all other control plane nodes
```bash
# replace x in 1.23.x-00 with the latest patch version
sudo apt-mark unhold kubeadm && \
sudo apt-get update && \
sudo apt-get install -y kubeadm=1.23.6-00 && \
sudo apt-mark hold kubeadm

# verify the upgrade plan
sudo kubeadm upgrade plan

# upgrade to the selected version
sudo kubeadm upgrade apply v1.23.6
```

### (Control) Uogade `kubelet` and `kubectl`
- Drain the node to start the upgrade of `kubelet` and `kubectl`
```bash
# replace <node-to-drain> with the name of your node you are draining
kubectl drain <node-to-drain> --ignore-daemonsets

# upgrade the components
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && \
sudo apt-get install -y kubelet=1.23.6-00 kubectl=1.23.6-00 && \
sudo apt-mark hold kubelet kubectl

# restart `kubelet`
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# uncordon the node
kubectl uncordon <node-to-drain>
```

### (Worker) Worker node upgrade

</details>

## Implement etcd backup and restore
Reference:

<details>
<summary>Solution</summary>


</details>