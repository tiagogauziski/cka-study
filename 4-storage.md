# Storage (10%)

## Table of contents
1. [Understand storage classes, persistent volumes](#understand-storage-classes-persistent-volumes)
1. [Understand volume mode, access modes and reclaim policies for volumes](#understand-volume-mode-access-modes-and-reclaim-policies-for-volumes)
1. [Understand persistent volume claims primitive](#understand-persistent-volume-claims-primitive)
1. [Know how to configure applications with persistent storage](#know-how-to-configure-applications-with-persistent-storage)


![Persistent storage objects](/img/4-persistent-storage-objects.png "Persistent storage objects")
Source: https://www.youtube.com/watch?v=_qfSzrPn9Cs

## Understand storage classes, persistent volumes
Reference: 
- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
- https://kubernetes.io/blog/2017/03/dynamic-provisioning-and-storage-classes-kubernetes/

<details>
<summary>Solution</summary>

### `StorageClass`
A storage class provides a way for administrator to describe the "classes" of storage they offer. Different classes might map to different quality-of-service levels, backup policies or to arbitrary policies determined by the cluster administrators. This concept is something called "profiles" in other storage systems.

Basically a `StorageClass` is a way for cluster administrator to configure metadata useful for external storage provider to configure the underlying persistent volume dynamically. 

When a `PersistentVolume` is created from a `PersistentVolumeClaim` that sets a specific `StorageClass`, the `PersistentVolume` will be dynamically provisioned based on the details from the `StorageClass`.

Lets say we configure a `StorageClass` with the following configuration:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

Note that:
- The `provisioner` field sets which external storage service will be provisioning the storage
- The `parameters` object defines metadata used by the external storage service to correctly provision a volume.
- `reclaimPolicy` defines what happens when the `PersistentVolumeClaim` is deleted. 
  - The default reclaim policy is `Delete`. If `PersistentVolumeClaim` is deleted, the `PersistentVolume` will also be deleted.
  - The automatic behavior might be inappropriate if the volume contains precious data. In such cases, it's more appropriate to use `Retain` policy. 
  - When set to `Retain`, if the `PersistentVolumeClaim` is deleted, the `PersistentVolume` will not be deleted. Instead, it is moved to the Released phase, where all of its data can be manually recovered.
- `allowVolumeExpansion` is a feature to allow users to resize the volume by editing the corresponding `PersistentVolumeClaim`. It only allows it to grow, not to shrink it.
- `volumeBindingMode` fiend control when the dynamic provisioning should occour.
  - The default value is `Immediate`. When set to `Immediate`, the volume binding and dynamic provisioning occurs once the `PersistentVolumeClaim` is created. 
  - For storage backends that are topology-constrained and not globally accessible from all Nodes in the cluster, `PersistentVolume` will be bound or provisioned without knowledge of the Pod's scheduling requirements. This may result in unschedulable Pods.
  - A cluster administrator can address this issue by set `volumeBindingMode` to `WaitForFirstConsumer`, which will delay the binding and provisioning of a `PersistentVolume` until a Pod using the `PersistentVolumeClaim` is created.


### `PersistentVolume`

A `PersistentVolume` is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using `StorageClass`. It is a resource in the cluster just like a node is a cluster resource. `PersistentVolume` have a lifecycle independent of any individual Pod that uses the PV.  
When a user creates a `PersistentVolumeClaim` with a specific amount of storage requested and with a certain access modes, a control loop in the master watches for new `PersistentVolumeClaim`s, finds a matching `PersistentVolume` if possible and binds them together. If the `PersistentVolume` was dynamically provisioned for a new `PersistentVolumeClaim`, the loop will always bind that `PersistentVolume` to the `PersistentVolumeClaim`.  
The user will always get at lest what they asked for, but the volume may be in excess of what was requested. Once bound, `PersistentVolumeClaim` binds are exclusive, regardless of how they were bound. 

Consider the following `PersistentVolume`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

Note that:
- `capacity` defines a specific storage capacity a `PersistentVolume` will have.
- Kubernetes supports two `volumeModes`: `Filesystem` and `Block`
  - `Filesystem`: a volume is mounted into Pods into a directory. If the volume is backed by a block device and the device is empty, Kubernetes creates a filesystem on the device before mounting it for the first time.
  - `Block`: to use a volume as a raw block device. Such volume is presented into a Pod as a block device, without any filesystem on it. This mode is useful to provide a Pod the fastest posible way to access a volume, without any filesystem layer between the Pod and the volume. 
- `accessMode`
  - `ReadWriteOnce` (RWO) - the volume can be mounted as read-write by a single node, but it still can allow multiple pods to access the volume when pods are running on the same node.
  - `ReadOnlyMany` (ROX) - the volume can be mounted as read-only by many nodes.
  - `ReadWriteMany` (RWX) - the volume can be mounted as read-write by many nodes.
  - `ReadWriteOncePod` (RWOP) - the volume can be mounted as read-write by a single Pod. You can use it to ensure one Pod across the whole cluster can read that `PersistentVolumeClaim` or write to it.
  - `persistentVolumeReclaimPolicy`:
    - `Retain` - the volume is retained after the PVC no longer exists. Safer option.
    - `Recycle` - it runs a script to clear the contents (`rm -rf /thevolume/*`) (this option should be avoided)
    - `Delete` - the volume and associated storrage assest such as any external resources (AWS EBS, CCE PD, Azure Disk) is deleted.

</details>

## Understand volume mode, access modes and reclaim policies for volumes
Reference: 
- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes

<details>
<summary>Solution</summary>

These topics have been cover on the notes section from the previous section. Let's bring these concepts back again.

### Volume modes

Volume mode is assigned to a `PersistentVolume` using the `volumeMode` field.

Kubernetes supports two `volumeModes`: `Filesystem` and `Block`.
- `Filesystem`: a volume is mounted into Pods into a directory. If the volume is backed by a block device and the device is empty, Kubernetes creates a filesystem on the device before mounting it for the first time.
- `Block`: to use a volume as a raw block device. Such volume is presented into a Pod as a block device, without any filesystem on it. This mode is useful to provide a Pod the fastest posible way to access a volume, without any filesystem layer between the Pod and the volume. 

### Access modes

Acesss mode is assigned to a `PersistentVolume` using the `accessModes` array.

- `ReadWriteOnce` (RWO) - the volume can be mounted as read-write by a single node, but it still can allow multiple pods to access the volume when pods are running on the same node.
- `ReadOnlyMany` (ROX) - the volume can be mounted as read-only by many nodes.
- `ReadWriteMany` (RWX) - the volume can be mounted as read-write by many nodes.
- `ReadWriteOncePod` (RWOP) - the volume can be mounted as read-write by a single Pod. You can use it to ensure one Pod across the whole cluster can read that `PersistentVolumeClaim` or write to it.

### Reclaim policy

Reclaim policy can set on either on a `StorageClass` as well as on a `PersistentVolume`:
- For dynamic provisioned `PersistentVolumes`, you can use `StorageClass` `reclaimPolicy` field. The `PersistentVolume` created will inherit the value from the `StorageClass`
- For static provisioned `PersistentVolumes`, it's possible to use `persistentVolumeReclaimPolicy` field.

- `Retain` - the volume is retained after the PVC no longer exists. Safer option.
- `Recycle` - it runs a script to clear the contents (`rm -rf /thevolume/*`) (this option should be avoided)
- `Delete` - the volume and associated storrage assest such as any external resources (AWS EBS, CCE PD, Azure Disk) is deleted.

</details>

## Understand persistent volume claims primitive
Reference: 
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims

<details>
<summary>Solution</summary>

Each `PersistentVolumeClaim` is a resource created by the developer in order to request storage to a specific Pod. Once the `PersistentVolumeClaim` is created, the cluster will try to match an available `PersistentVolume` or (if enabled) dynamically provision a new `PersistentVolume` based on the specified `StorageClass`.

Here is an example of `PersistentVolumeClaim`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

Note that:
- `accessModes`: the requested access mode (uses the same convention as `PersistentVolume`)
- `volumeMode`: the requested volume mode (uses the same conversion as `PersistentVolume`)
- `resources`: `PersistentVolumeClaims`, like Pods, can request specific quantities of a resource. In this case, the request is for storage. 
- `selector`: `PersistentVolumeClaims` can specify a label selector to further filter the set of volumes. Only the volumes whose labels match the selector can be bound to the claim.
- `storageClassName`: a claim can request a particular storage class. Only `PersistentVolumes` of the requested class can be bound to the `PersistentVolumeClaim`.
</details>

## Know how to configure applications with persistent storage
Reference: 
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

<details>
<summary>Solution</summary>

To configure an application with persistent storage we should have the following:
1. Because we don't have dynamically provision enabled, we will can create a `StorageClass` in order for a `PersistentVolume` to match with a `PersistentVolumeClaim`
1. We can create a `PersistentVolume` and assing the `storageClassName` field with the name of the `StorageClass`
1. Then we can proceed by creating a `PersistentVolumeClaim` and a `Pod` to bind it with the `PersistentVolume`. 
1. Note that the `storageClassName` provided on both PV and PVC should match. 
1. Note also that having a `StorageClass` is also optional for static provisioning of storage. In this case, the field should be empty of both PV and PVC.

- For this example, we want to create a NGINX pod and serve a file inside the PV. To do that, we need to create a `index.html` on every worker node:
```bash
# Check all the worker nodes:
kubectl get nodes

# Output:
# NAME          STATUS   ROLES                  AGE    VERSION
# k8s-control   Ready    control-plane,master   126d   v1.23.5
# k8s-worker1   Ready    <none>                 126d   v1.23.5
# k8s-worker2   Ready    <none>                 126d   v1.23.5

# SSH into every worker and create the file:
sudo mkdir /mnt/data
sudo sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
```

- Lets create a `StorageClass` (sample-storageclass.yaml):
```yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sample-storageclass
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

- Let's check the created `StorageClass`:
```bash
# Create the StorageClass
kubectl create -f sample-storageclass.yaml

# View the StorageClass
kubectl get storageclass

# Output:
# NAME                  PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
# sample-storageclass   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  25m
```

- Let's create a `PersistentVolume` that uses the `StorageClass` above (we are doing static provisioning here) (sample-persistentvolume.yaml):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sample-persistentstorage
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  storageClassName: sample-storageclass
  hostPath:
    path: "/mnt/data"
```

- Let's create and check the PV:
```bash
# Create the PV
kubectl create -f sample-persistentvolume.yaml

# View the PV
kubectl get pv

# Output:
# NAME                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS             REASON   AGE
# sample-persistentstorage   1Gi        RWX            Retain           Available           sample-storageclass               35m
```

- Next step is to create a `PersistentVolumeClaim` to request physical storage and a `Pod` to consume it:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sample-persistentvolumeclaim
spec:
  storageClassName: sample-storageclass
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: sample-persistentvolumeclaim-pod
spec:
  volumes:
  - name: nginx-storage
    persistentVolumeClaim:
      claimName: sample-persistentvolumeclaim
  containers:
  - name: nginx
    image: nginx
    ports:
    - name: http
      containerPort: 80
    volumeMounts:
    - name: nginx-storage
      mountPath: "/usr/share/nginx/html"
```

- Let's create and check the PVC and Pod:
```bash
# Create the resources
kubectl create -f sample-persistentvolumeclaim-pod.yaml

# Lets check the status of the PVC
kubectl get pvc

# Output: 
# NAME                           STATUS   VOLUME                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
# sample-persistentvolumeclaim   Bound    sample-persistentstorage   1Gi        RWX            sample-storageclass   3s

# Lets check the Pod
kubectl get pod -o wide

# Output: 
# NAME                               READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
# sample-persistentvolumeclaim-pod   1/1     Running   0          75s   10.244.1.25   k8s-worker1   <none>           <none>

# If we curl the Pod IP, we should get back the index.html content we previously set:
curl 10.244.1.25

# Output:
# Hello from Kubernetes storage
```

</details>