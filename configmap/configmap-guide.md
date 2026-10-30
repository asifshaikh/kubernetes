# Kubernetes ConfigMap Guide

## What is a ConfigMap?

A Kubernetes `ConfigMap` stores non-confidential configuration as key-value data. It separates settings from a container image so the same application image can be used in different environments with different configuration.

For example, an application might read its log level, feature flags, or the URL of an internal service from a ConfigMap rather than having those values built into its image.

A ConfigMap is an API object. It does not itself configure or restart an application: a Pod must refer to it through its specification, and the application must read the resulting environment variables or files.

Use ConfigMaps for values that are not secret, such as:

- Application settings and feature flags
- Service URLs and ports
- Non-sensitive environment-specific values
- Configuration files, such as `app.properties` or `nginx.conf`

Do not store passwords, API tokens, private keys, or other confidential data in a ConfigMap. Use a Secret or an external secret-management solution instead.

## A simple ConfigMap

The existing [config.yaml](./config.yaml) defines two string values:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: task-configmap
data:
  firstname: asif
  lastname: shaikh
```

Important fields:

- `apiVersion: v1` and `kind: ConfigMap` identify the Kubernetes API resource.
- `metadata.name` is the name Pods use to refer to this ConfigMap.
- `data` contains UTF-8 string values.

ConfigMaps are namespaced resources. A Pod can use a ConfigMap only from its own namespace. If `metadata.namespace` is omitted, `kubectl` uses the selected namespace (usually `default`).

## Ways to create a ConfigMap

### 1. Create it from a YAML manifest

This is the declarative approach and is usually best for configuration managed in source control:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
  namespace: default
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
  API_URL: "https://api.example.com"
```

Apply and inspect it:

```powershell
kubectl apply -f configmap.yaml
kubectl get configmap app-settings
kubectl describe configmap app-settings
kubectl get configmap app-settings -o yaml
```

Do not put real credentials in a manifest committed to source control.

### 2. Create it from literal values

`--from-literal` creates a key-value pair directly from the command line:

```powershell
kubectl create configmap app-settings `
  --from-literal=APP_MODE=production `
  --from-literal=LOG_LEVEL=info `
  --from-literal=API_URL=https://api.example.com
```

Each `--from-literal` adds one key. Quote values when the shell or the value's characters require it. This command creates the object imperatively; it is not a YAML file.

### 3. Create it from one or more files

The file name becomes the key by default, and the file contents become the value:

```powershell
kubectl create configmap app-settings --from-file=app.properties
```

To choose the key explicitly:

```powershell
kubectl create configmap app-settings `
  --from-file=application.properties=.\config\application.properties
```

You can add multiple files by repeating `--from-file`:

```powershell
kubectl create configmap app-settings `
  --from-file=application.properties=.\config\application.properties `
  --from-file=logging.properties=.\config\logging.properties
```

The file contents are stored as a ConfigMap value. This is useful when an application expects a configuration file rather than individual environment variables.

### 4. Create it from a directory of files

```powershell
kubectl create configmap app-settings --from-file=.\config
```

`kubectl` adds eligible files in the directory as separate keys, using their file names as keys. Subdirectories are not recursively included. Check the resulting keys before relying on the ConfigMap:

```powershell
kubectl describe configmap app-settings
```

### 5. Create it from an environment file

An environment file contains `KEY=value` lines, for example `app.env`:

```text
APP_MODE=production
LOG_LEVEL=info
API_URL=https://api.example.com
```

Create a ConfigMap from it:

```powershell
kubectl create configmap app-settings --from-env-file=app.env
```

This parses the file into separate ConfigMap keys. It is not the same as storing the entire file under one key with `--from-file`.

### Choosing a creation method

- Use a YAML manifest when configuration should be reviewed, versioned, and applied repeatedly.
- Use `--from-literal` for quick experiments or small values.
- Use `--from-file` when a setting is maintained as a file.
- Use `--from-env-file` when starting with a `KEY=value` file.
- Imperative `kubectl create configmap` fails if the named ConfigMap already exists. For repeatable manifest-based updates, use `kubectl apply -f ...`.

## Using a ConfigMap in a Pod

The Pod and ConfigMap must be in the same namespace. The Pod specification decides how the configuration is exposed to the container.

### Use individual keys as environment variables

The repository's [pod.yaml](./pod.yaml) shows this approach:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: configmap-container
      image: busybox
      command: ["sh", "-c", "echo The app is running; sleep 3600"]
      env:
        - name: FIRSTNAME
          valueFrom:
            configMapKeyRef:
              name: task-configmap
              key: firstname
        - name: LASTNAME
          valueFrom:
            configMapKeyRef:
              name: task-configmap
              key: lastname
```

Here, the `firstname` key becomes the `FIRSTNAME` environment variable and `lastname` becomes `LASTNAME`.

To make a missing key optional, add `optional: true` under `configMapKeyRef`. By default, a required ConfigMap or key that is missing prevents the container from starting.

### Import all keys as environment variables

Use `envFrom` when each ConfigMap key should become an environment variable:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-envfrom-pod
spec:
  containers:
    - name: app
      image: nginx
      envFrom:
        - configMapRef:
            name: app-settings
          prefix: APP_
```

With the example `app-settings` ConfigMap, the container receives variables such as `APP_APP_MODE` and `APP_LOG_LEVEL`. Omit `prefix` if the original key names should be used. Keys that are not valid environment variable names may not be made available this way.

### Mount ConfigMap keys as files

A ConfigMap can be mounted as a read-only volume. Each key becomes a file in the mount directory, and its value is the file contents:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: app-config
          mountPath: /etc/app-config
          readOnly: true
  volumes:
    - name: app-config
      configMap:
        name: app-settings
```

If `app-settings` contains `application.properties`, the container can read `/etc/app-config/application.properties`.

To mount only selected keys, use `items`:

```yaml
volumes:
  - name: app-config
    configMap:
      name: app-settings
      items:
        - key: application.properties
          path: application.properties
```

The `path` is relative to the volume mount directory. A ConfigMap volume is read-only; it is not a place for the application to write persistent data.

### Mount a single file with `subPath`

To mount one ConfigMap key at a particular file path without replacing the whole directory:

```yaml
volumeMounts:
  - name: app-config
    mountPath: /etc/nginx/conf.d/default.conf
    subPath: nginx.conf
    readOnly: true
```

This requires a ConfigMap volume named `app-config` with a key named `nginx.conf`. A `subPath` mount does **not** receive updates when the ConfigMap changes.

## Updating and applying configuration changes

ConfigMaps are not automatically a rollout trigger for Deployments. How a change becomes visible depends on how it is consumed:

- **Environment variables:** Existing containers keep their original environment. Restart or roll out Pods to pick up changed values.
- **ConfigMap volume:** Kubernetes updates the projected files eventually; this is not instantaneous. The application must reread or reload the file to use the new contents.
- **`subPath` volume mount:** The mounted file does not receive ConfigMap updates. Recreate the Pod to use the new value.

For predictable deployments, update the workload's Pod template (for example, by changing a versioned ConfigMap name or adding a configuration checksum annotation) so Kubernetes creates new Pods. Avoid assuming that changing a ConfigMap alone restarts an application.

Example commands:

```powershell
kubectl apply -f configmap.yaml
kubectl rollout restart deployment/my-app
kubectl rollout status deployment/my-app
```

The rollout commands apply when the ConfigMap is consumed by a Deployment. For a standalone Pod, delete and recreate the Pod instead.

## Where ConfigMaps are used

ConfigMaps are referenced from workload Pod templates, including:

- Pods
- Deployments and ReplicaSets
- StatefulSets and DaemonSets
- Jobs and CronJobs

They are commonly used to provide application environment variables, mounted configuration files, feature toggles, and environment-specific endpoints. A ConfigMap has no effect until a workload references it.

## ConfigMap limits and operational notes

- A ConfigMap is limited to 1 MiB of data. Large configuration artifacts should generally be stored elsewhere and delivered using an appropriate mechanism.
- Keys in `data` are string values. Use `binaryData` for binary data that cannot be represented as UTF-8 text.
- ConfigMaps are namespaced; a Pod cannot directly reference one in another namespace.
- A ConfigMap does not provide application-level validation. The application still needs to handle invalid or missing configuration.
- ConfigMap contents are not confidential. Access should still be controlled with RBAC because configuration can reveal internal architecture or endpoints.
- `immutable: true` can prevent updates to a ConfigMap after creation. This is useful for fixed configuration, but changing it requires creating a replacement ConfigMap and updating consumers.

Immutable example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fixed-app-settings
immutable: true
data:
  LOG_LEVEL: "info"
```

## ConfigMap vs Secret

| Topic | ConfigMap | Secret |
|---|---|---|
| Intended content | Non-confidential configuration | Confidential values such as passwords, tokens, and keys |
| Typical consumption | Environment variables or mounted files | Environment variables or mounted files |
| Data representation | `data` contains strings; `binaryData` supports binary content | `data` values are base64-encoded in API representations; `stringData` accepts plain strings on write |
| Confidentiality | Not designed to protect sensitive data | Intended for sensitive data, but base64 encoding alone is not encryption |
| Size | Limited to 1 MiB | Also limited to 1 MiB |

Example Secret (illustrative only; do not use a real password in a checked-in manifest):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  username: app-user
  password: replace-this-example
```

The API stores Secret values encoded in its representation, but base64 is encoding, not encryption. Configure encryption at rest for the Kubernetes datastore, restrict access with least-privilege RBAC, avoid exposing secret values in logs or source control, and consider an external secret manager for production credentials.

Use ConfigMaps and Secrets consistently: normal application settings belong in ConfigMaps; credentials and other sensitive values belong in Secrets. Both can be mounted or injected into Pods, but they serve different data-sensitivity needs.

## End-to-end example

Save the ConfigMap and Pod examples in this guide as `configmap.yaml` and `pod.yaml`, respectively, then run:

```powershell
kubectl apply -f configmap.yaml
kubectl apply -f pod.yaml
kubectl get configmap task-configmap
kubectl get pod configmap-pod
kubectl logs configmap-pod
```

To inspect the environment variables inside the example container:

```powershell
kubectl exec configmap-pod -- sh -c 'printf "%s %s\n" "$FIRSTNAME" "$LASTNAME"'
```

Expected output:

```text
asif shaikh
```

Clean up the resources when finished:

```powershell
kubectl delete -f pod.yaml
kubectl delete -f configmap.yaml
```

## Interview questions and answers

### 1. What is a ConfigMap?

A ConfigMap is a namespaced Kubernetes API object for storing non-confidential configuration as key-value pairs. Pods can consume its values as environment variables or files.

### 2. Does creating a ConfigMap change an application automatically?

No. A workload must reference the ConfigMap, and the application must read the injected environment variable or mounted file.

### 3. What are common ways to create a ConfigMap?

Create one from a YAML manifest, literal values, individual files, a directory of files, or an environment file using `kubectl create configmap`.

### 4. What is the difference between `--from-file` and `--from-env-file`?

`--from-file` stores file content under a key (usually the file name). `--from-env-file` parses `KEY=value` lines into separate ConfigMap keys.

### 5. How can a Pod consume a ConfigMap?

Use `configMapKeyRef` for a specific environment variable, `envFrom` for many keys, or a ConfigMap volume to expose keys as files.

### 6. Do ConfigMaps update running containers?

Environment variables do not change in existing containers. Mounted volume files are updated eventually, but the application must reread them. A `subPath` mount does not receive updates. Restarting or rolling out Pods is a common way to ensure new configuration is consumed.

### 7. Can a Pod use a ConfigMap in another namespace?

No. ConfigMaps and Pods are namespaced, and a Pod references a ConfigMap in its own namespace.

### 8. What is the main difference between a ConfigMap and a Secret?

They are both ways to provide data to workloads, but ConfigMaps are for non-sensitive values and Secrets are intended for confidential values. Secrets need appropriate encryption-at-rest and access controls; base64 encoding is not encryption.

### 9. Is a Secret automatically safe because it is base64 encoded?

No. Base64 is reversible encoding. Use encryption at rest, restrictive RBAC, and careful handling of Secret values.

### 10. What is the maximum size of a ConfigMap?

The combined data size is limited to 1 MiB. Larger content should use another storage or configuration-delivery approach.

### 11. What happens if a required ConfigMap or key is missing?

The Pod cannot successfully start the container that requires it. A reference can be marked optional when that behavior is appropriate.

### 12. What does `immutable: true` do?

It prevents the ConfigMap's data from being changed after creation. To change the configuration, create a replacement ConfigMap and update the workload reference.
