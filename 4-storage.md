# Storage (10%)

## Table of contents
1. [Understand storage classes, persistent volumes](#understand-storage-classes-persistent-volumes)

## Understand storage classes, persistent volumes
Reference: 
- https://kubernetes.io/docs/concepts/storage/storage-classes/

<details>
<summary>Solution</summary>

### `StorageClass`
A storage class provides a way for administrator to describe the "classes" of storage they offer. Different classes might map to different quality-of-service levels, backup policies or to arbitrary policies determined by the cluster administrators. This concept is something called "profiles" in other storage systems.

Basically a `StorageClass` is a way for cluster administrator to configure metadata useful for external storage provider to configure the underlying persistent volume. 

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

</details>