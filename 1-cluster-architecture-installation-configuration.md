# Cluster Architecture, Installation & Configuration (25%)

## Table of contents
1. [Bonus: Get started with kubectl config](#bonus-get-started-with-kubectl-config)
1. [Manage role based access control (RBAC)](#manage-role-based-access-control-rbac)
1. [Bonus: How to trobleshoot permission problems](#bonus-how-to-trobleshoot-permission-problems)
1. [Use Kubeadm to install a basic cluster](#use-kubeadm-to-install-a-basic-cluster)
1. [Manage a highly-available Kubernetes cluster](#manage-a-highly-available-kubernetes-cluster)
1. [Provision underlying infrastructure to deploy a Kubernetes cluster](#provision-underlying-infrastructure-to-deploy-a-kubernetes-cluster)
1. [Implement etcd backup and restore](#implement-etcd-backup-and-restore)

## Bonus: Get started with kubectl config
Reference: 
- https://kubernetes.io/docs/reference/access-authn-authz/authentication/

<details>

When interacting with Kubernetes clusters using `kubectl` tool, it is required to configure it first with information about the who is accessing the cluster (`kubectl config get-credentials`) and where is the cluster hosted (`kubectl config get-clusters`). Binding these two pieces for information together and you have a context (`kubectl config get-context`).

When provisioning a cluster with `kubeadm`, it generates a `.kube` file that contains a admin user, the recently created cluster and a default context. 

The following commands assumes there is a basic cluster running on the instance. The following commands creates a new user certificate and key to be used on a kubectl context
```bash
# Create a key
openssl genrsa -out tiago.key 2048

# Create user CSR
# Kubernetes uses what is inside Organization as user group information
# In this case, the user tiago belongs to "administrators" and "users" groups
openssl req -new -key tiago.key -out tiago.csr -subj "/CN=tiago/O=administrators/O=users"

# Approve the CSR
sudo openssl x509 -req -in tiago.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out tiago.crt -days 500

# Create kubectl credential
kubectl config set-credentials tiago --client-certificate=/home/tiago/tiago.crt --client-key=/home/iago/tiago.key
# Create kubectl context using the new user and the existing cluster
kubectl config set-context tiago@kubernetes --cluster=kubernetes --user=tiago

# Let's all available contexts
kubectl config get-contexts

# Output
# CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
# *         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
#           tiago@kubernetes              kubernetes   tiago

# Let's switch to the recently created context
kubectl config use-context tiago@kubernetes

# Try listing all pods (it will fail, as the user does not have any permissions assigned)
kubectl get pods

# Output
# Error from server (Forbidden): pods is forbidden: User "tiago" cannot list resource "pods" in API group "" in the namespace "default"
```

Once the new user and the contexts are created, we can proceed to the next section in order to assign permissions.

</details>

### Summary
```bash
kubectl config set-credentials tiago --client-certificate=/home/tiago/tiago.crt --client-key=/home/iago/tiago.key
kubectl config set-context tiago@kubernetes --cluster=kubernetes --user=tiago
kubectl config get-contexts
kubectl config use-context tiago@kubernetes
```

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
- If you want to define a role within a specific namespace, use `Role`. (Cannot be reused on another namespace)
- If you want to define a role and reuse it for multiple namespaces, or for a cluster-scoped resource, use `ClusterRole`.

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
```bash
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
```bash
kubectl create clusterrole nodes-reader --verb=get --verb=list --verb=watch --resource=nodes

kubectl create clusterrolebinding nodes-reader --user=tiago --clusterrole=nodes-reader
```

</details>

### Summary
```bash
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods
kubectl create rolebinding read-pods --user=tiago --role=pod-reader --namespace=development
kubectl create clusterrole nodes-reader --verb=get --verb=list --verb=watch --resource=nodes
kubectl create clusterrolebinding nodes-reader --user=tiago --clusterrole=nodes-reader
```

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

### Summary
```bash
# Cluster administrators only
kubectl auth can-i list pods --as=tiago --namespace=development
# Uses the current user
kubectl auth can-i list pods --namespace=development
```

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
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# Add the Kubernetes apt repository
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

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

### Summary
```bash
# control-plane/worker - install containerd
sudo swapoff -A
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab # comment swap entry
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# control-plane/worker - install k8s components
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# control-plane only
sudo kubeadm init --pod-network-cidr 10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# worker only
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

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
- https://v1-22.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## Perform a version upgrade on a Kubernetes cluster using `kubeadm`
Reference:
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

<details>
<summary>Solution</summary>

For upgrading a cluster, you should use the reference material and follow the steps. 

The following steps are based on a highly available cluster described on the previous objectives.

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

### (Control) Upgrade `kubelet` and `kubectl`
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
</details>

## Implement etcd backup and restore
Reference:
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#restoring-an-etcd-cluster

<details>
<summary>Solution</summary>

The following steps are for a stacked HA Kubernetes cluster, with stacked `etcd`. For external `etcd` setup, the commands might be different.

### Implement etcd backup

- To get a backup of `etcd`, you will first need to get the `etcdctl` command line client.

You can get the package using:
```bash
sudo apt install etcd-client
``` 

- To use the `etcdctl` on your cluster, you will need to get all nodes IP addresses, as well as the path for certificates and the certificate key being used by `etcd`. 

```bash
# Get the nodes IP addresses
kubectl get pods -n kube-system -o wide | grep etcd

# Print the pod definition for `etcd`, we will need some of the parameters.
sudo cat /etc/kubernetes/manifests/etcd.yaml
```

- To get started with `etcdctl`, we can get the list of members of the cluster:
```bash
# The --endpoints can be extracted from the etcd pods
# --cacert matches the etcd pod definition --trusted-ca-file=" parameter
# --cert matches the etcd pod definition "--cert-file=" parameter
# --key matches the etcd pod definition "--key-file=" parameter

sudo ETCDCTL_API=3 etcdctl \
--endpoints=https://192.168.1.14:2379,https://192.168.1.17:2379,https://192.168.1.18:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
member list --write-out table

# It should output something like this:
# +------------------+---------+---------------+---------------------------+---------------------------+
# |        ID        | STATUS  |     NAME      |        PEER ADDRS         |       CLIENT ADDRS        |
# +------------------+---------+---------------+---------------------------+---------------------------+
# | 5997140dfeb3820d | started |   k8s-control | https://192.168.1.14:2380 | https://192.168.1.14:2379 |
# | 715ce16910f037a8 | started | k8s-control-2 | https://192.168.1.17:2380 | https://192.168.1.17:2379 |
# | 8cfeffbdc61e61cb | started | k8s-control-3 | https://192.168.1.18:2380 | https://192.168.1.18:2379 |
# +------------------+---------+---------------+---------------------------+---------------------------+
```

- To perform the backup, just change the arguments
```bash
sudo ETCDCTL_API=3 etcdctl \
--endpoints=https://192.168.1.14:2379,https://192.168.1.17:2379,https://192.168.1.18:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save ~/etcd_backup.db
```

### Implement etcd restore
When we perform etcd restore, it will create create new etcd data directories. All members should restore the same snapshot.

- First, you have to copy the backup file from the node it was taken and copy on all other control plane nodes:
```bash
# make sure to replace the username and node names
# copy to node 2
scp ~/etcd_backup.db administrator@k8s-control-2:~/etcd_backup.db

# copy to node 3
scp ~/etcd_backup.db administrator@k8s-control-3:~/etcd_backup.db
```

- To perform the restore using the backup file previously created:
```bash
# SSH into node 1
# Perform a restore on node 1
sudo ETCDCTL_API=3 etcdctl \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore ~/etcd_backup.db \
--name=k8s-control \
--initial-cluster k8s-control=https://192.168.1.14:2380,k8s-control-2=https://192.168.1.17:2380,k8s-control-3=https://192.168.1.18:2380 \
--initial-cluster-token etcd-cluster-140622 \
--initial-advertise-peer-urls https://192.168.1.14:2380 \
--data-dir=/var/lib/etcd_restore

# SSH into node 2
# Perform a restore on node 2
sudo ETCDCTL_API=3 etcdctl \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore ~/etcd_backup.db \
--name=k8s-control-2 \
--initial-cluster k8s-control=https://192.168.1.14:2380,k8s-control-2=https://192.168.1.17:2380,k8s-control-3=https://192.168.1.18:2380 \
--initial-cluster-token etcd-cluster-140622 \
--initial-advertise-peer-urls https://192.168.1.17:2380 \
--data-dir=/var/lib/etcd_restore

# SSH into node 3
# Perform a restore on node 3
sudo ETCDCTL_API=3 etcdctl \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore ~/etcd_backup.db \
--name=k8s-control-3 \
--initial-cluster k8s-control=https://192.168.1.14:2380,k8s-control-2=https://192.168.1.17:2380,k8s-control-3=https://192.168.1.18:2380 \
--initial-cluster-token etcd-cluster-140622 \
--initial-advertise-peer-urls https://192.168.1.18:2380 \
--data-dir=/var/lib/etcd_restore
```

- We need to stop all API Server and etcd pods to replace etcd's data directory. We will do this by removing the static pods manifest on each control plane node. This action will make kubelet stop the pods. Execute the commands on each control plane node:
```bash
# Create a tmp folder and move all manifests there.
mkdir /tmp/kubernetes-manifests/
sudo mv /etc/kubernetes/manifests/*.yaml /tmp/kubernetes-manifests/
```

- Lets create a backup from the last etcd data directory and replace with the restored one. Execute the commands on each control plane node:
```bash
# Create a backup folder and lets rename the restored etcd into the /var/lib/etcd
sudo mv /var/lib/etcd /var/lib/etcd.bak
sudo mv /var/lib/etcd_restore /var/lib/etcd
```

- Now we move the static pods manifests back to the Kubernetes folder. Execute the commands on each control plane node:
```bash
# Create a tmp folder and move all manifests there.
sudo mv /tmp/kubernetes-manifests/*.yaml /etc/kubernetes/manifests/
```
</details>