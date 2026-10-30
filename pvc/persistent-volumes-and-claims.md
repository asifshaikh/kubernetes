# Kubernetes PersistentVolumes and PersistentVolumeClaims

## Overview

Kubernetes separates storage provisioning from the applications that use storage:

- A **PersistentVolume (PV)** is a cluster resource representing storage that has been provisioned for Kubernetes. It may be backed by a cloud disk, network file system, or another storage system.
- A **PersistentVolumeClaim (PVC)** is a request for storage made by a user or workload. It specifies requirements such as capacity, access mode, and optionally a StorageClass.

A Pod references a PVC, not a PV directly. This lets workloads request storage without needing to know the underlying storage details.

## How PVs and PVCs work

1. **Storage is made available.** An administrator can create PVs manually (static provisioning), or a StorageClass can provision them automatically (dynamic provisioning).
2. **A PVC requests storage.** Kubernetes looks for a PV that satisfies the claim's requirements, or provisions one through the requested StorageClass.
3. **The claim binds to a volume.** A PVC binds to one PV. The PV and PVC have a one-to-one binding.
4. **A Pod mounts the claim.** The Pod refers to the PVC by name and mounts it into a container.
5. **The data outlives the Pod.** Deleting a Pod generally does not delete its PVC or the underlying data. PVC and PV deletion behavior depends on reclaim policy and storage-provisioner behavior.

A PVC must be in the same namespace as the Pod that uses it. PVs are cluster-scoped.

## Access modes

Access modes describe how a volume may be mounted. They are not a universal guarantee that the storage system will enforce application-level write coordination.

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (`RWO`) | Can be mounted read-write by a single node at a time. Multiple Pods on that node may be able to use it, depending on the volume and configuration. |
| `ReadOnlyMany` (`ROX`) | Can be mounted read-only by many nodes. |
| `ReadWriteMany` (`RWX`) | Can be mounted read-write by many nodes. |
| `ReadWriteOncePod` (`RWOP`) | Can be mounted read-write by a single Pod cluster-wide. Requires support from the CSI driver and Kubernetes version. |

The storage backend and driver determine which modes are supported. For example, many block-storage volumes support `RWO`, while shared file storage may support `RWX`.

## Static provisioning example

This example creates a local PV for learning or single-node testing. A `hostPath` volume is tied to a node and is generally not appropriate for production clusters.

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
~~~

Create a PVC that can bind to the PV:

~~~yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
~~~

The claim requests less than the PV's capacity, which is sufficient for binding. Its requested access mode and StorageClass must also be compatible with the PV.

## Using a PVC in a Pod

A Pod mounts the claim as a volume. Files written under the mount path are stored on the volume rather than only in the container's writable layer.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
  volumes:
    - name: web-content
      persistentVolumeClaim:
        claimName: demo-pvc
~~~

The Pod and `demo-pvc` must be in the same namespace.

## Dynamic provisioning example

A StorageClass connects PVC requests to a storage provisioner. StorageClass names and provisioner settings are cluster-specific; use one provided by your cluster administrator or cloud provider.

~~~yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
~~~

If `standard` is configured for dynamic provisioning, Kubernetes asks its provisioner to create storage for the claim. The resulting PV is then bound to the PVC. If `storageClassName` is omitted, the cluster's default StorageClass may be used. To request a PVC without a StorageClass, set `storageClassName: ""`.

## Reclaim policies and lifecycle

A PV's `persistentVolumeReclaimPolicy` controls what happens to its storage after its PVC is deleted:

- **`Retain`**: The PV and data are kept for manual recovery or cleanup. The PV generally needs manual intervention before it can be reused.
- **`Delete`**: The PV is deleted, and the provisioner may delete the backing storage.
- **`Recycle`**: Deprecated; do not use for new configurations.

Check the StorageClass, PV, CSI driver, and provider documentation before relying on deletion behavior. Back up important data independently of Kubernetes reclaim settings.

## Where PVs and PVCs are used

PVs and PVCs are useful when application data must persist beyond a container or Pod's lifetime, including:

- Database data directories
- Uploaded files and shared application content
- Build caches and other reusable data
- Stateful workloads managed by a StatefulSet
- Workloads requiring a particular storage capacity or access mode

For stateless workloads that can recreate their data, a persistent volume may not be necessary.

## Common interview questions and answers

### 1. What is the difference between a PV and a PVC?

A PV represents available or provisioned storage. A PVC is a namespaced request for storage. Kubernetes binds a compatible PV to a PVC, and Pods consume the PVC.

### 2. Are PVs namespaced?

No. PVs are cluster-scoped. PVCs are namespaced, as are the Pods that use them.

### 3. What is the difference between static and dynamic provisioning?

With static provisioning, an administrator creates PVs ahead of time. With dynamic provisioning, a PVC and StorageClass trigger a provisioner to create a suitable volume.

### 4. How does Kubernetes select a PV for a PVC?

Kubernetes looks for an available PV satisfying the claim's requirements, including capacity, access modes, volume mode, and StorageClass. A compatible dynamically provisioned volume may be created if the claim uses a suitable StorageClass.

### 5. Does `ReadWriteOnce` mean only one Pod can use a volume?

Not necessarily. It means read-write mounting is restricted to one node at a time. Multiple Pods on that node may use the volume if the driver and workload permit it. `ReadWriteOncePod` is the mode intended to restrict mounting to one Pod.

### 6. Can one PVC bind to multiple PVs?

No. A PVC binds to one PV. A PV can be bound to only one PVC at a time.

### 7. What happens to a PV when its PVC is deleted?

It depends on the PV's reclaim policy. `Retain` keeps the volume for manual handling; `Delete` requests deletion of the PV and potentially its backing storage.

### 8. Why might a PVC remain in `Pending`?

Common causes include no matching PV, an unsupported access mode, insufficient capacity, a StorageClass that does not exist, or a provisioner that cannot create the requested storage. With `WaitForFirstConsumer`, provisioning or binding may wait until a Pod is scheduled.

### 9. What is a StorageClass?

A StorageClass describes a class of storage and identifies the provisioner and configuration used for dynamic provisioning. It can also define parameters such as reclaim policy and volume-binding behavior.

### 10. Does deleting a Pod delete its PVC data?

Usually not. A Pod's volume mount does not own the PVC. The claim and backing storage have their own lifecycle, governed by PVC deletion, PV reclaim policy, and the storage provider.

## Practical checks

~~~sh
kubectl get pv
kubectl get pvc
kubectl describe pvc demo-pvc
kubectl describe pv demo-pv
~~~

If a claim is pending, inspect its events with `kubectl describe pvc` and verify the StorageClass and storage-driver configuration.