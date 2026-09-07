# Kubernetes Pod YAML Guide

## How a `pod.yaml` Is Created

A Kubernetes Pod manifest is a YAML file that describes the desired state of a Pod. Create a file such as `pod.yaml`, add the resource definition, and submit it to the Kubernetes API server:

```bash
kubectl apply -f pod.yaml
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Useful commands:

```bash
kubectl get pods
kubectl describe pod nginx
kubectl delete -f pod.yaml
```

## Field-by-Field Explanation

### `apiVersion`

```yaml
apiVersion: v1
```

Specifies which Kubernetes API version should interpret the manifest. A basic Pod uses `v1`.

This field is required because Kubernetes needs to know which API schema and behavior apply to the resource.

### `kind`

```yaml
kind: Pod
```

Identifies the type of Kubernetes object being created. In this example, the object is a `Pod`.

This field is required because Kubernetes needs to know what resource to create.

### `metadata`

```yaml
metadata:
  name: nginx
```

Contains information used to identify and organize the object.

#### `metadata.name`

```yaml
name: nginx
```

Gives the Pod its name. The name must be unique within its namespace and can be used with commands such as:

```bash
kubectl get pod nginx
kubectl describe pod nginx
kubectl delete pod nginx
```

Metadata can also contain labels and annotations:

```yaml
metadata:
  name: nginx
  labels:
    app: nginx
    environment: development
```

Labels are commonly used by Services, Deployments, and selectors.

### `spec`

```yaml
spec:
```

Describes the desired state of the Pod. It tells Kubernetes what the Pod should contain and how it should run.

The specification can include containers, volumes, environment variables, resource limits, restart policies, networking settings, and security settings.

### `spec.containers`

```yaml
containers:
```

Defines the containers that should run inside the Pod. A Pod must contain at least one container.

The `-` indicates an item in a YAML list. A Pod can contain multiple containers, and containers in the same Pod share the Pod's network namespace.

### `containers.name`

```yaml
- name: nginx
```

Names the container inside the Pod. The name must be unique within the Pod and is useful when viewing logs:

```bash
kubectl logs nginx -c nginx
```

The container name is required for each container.

### `containers.image`

```yaml
image: nginx:latest
```

Specifies the container image Kubernetes should run. In this example, `nginx` is the image name and `latest` is the tag.

The image is normally pulled from a container registry such as Docker Hub. A fixed version is generally safer for production:

```yaml
image: nginx:1.27.1
```

The image field is required because Kubernetes needs to know what program and filesystem to run.

### `containers.ports`

```yaml
ports:
  - containerPort: 80
```

Documents that the container listens on port `80`. The `ports` field is optional, and declaring it does not expose the Pod outside the cluster.

To provide stable network access, create a Kubernetes Service. For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

## Required and Optional Fields

| Field              |           Required? | Purpose                               |
| ------------------ | ------------------: | ------------------------------------- |
| `apiVersion`       |                 Yes | Selects the Kubernetes API schema     |
| `kind`             |                 Yes | Identifies the resource type          |
| `metadata`         |                 Yes | Identifies the object                 |
| `metadata.name`    | Yes for a named Pod | Gives the Pod its name                |
| `spec`             |                 Yes | Defines the desired state             |
| `spec.containers`  |                 Yes | Defines at least one container        |
| `containers.name`  |                 Yes | Names the container                   |
| `containers.image` |                 Yes | Specifies the image to run            |
| `containers.ports` |                  No | Documents ports used by the container |
| `containerPort`    |                  No | Documents a port number               |

The smallest practical Pod manifest is:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27.1
```

## How to Remember the Structure

Remember the main structure as:

> A Pod has an API version, a kind, metadata, and a specification.

Use this sequence:

> **A-K-M-S-C-N-I-P**

- **A**: `apiVersion`
- **K**: `kind`
- **M**: `metadata`
- **S**: `spec`
- **C**: `containers`
- **N**: container `name`
- **I**: container `image`
- **P**: container `ports`

A sentence to help remember it is:

> **API Knows Metadata; Specification Contains Named Images and Ports.**

The nesting looks like this:

```text
Pod
├── apiVersion
├── kind
├── metadata
│   └── name
└── spec
    └── containers
        ├── name
        ├── image
        └── ports
            └── containerPort
```

You can also remember the manifest by asking four questions:

1. **Which API understands this?** `apiVersion`
2. **What am I creating?** `kind`
3. **What should it be called?** `metadata.name`
4. **What should run inside it?** `spec.containers`

> Important: `containerPort: 80` does not expose Nginx to the outside world. A Kubernetes `Service` is normally required for stable network access.
