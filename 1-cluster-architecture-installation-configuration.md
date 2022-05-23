# Cluster Architecture, Installation & Configuration (25%)

## Table of contents
1. [Manage role based access control (RBAC)](#manage-role-based-access-control-rbac)

## Manage role based access control (RBAC)
Reference: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

<details>
<summary>Solution</summary>
Role-based access (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within your organization.

It allows fine grain access permissions to users, groups or service accounts within your cluster.

`Role` and `ClusterRole` contains rules that represents a set of permissions (e.g. get a list of pods of a namamespace, or get a list all the nodes of a cluster)

`RoleBinding` and `ClusterRoleBinding` associates permissions defined in a Role/ClusterRole to a list of subjects (users, groups or service accounts).

### Create a Role and associate it with a RoleBinding
Reference: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-example
1. Create a namespace to be used in the Role and RoleBinding
```
kubectl create namespace development
```

1. Create a Role
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
```

1. Create a RoleBinding
```yaml
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

</details>