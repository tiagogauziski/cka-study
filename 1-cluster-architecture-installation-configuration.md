# Cluster Architecture, Installation & Configuration (25%)

## Table of contents
1. [Manage role based access control (RBAC)](#manage-role-based-access-control-rbac)
1. [Bonus: How to trobleshoot permission problems](#bonus-how-to-trobleshoot-permission-problems)
1. [Use Kubeadm to install a basic cluster](#use-kubeadm-to-install-a-basic-cluster)

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
<details>
<summary>Solution</summary>

Reference: 
- https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access
- https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation

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
<details>
<summary>Solution</summary>

</details>