# Kubernetes ReplicaSet Guide

## What Is a ReplicaSet?

A Kubernetes `ReplicaSet` maintains a stable number of identical Pods. It compares the desired number of replicas with the number of matching Pods currently running and creates or removes Pods until the two numbers match.

A ReplicaSet helps ensure that:

- The requested number of Pods is running.
- Failed or deleted Pods are replaced automatically.
- Pods with the expected labels are managed by the ReplicaSet.
- The application continues running when an individual Pod fails.

A ReplicaSet is usually managed by a `Deployment`. Creating a ReplicaSet directly is useful for learning or for cases where you do not need Deployment features such as rolling updates and rollbacks.

## Example ReplicaSet

The following ReplicaSet maintains three Nginx Pods:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply and inspect it with:

```powershell
kubectl apply -f replicaset.yaml
kubectl get replicasets
kubectl get pods -l app=nginx
kubectl describe replicaset nginx-replicaset
```

Delete the ReplicaSet and the Pods it owns with:

```powershell
kubectl delete -f replicaset.yaml
```

## How a ReplicaSet Works

1. The ReplicaSet controller reads `spec.replicas` and determines the desired number of Pods.
2. It uses `spec.selector` to find Pods that belong to the ReplicaSet.
3. If too few matching Pods exist, it creates Pods from `spec.template`.
4. If too many matching Pods exist, it removes Pods until the desired count is reached.
5. It repeats this reconciliation process whenever the cluster state changes.

In this example, the ReplicaSet wants three Pods and selects Pods with the label `app: nginx`.

## Components of a ReplicaSet

### `apiVersion`

```yaml
apiVersion: apps/v1
```

Specifies the Kubernetes API group and version used to interpret the resource. `apps/v1` is the stable API version for ReplicaSets.

### `kind`

```yaml
kind: ReplicaSet
```

Identifies the Kubernetes resource type. This tells the API server to create a ReplicaSet.

### `metadata`

```yaml
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
```

Contains information that identifies and organizes the ReplicaSet.

- `metadata.name` gives the ReplicaSet a unique name within its namespace.
- `metadata.labels` adds labels to the ReplicaSet itself. These labels help organize and query resources, but they do not select the Pods.

For example:

```powershell
kubectl get replicasets -l app=nginx
```

### `spec`

The `spec` describes the desired state of the ReplicaSet: how many Pods to maintain, which Pods it owns, and how new Pods should be created.

### `spec.replicas`

```yaml
replicas: 3
```

Sets the desired number of Pods. The ReplicaSet continuously works to keep three matching Pods running.

You can change the count with:

```powershell
kubectl scale replicaset nginx-replicaset --replicas=5
```

### `spec.selector`

```yaml
selector:
  matchLabels:
    app: nginx
```

Defines which Pods the ReplicaSet manages. The selector must match the labels in the Pod template.

A selector that is too broad can cause a ReplicaSet to manage Pods that it did not create, so selectors should be specific and unique.

### `spec.template`

```yaml
template:
```

Defines the Pod template used to create replacement Pods. It contains the Pod metadata and Pod specification.

#### `template.metadata.labels`

```yaml
template:
  metadata:
    labels:
      app: nginx
```

Adds labels to every Pod created by the ReplicaSet. These labels must satisfy `spec.selector.matchLabels`.

The key relationship is:

```text
spec.selector.matchLabels == spec.template.metadata.labels
```

The template can contain additional labels, but it must include every label required by the selector.

#### `template.spec`

Defines the containers and other runtime settings for each Pod created by the ReplicaSet.

#### `containers`

```yaml
containers:
  - name: nginx
    image: nginx:latest
```

Lists the containers that run inside each Pod. Every Pod must have at least one container.

#### Container `name`

```yaml
name: nginx
```

Names the container. The name must be unique within the Pod.

#### Container `image`

```yaml
image: nginx:latest
```

Specifies the container image to run. A fixed image tag, such as `nginx:1.27.1`, is generally more predictable than `latest`.

#### `ports.containerPort`

```yaml
ports:
  - containerPort: 80
```

Documents the port used by the container. Declaring `containerPort` does not expose the application outside the Pod. A Service is needed for stable network access.

## ReplicaSet and Deployment Differences

| Feature            | ReplicaSet                                                               | Deployment                                                    |
| ------------------ | ------------------------------------------------------------------------ | ------------------------------------------------------------- |
| Main purpose       | Maintains a desired number of identical Pods                             | Manages application releases and the ReplicaSets behind them  |
| Pod maintenance    | Creates replacement Pods when needed                                     | Creates and manages ReplicaSets, which maintain the Pods      |
| Scaling            | Supports manual scaling                                                  | Supports manual scaling and integrates with rollout workflows |
| Rolling updates    | Does not provide a rollout strategy by itself                            | Supports rolling updates when the Pod template changes        |
| Rollbacks          | No built-in revision history or rollback workflow                        | Supports rollout history and rollback to an earlier revision  |
| Version management | Changing the template does not provide Deployment-style release tracking | Tracks revisions as new ReplicaSets                           |
| Typical use        | Low-level replication or learning                                        | Recommended resource for running stateless applications       |

The relationship is:

```text
Deployment
    |
    +-- ReplicaSet
            |
            +-- Pod
            +-- Pod
            +-- Pod
```

A Deployment normally creates a new ReplicaSet when its Pod template changes. During a rolling update, the Deployment increases the new ReplicaSet and decreases the old ReplicaSet according to its update strategy. This is why applications are generally deployed with a Deployment rather than a standalone ReplicaSet.

## ReplicaSet Versus Deployment Example

A standalone ReplicaSet can maintain three Pods:

```yaml
kind: ReplicaSet
spec:
  replicas: 3
```

However, changing the image directly on the ReplicaSet does not provide a controlled rollout of existing Pods. A Deployment is better for that workflow because it creates a new ReplicaSet, gradually replaces Pods, and records the revision for a possible rollback.

Use a ReplicaSet directly when you specifically need its replication behavior. Use a Deployment for most long-running, stateless applications.

## Important Notes

- A ReplicaSet does not provide stable network access. Use a Service to reach its Pods.
- A ReplicaSet selects Pods by labels; it does not select them by Pod name.
- The selector and Pod template labels must be compatible.
- Deleting a ReplicaSet normally deletes the Pods it owns. Use `--cascade=orphan` when you need to leave the Pods running.
- A ReplicaSet is not a replacement for application-level high availability. The Pods may still be placed on the same worker node unless scheduling rules are configured.
