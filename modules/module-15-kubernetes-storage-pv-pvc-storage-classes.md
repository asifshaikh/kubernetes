# Module 15 — Kubernetes Storage: Volumes, PVs, PVCs and StorageClasses

**Audience:** Students familiar with Docker basics and Kubernetes Pods, Deployments and Services.

## Learning objectives

By the end of this module, students should be able to:
- Explain container writable layers and why important data needs separate storage.
- Use Docker bind mounts and named volumes.
- Mount local directories into containers and local Kubernetes clusters.
- Explain `emptyDir`, `hostPath`, local PVs and CSI-backed storage.
- Create and consume PersistentVolumes (PV) and PersistentVolumeClaims (PVC).
- Explain static/dynamic provisioning, StorageClasses, access modes and reclaim policies.
- Compare AWS EBS, EFS and S3 for Kubernetes workloads.
- Troubleshoot common storage and mount problems.

---

## 1. Why applications need persistent storage

Imagine a shopping application that stores product images, invoices, reports and database files. If this data exists only in a container's writable layer, removing or replacing that container can result in data loss.

Kubernetes routinely replaces Pods. A replacement Pod is not guaranteed to have the previous Pod's local filesystem.

**Core principle:** Treat containers and Pods as replaceable. Store important data in storage whose lifecycle is independent of an individual container or Pod.

| Data | Typical storage choice |
|---|---|
| Temporary processing files | `emptyDir` |
| Cache shared by containers in one Pod | `emptyDir` |
| Database data directory | PVC backed by block storage |
| Shared files across replicas | RWX-capable filesystem such as EFS |
| User-uploaded images and documents | Often object storage such as S3 |
| Application configuration | ConfigMap or Secret, not a PVC by default |

## 2. Container filesystem: image, writable layer and volume

A container image is a read-only template. When a container starts, the runtime adds a writable layer. Changes in this layer are tied to that container's lifecycle.

A mounted volume is a separate filesystem location made available inside the container. Its persistence depends on the volume type and storage provider.

```text
Container image (read-only)
          |
          v
Container writable layer  ---> temporary container changes

Separate mounted volume  ---> data stored outside writable layer
```

A volume is not automatically permanent. For example, `emptyDir` survives a container restart while its Pod remains on the node, but it is removed when the Pod is deleted.

## 3. Docker storage: bind mounts and named volumes

### 3.1 Bind mount

A bind mount maps an explicit host path into a container. It is particularly useful during local development.

```bash
mkdir -p ./app-data

docker run -d --name web -p 8080:80 \
  --mount type=bind,source="$(pwd)/app-data",target=/usr/share/nginx/html \
  nginx:alpine

echo "Hello from my host directory" > ./app-data/index.html
```

Open `http://localhost:8080`. NGINX serves the file from the host directory.

Inspect the mount:

```bash
docker inspect web
```

The source path is on the Docker host. With Docker Desktop, host paths are shared through its VM.

### 3.2 Named volume

A named volume is managed by Docker rather than being tied to a project directory.

```bash
docker volume create mysql-data

docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=example \
  -v mysql-data:/var/lib/mysql \
  mysql:8

docker volume ls
docker volume inspect mysql-data
```

Remove and recreate the container using the same volume:

```bash
docker rm -f mysql

docker run -d --name mysql-again \
  -e MYSQL_ROOT_PASSWORD=example \
  -v mysql-data:/var/lib/mysql \
  mysql:8
```

The data remains in the named volume. The password is illustrative only; do not use a real production password in command-line examples.

### 3.3 Bind mount vs named volume

| Feature | Bind mount | Named volume |
|---|---|---|
| Source | Explicit host path | Docker-managed location |
| Source-code development | Very convenient | Less common |
| Persistent application data | Possible | Common |
| Portability | Depends on host path | Better within Docker-managed environments |

## 4. Kubernetes volume model

Kubernetes defines a volume at the **Pod** level. Containers in the same Pod can mount that volume at different paths and share files through it.

```text
                 Kubernetes Pod
       +--------------------------------+
       | Container A    Container B    |
       | /app/output    /shared        |
       |       \          /            |
       |        Shared Volume          |
       +--------------------------------+
```

The Pod's `volumes` section defines storage. Each container's `volumeMounts` section defines where it appears inside that container.

## 5. Common Kubernetes volume types

### 5.1 `emptyDir`

An `emptyDir` is created when a Pod is assigned to a node. It starts empty and can be shared by containers in that Pod. It is useful for temporary files, scratch space and sidecar communication.

Create `emptydir-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /shared/time.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "while true; do cat /shared/time.txt 2>/dev/null; sleep 10; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  volumes:
    - name: shared-data
      emptyDir: {}
```

```bash
kubectl apply -f emptydir-demo.yaml
kubectl get pod emptydir-demo
kubectl logs emptydir-demo -c reader
```

Delete and recreate the Pod. Its new `emptyDir` starts empty.

```bash
kubectl delete pod emptydir-demo
kubectl apply -f emptydir-demo.yaml
```

Memory-backed `emptyDir`:

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory
      sizeLimit: 128Mi
```

This uses RAM and counts against the memory available to the Pod. Configure resource requests and limits carefully.

### 5.2 `hostPath`

A `hostPath` mounts a directory from the Kubernetes **node** filesystem into a Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo hello-from-node >> /node-data/message.txt; sleep 3600"]
      volumeMounts:
        - name: node-data
          mountPath: /node-data
  volumes:
    - name: node-data
      hostPath:
        path: /tmp/k8s-lab
        type: DirectoryOrCreate
```

```bash
kubectl apply -f hostpath-demo.yaml
kubectl describe pod hostpath-demo
kubectl exec hostpath-demo -- cat /node-data/message.txt
```

Limitations:
- The path is on the node, not automatically on your laptop.
- Another node has a different local filesystem.
- If the Pod moves to another node, the same data may not be present.
- Careless paths can expose sensitive host files.

Use `hostPath` for controlled labs and certain node-level agents, not as the default production storage choice for application data.

### 5.3 Local PersistentVolume

A local PV represents storage attached to a particular node. It has node affinity because the data is available only on that node. It can offer good performance but reduces scheduling flexibility. It is not a laptop directory automatically shared with a cluster.

### 5.4 Network and cloud-backed storage

Kubernetes can use NFS, iSCSI, cloud block storage, cloud file storage and other external storage systems. The **Container Storage Interface (CSI)** is the standard integration mechanism for storage drivers.

## 6. Mount a local laptop directory into Kubernetes

This is a common source of confusion:

- Docker bind mount source: path on the Docker host.
- Kubernetes `hostPath` source: path on the Kubernetes node.
- With kind, the node itself runs in a container.
- With Minikube, the node runs in a VM or container depending on its driver.
- A local laptop directory must first be made available inside the cluster node.

### 6.1 Example using kind

Create a directory and file:

```bash
mkdir -p ./k8s-local-data
echo "data from my laptop" > ./k8s-local-data/message.txt
```

Create `kind-storage.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: ./k8s-local-data
        containerPath: /var/local/k8s-local-data
```

Create the cluster:

```bash
kind create cluster --name storage-lab --config kind-storage.yaml
```

The directory is now available inside the kind node container at `/var/local/k8s-local-data`. A Pod can use that node path:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: local-directory-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /host-data/message.txt; sleep 3600"]
      volumeMounts:
        - name: host-data
          mountPath: /host-data
          readOnly: true
  volumes:
    - name: host-data
      hostPath:
        path: /var/local/k8s-local-data
        type: Directory
```

```bash
kubectl apply -f local-directory-demo.yaml
kubectl logs local-directory-demo
```

This is a local development lab. Do not use this `hostPath` pattern as production multi-node application storage.

### 6.2 Minikube

Minikube offers `minikube mount` to expose a host directory inside the Minikube node. The exact setup depends on OS and driver.

```bash
minikube mount ./k8s-local-data:/mnt/k8s-local-data
```

Keep the mount process running. Use `/mnt/k8s-local-data` as the node path in a `hostPath` volume.

## 7. PersistentVolumes (PV)

A PersistentVolume is a cluster resource representing storage available for workloads. Its lifecycle is independent of an individual Pod. A PV may be created manually (static provisioning) or dynamically by a provisioner.

Example local PV for a deliberately configured lab node:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  local:
    path: /mnt/k8s-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - storage-node
```

This is illustrative: the node name and path must exist and the path must be prepared on that node. Local PVs require node affinity.

Important PV fields:
- `capacity`: advertised storage capacity.
- `accessModes`: supported access modes.
- `volumeMode`: `Filesystem` or `Block`.
- `storageClassName`: class used to match a claim.
- `persistentVolumeReclaimPolicy`: behavior after release.
- `nodeAffinity`: required for local volumes.

## 8. PersistentVolumeClaims (PVC)

A PVC is a request for storage made by a user or workload. A Pod normally references the PVC, rather than directly referencing a PV.

Think:
- PV = storage resource.
- PVC = request for storage.
- Pod = workload consuming the claim.

Example `local-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl apply -f local-pvc.yaml
kubectl get pv
kubectl get pvc
kubectl describe pvc app-data
```

The claim binds only if a compatible PV exists or a StorageClass can dynamically provision one.

## 9. Mount a PVC into a Pod

Create `pvc-consumer.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-consumer
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo persistent-data > /data/message.txt; sleep 3600"]
      volumeMounts:
        - name: app-storage
          mountPath: /data
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: app-data
```

```bash
kubectl apply -f pvc-consumer.yaml
kubectl exec pvc-consumer -- cat /data/message.txt
```

Delete and recreate the Pod, but do not delete its PVC:

```bash
kubectl delete pod pvc-consumer
kubectl apply -f pvc-consumer.yaml
kubectl exec pvc-consumer -- cat /data/message.txt
```

The data should remain if the PVC is still bound to backing storage that preserves it. Durability depends on the storage system, replication, failure domain, backups and reclaim policy.

## 10. Static vs dynamic provisioning

**Static provisioning:** An administrator creates PVs in advance. A PVC is matched with a compatible PV.

```text
Admin creates PV -> User creates PVC -> Kubernetes binds claim -> Pod mounts PVC
```

**Dynamic provisioning:** A PVC requests storage using a StorageClass. A provisioner, commonly a CSI driver, creates the backing storage and PV.

```text
User creates PVC -> StorageClass -> CSI provisioner creates storage
                 -> PV is created and bound -> Pod mounts PVC
```

Dynamic provisioning is common in cloud clusters because teams do not need to pre-create every PV.

## 11. StorageClasses

A StorageClass describes a class/profile of storage and the provisioner and parameters used to create it dynamically.

Inspect the cluster:

```bash
kubectl get storageclass
kubectl describe storageclass
```

Generic example (do not apply unchanged):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-storage
provisioner: csi.example.com
parameters:
  type: fast
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

`csi.example.com` is a placeholder, not a real installed driver.

Important fields:
- `provisioner`: CSI driver responsible for provisioning.
- `parameters`: provider-specific settings, such as disk type.
- `reclaimPolicy`: policy applied to provisioned PVs.
- `allowVolumeExpansion`: whether supported claims can be expanded.
- `volumeBindingMode`: when provisioning/binding happens.

### Immediate vs WaitForFirstConsumer

- `Immediate`: provisioning/binding begins when the PVC is created.
- `WaitForFirstConsumer`: provisioning/binding waits until a Pod using the claim is scheduled.

The latter helps topology-aware storage choose a compatible node or Availability Zone. It is useful for zonal disks where scheduling and volume placement must align.

## 12. Access modes and volume modes

| Access mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read-write mount by one node. Multiple Pods on that node may use it, depending on driver/filesystem. |
| `ReadOnlyMany` (ROX) | Read-only mount by many nodes, if supported. |
| `ReadWriteMany` (RWX) | Read-write mount by many nodes, if supported. |
| `ReadWriteOncePod` (RWOP) | Read-write mount by one Pod, with CSI support requirements. |

**Interview trap:** RWO means one node, not necessarily one Pod. Support and enforcement depend on the storage driver and Kubernetes implementation.

Volume modes:
- `Filesystem`: volume is mounted as a directory.
- `Block`: raw block device is exposed to the container.

Most application workloads use filesystem volumes.

## 13. Reclaim policies

When a PVC is deleted, the PV may become released. The reclaim policy controls what happens to the backing storage.

- `Retain`: preserve storage for manual recovery or cleanup.
- `Delete`: delete the PV and, where supported by the provisioner, the backing storage.
- `Recycle`: deprecated legacy behavior; do not use for new designs.

Understand the policy before deleting a production PVC. `Delete` can result in permanent data loss.

## 14. StatefulSets and storage

Stateful applications may need stable identities and stable storage. A StatefulSet can use `volumeClaimTemplates` to create a separate PVC for each Pod.

Example `statefulset-demo.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          volumeMounts:
            - name: web-data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: web-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

Each Pod receives its own claim, for example `web-data-web-0` and `web-data-web-1`, if provisioning succeeds.

A StatefulSet does **not** automatically replicate data between replicas. Each Pod's volume is separate unless the application or storage layer provides replication.

## 15. Storage on AWS EKS

EKS commonly uses CSI drivers to connect Kubernetes to AWS storage services.

### 15.1 Amazon EBS CSI driver

Amazon EBS provides block storage. It is commonly used for databases and applications needing block-backed filesystems.

Typical characteristics:
- EBS volumes are Availability-Zone scoped.
- Commonly used with RWO workloads.
- A volume generally cannot be mounted read-write by Pods on arbitrary nodes across different Availability Zones.
- Snapshots and expansion depend on configuration and supported features.

### 15.2 Amazon EFS CSI driver

Amazon EFS provides shared network file storage.

Typical characteristics:
- Supports shared filesystem access patterns.
- Useful when multiple Pods/nodes need shared files.
- Commonly used for RWX workloads.
- Throughput, performance, access points and cost need consideration.

### 15.3 EBS vs EFS vs S3

| Feature | EBS | EFS | S3 |
|---|---|---|---|
| Storage model | Block | Shared file | Object |
| Typical use | Database/data disk | Shared files | Uploads, images, backups |
| Kubernetes mount | CSI volume | CSI volume | Usually API/SDK, not a normal POSIX filesystem |
| Multi-node shared writes | Not typical | Supported | Object API access |
| Main consideration | AZ placement and performance | Throughput and access patterns | Object semantics and application integration |

Choose based on application access patterns, consistency, latency, availability and cost—not just ease of configuration.

### 15.4 EKS storage architecture

```text
                 EKS Cluster
                      |
                 Application Pod
                      |
                     PVC
                      |
                 StorageClass
                      |
                  CSI Driver
                   /      \
                  /        \
              Amazon EBS  Amazon EFS
              block disk  shared filesystem
```

The CSI driver must be installed and configured. The cluster needs the appropriate IAM permissions and network access. For EBS, zone placement matters.

Inspect available StorageClasses before creating a claim:

```bash
kubectl get storageclass
```

Example claim (the StorageClass name must actually exist):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 5Gi
```

Example Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo stored-on-ebs > /data/info.txt; sleep 3600"]
      volumeMounts:
        - name: app-data
          mountPath: /data
  volumes:
    - name: app-data
      persistentVolumeClaim:
        claimName: ebs-app-data
```

```bash
kubectl get pvc
kubectl get pv
kubectl describe pvc ebs-app-data
kubectl get pods -o wide
```

Do not assume `ebs-gp3` exists by default. Configure the EBS CSI driver, IAM permissions and a suitable StorageClass first.

## 16. Volume mount options

```yaml
volumeMounts:
  - name: app-data
    mountPath: /var/lib/app
    readOnly: false
    subPath: reports
```

- `name`: matches a Pod volume.
- `mountPath`: path inside the container.
- `readOnly`: mounts read-only when true.
- `subPath`: mounts a subdirectory or file within the volume.

For sensitive configuration, use read-only mounts where appropriate. A `subPath` mount does not receive ConfigMap/Secret updates in the same way as a normal projected mount. Permissions and ownership can also affect writes.

## 17. Permissions and security

A volume may mount successfully while the process cannot write to it. Common causes include UID/GID mismatch, directory permissions, non-root execution and driver-specific behavior.

A Pod-level `fsGroup` can help for supported volume types:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: permission-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 2000
    fsGroup: 2000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "id; touch /data/test.txt; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
```

Exact ownership behavior depends on volume type and CSI driver. Do not solve every permissions problem by running the application as root.

## 18. Expand a PVC

If the StorageClass allows expansion and the driver supports it, increase the requested size. Example:

```bash
kubectl edit pvc app-data
```

Change the request:

```yaml
resources:
  requests:
    storage: 1Gi
```

Then inspect:

```bash
kubectl get pvc app-data
kubectl describe pvc app-data
```

Expansion depends on provisioner, filesystem and Kubernetes version. Shrinking PVCs is generally unsupported. Back up important data before resizing.

## 19. Practical lab: persistent notes application

**Goal:** Replace a Pod while preserving its data on a PVC.

First inspect your local cluster:

```bash
kubectl get storageclass
```

If a default dynamic StorageClass exists, create `notes-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Create `notes-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: notes-app
spec:
  containers:
    - name: notes
      image: busybox:1.36
      command: ["sh", "-c", "sleep 36000"]
      volumeMounts:
        - name: notes-storage
          mountPath: /notes
  volumes:
    - name: notes-storage
      persistentVolumeClaim:
        claimName: notes-data
```

Run:

```bash
kubectl apply -f notes-pvc.yaml
kubectl get pvc notes-data
kubectl apply -f notes-pod.yaml
kubectl wait --for=condition=Ready pod/notes-app --timeout=120s

kubectl exec notes-app -- sh -c 'echo "My first persistent note" > /notes/note.txt'
kubectl exec notes-app -- cat /notes/note.txt
```

Delete and recreate only the Pod:

```bash
kubectl delete pod notes-app
kubectl apply -f notes-pod.yaml
kubectl wait --for=condition=Ready pod/notes-app --timeout=120s
kubectl exec notes-app -- cat /notes/note.txt
```

The note should remain if the PVC is still bound to backing storage that preserves it.

Cleanup:

```bash
kubectl delete pod notes-app
kubectl delete pvc notes-data
```

**Warning:** Deleting a PVC can delete backing storage when the PV reclaim policy is `Delete`. Inspect the policy before cleanup if the data matters.

## 20. Practical lab: two containers sharing a volume

Create `shared-volume-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
    - name: producer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /shared/events.log; sleep 5; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: consumer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do tail -n 5 /shared/events.log 2>/dev/null; sleep 10; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

```bash
kubectl apply -f shared-volume-pod.yaml
kubectl logs shared-volume-pod -c consumer
```

This demonstrates sharing within one Pod, not sharing between Pods on different nodes.

## 21. Troubleshooting

### PVC stuck in `Pending`

```bash
kubectl get pvc
kubectl describe pvc <claim-name>
kubectl get pv
kubectl get storageclass
```

Possible causes:
- No compatible PV.
- Incorrect or missing StorageClass.
- CSI driver or provisioner unavailable.
- Unsupported capacity or access mode.
- Topology/node-affinity mismatch.
- Cloud permissions, quota or provisioning failure.

### Pod stuck in `ContainerCreating`

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for mount/attach errors, missing claims, zone mismatch and CSI driver errors.

### `Permission denied`

```bash
kubectl exec <pod-name> -- id
kubectl exec <pod-name> -- ls -ld <mount-path>
kubectl describe pod <pod-name>
```

Review UID/GID, `fsGroup`, filesystem permissions and driver behavior.

### Data missing after rescheduling

Check whether the volume is `hostPath` or local storage tied to one node, whether the PV is zone-bound, whether the CSI driver can attach it to the new node, and whether the application itself replicates data.

## 22. Production design recommendations

1. Keep application containers stateless wherever practical.
2. Use PVCs for durable application data, not `emptyDir`.
3. Use CSI-backed StorageClasses for dynamic provisioning.
4. Choose EBS, EFS or another storage system based on access patterns.
5. Understand access modes and topology before scaling replicas.
6. Set realistic capacity requests and monitor usage.
7. Plan backups, snapshots, restore testing and disaster recovery.
8. Review reclaim policies before deleting claims.
9. Use least-privilege permissions and non-root containers where possible.
10. Avoid `hostPath` for ordinary application persistence in production.
11. Do not assume multiple replicas can safely share one PVC; validate driver and application behavior.
12. Treat storage as a stateful dependency with its own availability and recovery plan.

## 23. Mini-project: persistent file storage for CloudCart

Extend CloudCart with a file-upload capability.

Requirements:
- Deploy a backend that writes uploaded files to `/uploads`.
- Create a PVC for the upload directory.
- Mount the PVC into the backend Pod.
- Create or upload a sample file.
- Delete and recreate the Pod and verify the file remains.
- Document StorageClass, access mode, capacity and reclaim policy.
- For EKS, evaluate EBS for a single-node-attached workload or EFS if multiple replicas need shared writable files.

Architecture:

```text
Browser -> CloudCart Backend Pod
                    |
                /uploads
                    |
                   PVC
                    |
              StorageClass
                    |
                CSI Driver
                    |
                 EBS/EFS
```

Extension challenges:
- Scale the backend to two replicas. Does the storage support the required access pattern?
- What changes if the backend moves to another Availability Zone?
- How would you back up uploaded files?
- Should the database store file contents or object-storage references?
- Would S3 be a better fit for user-uploaded objects?

**Design discussion:** For production uploads, object storage such as Amazon S3 is often a better fit than mounting a filesystem into application Pods. Store object keys and metadata in the database. Use EBS/EFS when the application genuinely needs block or shared filesystem semantics.

## 24. Interview questions and answers

**Q1. Why shouldn't important data live in a container writable layer?**  
It is tied to the container lifecycle. Replacing/removing the container can remove the data.

**Q2. What is a Kubernetes Volume?**  
A storage resource made available to containers in a Pod through a mount. Persistence depends on the volume type.

**Q3. `emptyDir` vs PVC?**  
`emptyDir` is temporary and Pod-scoped. A PVC requests storage backed by a PV and can outlive a Pod.

**Q4. What is a PV?**  
A cluster resource representing storage available for workloads.

**Q5. What is a PVC?**  
A request for storage specifying requirements such as capacity, access mode and StorageClass.

**Q6. What is a StorageClass?**  
A description of a storage class and the provisioner/parameters used for dynamic provisioning.

**Q7. What is dynamic provisioning?**  
Storage is provisioned automatically in response to a PVC referencing a suitable StorageClass.

**Q8. What is static provisioning?**  
An administrator creates PV resources before users create claims.

**Q9. PV vs PVC?**  
PV represents the storage resource; PVC represents a workload's request for it.

**Q10. Does RWO mean only one Pod can mount a volume?**  
No. It means read-write mounting by one node. Multiple Pods on that node may use it, depending on driver and filesystem. RWOP is intended to constrain read-write mounting to one Pod when supported.

**Q11. What is RWX?**  
Read-write mounting from multiple nodes, if supported by the storage system and driver.

**Q12. `hostPath` vs local PV?**  
`hostPath` directly mounts a node path. A local PV is a cluster storage resource with node affinity that can be claimed through a PVC.

**Q13. What is CSI?**  
The Container Storage Interface, a standard integration mechanism for storage drivers.

**Q14. What happens when a PVC is deleted?**  
The PV is released and its reclaim policy controls what happens to backing storage.

**Q15. EBS vs EFS?**  
EBS is block storage commonly attached to one node at a time for a workload. EFS is a shared network filesystem suitable for multi-node shared access.

**Q16. Why might a PVC be Pending?**  
No compatible PV, incorrect StorageClass, missing CSI driver, unsupported capacity/access mode, topology constraints or provisioning failure.

**Q17. Does a StatefulSet replicate data?**  
No. It provides stable identities and separate claims. Replication must be handled by the application or storage system.

**Q18. Is a PVC a backup?**  
No. Use snapshots/backups and test restore procedures.

**Q19. Why is `hostPath` discouraged for production app data?**  
It ties data to a node, constrains scheduling and can expose host filesystem paths.

**Q20. When choose S3 over EBS/EFS?**  
For object-oriented data such as uploads, images, documents and backups when the application can use an object API.

## 25. Homework

1. Explain Docker bind mounts vs named volumes.
2. Create an `emptyDir` Pod and show that data disappears after Pod deletion.
3. Create a `hostPath` lab and explain why the path is node-specific.
4. Create a PVC using your local cluster's default StorageClass.
5. Mount it into a Pod and verify data after Pod replacement.
6. Explain static vs dynamic provisioning with diagrams.
7. Compare RWO, RWX and RWOP.
8. Find your cluster's default StorageClass and identify its provisioner.
9. Explain the risk of deleting a PVC with a `Delete` reclaim policy.
10. Design storage for a three-replica upload service. Choose EBS, EFS or S3 and justify the decision.

## 26. Quick revision sheet

| Term | Remember |
|---|---|
| Bind mount | Maps a host path into a Docker container |
| Docker named volume | Docker-managed data location |
| `emptyDir` | Temporary storage for a Pod |
| `hostPath` | Mounts a path from a Kubernetes node |
| PV | Cluster storage resource |
| PVC | Request for storage |
| StorageClass | Defines a storage class and provisioner |
| CSI | Standard interface for storage drivers |
| RWO | Read-write from one node |
| RWX | Read-write from many nodes, if supported |
| RWOP | Read-write from one Pod, if supported |
| Retain | Keep backing storage for manual handling |
| Delete | Delete backing storage when supported |
| EBS | AWS block storage |
| EFS | AWS shared file storage |
| S3 | AWS object storage |

---

## Suggested instructor delivery

1. Start with Docker bind mounts and named volumes.
2. Demonstrate why a container writable layer is not durable.
3. Introduce `emptyDir` and `hostPath`, including their limitations.
4. Explain PV → PVC → Pod.
5. Run the persistent notes lab and replace the Pod.
6. Demonstrate dynamic provisioning using the local cluster's StorageClass.
7. Finish with EBS vs EFS vs S3 and the CloudCart mini-project.

**Module outcome:** Students can explain Kubernetes storage, mount local/shared volumes in labs, provision persistent storage through PVCs and StorageClasses, and choose an appropriate storage design for an application.
