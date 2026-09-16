# Multi-Container Pods

A **multi-container Pod** contains two or more containers that are scheduled on the same Kubernetes node and managed as one unit. Containers in the same Pod share:

- The Pod's network namespace and IP address. They can reach each other through `localhost`.
- Volumes mounted by more than one container.
- The Pod lifecycle and scheduling decision.

Each container still has its own filesystem, process space, image, environment, and resource settings unless a resource or volume is explicitly shared.

Use a multi-container Pod when the containers are tightly coupled and must run together. If two components need independent scaling, independent rollout timing, or independent failure domains, use separate Pods instead.

## Basic Structure

The `containers` field contains the main, long-running containers. An additional container can share a volume or communicate over `localhost`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  containers:
    - name: writer
      image: busybox:1.36
      command:
        ['sh', '-c', 'while true; do date >> /data/events.log; sleep 5; done']
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ['sh', '-c', 'tail -f /data/events.log']
      volumeMounts:
        - name: shared-data
          mountPath: /data
```

Apply and inspect it with:

```bash
kubectl apply -f shared-volume-pod.yaml
kubectl get pod shared-volume-pod
kubectl logs shared-volume-pod -c reader
kubectl delete pod shared-volume-pod
```

`emptyDir` exists for the lifetime of the Pod. It is useful for temporary communication or scratch data, but it is not durable storage.

## What Is an Init Container?

An **init container** is a container that runs before the regular application containers. Init containers run sequentially and must complete successfully before the next init container, or the application containers, can start.

They are useful for one-time preparation tasks such as:

- Waiting for a required Service or external dependency to become reachable.
- Creating directories or files in a shared volume.
- Downloading configuration or generating certificates.
- Running a database migration or validating configuration.
- Registering the Pod with another system before the application starts.

### Real Example: Wait for a Dependency

The following Pod waits until the Kubernetes Service named `myservice` can be resolved. This is similar to the `multi-container-pod/pod.yaml` example in this directory:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-example
spec:
  initContainers:
    - name: wait-for-service
      image: busybox:1.36
      command: ['sh', '-c']
      args:
        - >-
          until nslookup myservice.default.svc.cluster.local;
          do echo "waiting for myservice"; sleep 2; done
  containers:
    - name: application
      image: busybox:1.36
      command: ['sh', '-c', 'echo Application started; sleep 3600']
```

Check init-container progress with:

```bash
kubectl get pod init-container-example
kubectl describe pod init-container-example
kubectl logs init-container-example -c wait-for-service
```

If an init container fails, Kubernetes restarts it according to the Pod's restart policy. The application containers do not start until all init containers succeed. An init container should therefore have a bounded, observable task; waiting forever can keep the Pod stuck in `Init` status.

### Real Example: Prepare a Shared Volume

This example creates configuration before the web server starts:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prepared-config-pod
spec:
  volumes:
    - name: generated-config
      emptyDir: {}
  initContainers:
    - name: generate-config
      image: busybox:1.36
      command: ['sh', '-c']
      args:
        - echo 'server { listen 8080; }' > /config/default.conf
      volumeMounts:
        - name: generated-config
          mountPath: /config
  containers:
    - name: web
      image: nginx:1.27
      volumeMounts:
        - name: generated-config
          mountPath: /etc/nginx/conf.d
```

The init container and the web container use the same volume, so the generated file is available when NGINX starts.

## What Is a Sidecar Container?

A **sidecar container** is a regular container that runs alongside the main application container in the same Pod. It extends or supports the application without being the application's primary process.

Common sidecar responsibilities include:

- Shipping application logs to a logging system.
- Providing a proxy for traffic management, mTLS, retries, or telemetry.
- Refreshing configuration or certificates.
- Exposing metrics or transforming data for another process.
- Synchronizing files between a shared volume and an external system.

Unlike an init container, a traditional sidecar stays running while the main application runs. Kubernetes also supports native sidecar containers through `restartPolicy: Always` on an init container in newer Kubernetes versions, but ordinary sidecars in `spec.containers` remain the most broadly understood pattern.

### Real Example: Log-Shipper Sidecar

The application writes logs to a shared volume. The sidecar reads the same file and could forward each line to a logging service:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-sidecar-example
spec:
  volumes:
    - name: app-logs
      emptyDir: {}
  containers:
    - name: application
      image: busybox:1.36
      command: ['sh', '-c']
      args:
        - >-
          i=0; while true; do
          echo "application event $i" >> /var/log/app/events.log;
          i=$((i + 1)); sleep 5; done
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
    - name: log-shipper
      image: busybox:1.36
      command: ['sh', '-c', 'tail -F /var/log/app/events.log']
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
```

In production, replace `tail` with a real log collector configured to send records to the organization's logging backend. A sidecar adds CPU, memory, image, and operational overhead to every Pod replica, so a node-level logging agent is often more efficient when the same collection behavior applies to many applications.

## Sidecar vs. Init Container

| Characteristic    | Init container                                        | Sidecar container                                          |
| ----------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| Start time        | Before application containers                         | Alongside application containers                           |
| Expected behavior | Completes a preparation task                          | Usually keeps running                                      |
| Ordering          | Init containers run in order                          | Regular containers do not provide general startup ordering |
| Typical purpose   | Setup, validation, migration, dependency wait         | Proxy, logging, monitoring, synchronization                |
| Failure effect    | Application containers cannot start until it succeeds | Depends on Pod behavior and the sidecar's role             |
| Example           | Generate config in a shared volume                    | Read and ship application logs                             |

## When Should You Use a Multi-Container Pod?

Use one when:

- The processes must share the same network identity or localhost communication.
- The processes must share a volume with low-latency access.
- One process is a helper for exactly one application instance.
- The helper and application must be scheduled, scaled, and deployed together.
- A setup task must finish before the application starts.

Avoid one when:

- Components need different replica counts, autoscaling rules, or release schedules.
- One component could consume resources and starve the other.
- The helper can be shared by many Pods and is better deployed as a DaemonSet or separate Service.
- The components have independent availability requirements or should fail independently.
- The design only combines containers to avoid creating a proper Service or Deployment.

## Operational Notes

### Networking

All containers in a Pod share one IP address and port space. If one container listens on port `8080`, another container in the same Pod can call it through `http://localhost:8080`. They cannot both bind the same port.

### Resources

The Pod's scheduling needs are based on the sum of the containers' resource requests. Set requests and limits for every container, including sidecars, so scheduling and capacity planning reflect the real workload.

### Logs and Debugging

Container logs are separate even though the containers share a Pod:

```bash
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -c <container-name> --previous
kubectl describe pod <pod-name>
```

### Deployments

In real applications, put the Pod template under a Deployment, StatefulSet, or Job rather than creating a standalone Pod. This provides rollout, replacement, and replica management while preserving the multi-container design inside each Pod.

## Quick Decision Guide

Ask these questions:

1. Does the task need to happen before the application starts and then stop? Use an **init container**.
2. Does a helper need to run for the application's entire lifetime? Consider a **sidecar**.
3. Must the helper share the Pod's IP, localhost, or a volume? A **multi-container Pod** may fit.
4. Does the helper need independent scaling, rollout, or availability? Use a **separate workload** instead.
