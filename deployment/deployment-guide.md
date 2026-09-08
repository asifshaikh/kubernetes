# Kubernetes Deployment YAML Guide

## What Is a Deployment?

A Kubernetes `Deployment` manages replicated Pods. It ensures that:

- The desired number of Pods are running.
- Failed Pods are recreated.
- Application versions can be updated gradually.
- You can roll back to an earlier version.

This Deployment runs three Nginx Pods.

## Creating and Applying a Deployment

Create a file named `deployment.yaml` and add a Deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

Apply it to the cluster:

```powershell
kubectl apply -f deployment.yaml
```

Inspect the result:

```powershell
kubectl get deployments
kubectl get pods
kubectl describe deployment nginx-deployment
```

Kubernetes reads the YAML and sends the desired configuration to the Kubernetes API server.

## Field-by-Field Explanation

### `apiVersion`

```yaml
apiVersion: apps/v1
```

Specifies the Kubernetes API version and schema used by the resource. `apps/v1` is the stable API version for Deployments.

### `kind`

```yaml
kind: Deployment
```

Specifies the type of Kubernetes object. This tells Kubernetes to create a Deployment.

### `metadata.name`

```yaml
metadata:
  name: nginx-deployment
```

Gives the Deployment its name. The name must be unique within its namespace.

### `metadata.labels`

```yaml
metadata:
  labels:
    app: nginx
```

Adds searchable metadata. Labels can organize and filter resources:

```powershell
kubectl get deployments -l app=nginx
```

Labels under `metadata` are useful but are not always strictly required.

### `spec`

The `spec` defines the desired state of the Deployment: how many Pods should run and how those Pods should be created.

### `spec.replicas`

```yaml
replicas: 3
```

Requests three identical Pods. Kubernetes continuously works to maintain this number. If one Pod fails, Kubernetes creates a replacement.

This field is optional; the usual default is one replica.

### `spec.selector`

```yaml
selector:
  matchLabels:
    app: nginx
```

Tells the Deployment which Pods it owns. The selector must match the labels in the Pod template.

### `spec.template`

```yaml
template:
```

Defines the Pod template. The Deployment uses this template whenever it creates Pods.

### `template.metadata.labels`

```yaml
template:
  metadata:
    labels:
      app: nginx
```

Adds labels to every Pod created by this Deployment. These labels must match `spec.selector.matchLabels`.

The important relationship is:

```text
spec.selector.matchLabels == spec.template.metadata.labels
```

### `template.spec`

Defines the contents and behavior of each Pod.

### `containers`

```yaml
containers:
  - name: nginx
```

Lists the containers inside the Pod. A Pod must contain at least one container.

### Container `name`

```yaml
name: nginx
```

Names the container. Container names must be unique inside the Pod.

### Container `image`

```yaml
image: nginx:1.14.2
```

Specifies the container image and tag:

```text
Repository: nginx
Tag: 1.14.2
```

Using a specific version is more predictable than using `latest`.

### `ports.containerPort`

```yaml
ports:
  - containerPort: 80
```

Documents that the container listens on port 80. This does not expose the application outside the cluster. A Kubernetes Service is needed for that.

## Required and Optional Fields

A practical minimum Deployment is:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx:1.25
```

Usually required:

- `apiVersion`
- `kind`
- `metadata.name`
- `spec.selector`
- `spec.template`
- `spec.template.metadata.labels`
- `spec.template.spec.containers`
- Container `name`
- Container `image`

Common optional fields:

- `replicas`
- `ports`
- `resources`
- `env`
- `volumeMounts`
- `livenessProbe`
- `readinessProbe`
- `strategy`
- `securityContext`

## How to Remember the Structure

Remember the hierarchy:

```text
API
  Kind
    Metadata
      Name and labels
    Spec
      Replica count
      Selector
      Pod template
        Pod metadata
        Pod spec
          Containers
            Name
            Image
            Ports
```

A useful phrase is:

> Who am I, what do I want, and what should my Pods contain?

- `metadata`: Who am I?
- `spec.replicas`: How many Pods do I want?
- `selector`: Which Pods do I manage?
- `template`: How should new Pods be created?
- `containers`: What application runs inside them?

## Rolling Updates

Deployments use `RollingUpdate` by default. Kubernetes gradually replaces old Pods with new Pods.

You can configure it explicitly:

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

- `maxUnavailable: 1`: At most one desired Pod can be unavailable during the update.
- `maxSurge: 1`: Kubernetes can temporarily create one extra Pod during the update.

With three replicas, Kubernetes can temporarily run four Pods during the rollout.

Change the image in `deployment.yaml`:

```yaml
image: nginx:1.25
```

Apply and monitor the update:

```powershell
kubectl apply -f deployment.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -w
```

View rollout history:

```powershell
kubectl rollout history deployment/nginx-deployment
```

Undo the latest update:

```powershell
kubectl rollout undo deployment/nginx-deployment
```

Pause and resume a rollout:

```powershell
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment
```

## Recreate Strategy

The `Recreate` strategy stops all existing Pods before creating new ones:

```yaml
spec:
  strategy:
    type: Recreate
```

This causes downtime, but it is useful when old and new versions cannot run at the same time.

Use:

- `RollingUpdate` for highly available applications.
- `Recreate` when simultaneous versions would cause conflicts.

## Useful Deployment Features

### Environment Variables

```yaml
env:
  - name: APP_ENV
    value: production
```

### Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: '100m'
    memory: '128Mi'
  limits:
    cpu: '500m'
    memory: '256Mi'
```

Requests help Kubernetes schedule Pods. Limits restrict maximum resource usage.

### Readiness Probe

A readiness probe determines whether a Pod is ready to receive traffic:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Liveness Probe

A liveness probe determines whether a container should be restarted:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 15
  periodSeconds: 20
```

### Progress Deadline

```yaml
progressDeadlineSeconds: 600
```

Marks the rollout as failed if it does not make progress within ten minutes.

### Revision History

```yaml
revisionHistoryLimit: 5
```

Keeps the last five old ReplicaSets so that previous versions can be inspected or restored.

## Scaling the Number of Pods

Scale a Deployment immediately with the `kubectl scale` command:

```powershell
kubectl scale deployment/nginx-deployment --replicas=5
```

This changes the Deployment to five Pods. Check the result with:

```powershell
kubectl get deployment nginx-deployment
kubectl get pods
```

For a declarative change, update the `replicas` value in `deployment.yaml`:

```yaml
spec:
  replicas: 5
```

Then apply the file:

```powershell
kubectl apply -f deployment.yaml
```

The command-line method is useful for a quick or temporary change. Updating the YAML is better when the new replica count should be saved as the desired configuration and reproduced later.

## Typical Deployment Workflow

```powershell
kubectl apply -f deployment.yaml
kubectl get deployment
kubectl get replicasets
kubectl get pods
kubectl rollout status deployment/nginx-deployment
```

Update an image directly:

```powershell
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
```

Troubleshoot a Deployment or Pod:

```powershell
kubectl describe deployment nginx-deployment
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

## The Main Kubernetes Objects

```text
Deployment  -> manages the desired application version
ReplicaSet  -> maintains the required number of Pods
Pod         -> runs the container
Container   -> runs the application process
```
