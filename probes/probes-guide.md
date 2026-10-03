# Kubernetes Probes Guide

## What Is a Probe?

A probe is a health check that the kubelet runs against a container. It lets Kubernetes make decisions based on whether an application has started, is working, and can receive traffic. A container being in the `Running` state only means its process is running; it does not prove the application is ready to serve requests.

Probes are configured per container under `spec.containers[]` (or under a container in a workload's Pod template). They are not configured on a Service. The kubelet performs the check, while controllers and Services react to the resulting Pod/container status.

## Why Probes Are Used

Without probes, Kubernetes cannot reliably distinguish a healthy application from one that is still initializing, temporarily overloaded, deadlocked, or unable to reach a required dependency.

Probes help Kubernetes to:

1. Wait for slow-starting applications before starting normal health checks.
2. Restart a container that has become irrecoverably unhealthy.
3. Stop sending Service traffic to a Pod that cannot currently handle requests, without restarting it.
4. Keep a rollout from treating a newly created but not-yet-ready Pod as a usable replica.

For example, a web API may take 45 seconds to load a model after starting. A startup probe can allow that startup time. If the API later deadlocks, a liveness probe can trigger a container restart. If it temporarily loses access to its database, a readiness probe can remove it from traffic while leaving it running so that it can recover.

## The Three Probe Purposes

These are the three probe fields Kubernetes supports. They answer different questions and have different effects.

| Probe            | Question                                                        | If the check fails                                                                                                                                     |
| ---------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `startupProbe`   | Has this container finished starting?                           | Kubernetes keeps retrying; after the failure threshold, it restarts the container. Liveness and readiness checks are held back until startup succeeds. |
| `livenessProbe`  | Is this container still functioning, or should it be restarted? | After the failure threshold, kubelet restarts the container, subject to its restart policy.                                                            |
| `readinessProbe` | Can this container handle requests right now?                   | The Pod is marked not ready and is removed from the ready endpoints of matching Services. The container is not restarted just because readiness fails. |

All three are optional. A container may use one, two, or all three. Startup and liveness checks should test whether the application itself can make progress; do not make liveness depend on a remote database or other service. If a dependency is temporarily unavailable, restarting every application replica can make an outage worse. Readiness is usually the right place to report that the instance cannot currently serve traffic.

### Startup probe

Use a startup probe when initialization duration varies or takes longer than a normal liveness timeout. While the startup probe has not succeeded, Kubernetes does not run that container's liveness or readiness probes.

This example allows about five minutes for an application to start: `failureThreshold` is 30 and `periodSeconds` is 10. Once `/startupz` succeeds, the liveness and readiness checks become active.

```yaml
startupProbe:
  httpGet:
    path: /startupz
    port: http
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 30
```

The rough startup allowance is `failureThreshold * periodSeconds`. Actual timing can vary because the probe starts after the container starts and each check has its own execution time.

### Liveness probe

Use liveness to detect a failure from which the process cannot recover by itself, such as a deadlock. If checks fail consecutively up to `failureThreshold`, Kubernetes restarts the container.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

The endpoint should report whether the application process is functioning. It should generally not fail just because a downstream database is briefly unavailable.

### Readiness probe

Use readiness to control whether a container should receive traffic. A failing readiness check does not restart the container. The Pod remains running, but is not considered ready; Services stop routing new traffic to it while it is unready.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
  successThreshold: 1
```

The readiness endpoint can include checks needed to serve a request, such as whether required configuration has loaded or a critical dependency is available. Keep it fast and avoid making it so sensitive that small, harmless delays cause frequent traffic changes.

## Probe Mechanisms

The probe purpose (`startupProbe`, `livenessProbe`, or `readinessProbe`) determines what Kubernetes does with the result. The mechanism (`httpGet`, `tcpSocket`, `exec`, or `grpc`) determines how Kubernetes checks the container. Each probe uses exactly one mechanism.

### HTTP GET

Kubelet sends an HTTP GET request to the container. A response status from 200 through 399 is considered successful. Use a named container port or a numeric port. The application must listen on that port and implement the path.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
    scheme: HTTP
  periodSeconds: 5
```

`scheme` defaults to `HTTP`; set it to `HTTPS` when the endpoint uses TLS. HTTP probes can also set `httpHeaders`, for example to send a host header. Do not put secrets in probe headers: probe configuration is part of the Pod specification and can be visible to users with permission to read Pods.

**Real example:** A REST API exposes `/healthz` to report process health and `/ready` to report whether it has loaded configuration and can serve requests. Use `/healthz` for liveness and `/ready` for readiness.

### TCP socket

Kubelet attempts to open a TCP connection to the specified container port. A successful connection means the check succeeded. It does not verify that the application can process a request or that the protocol is healthy.

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

**Real example:** A legacy database has no health endpoint, but it listens on port `3306` only when its server process is accepting connections. A TCP check can test that listener. For a database with a reliable application-level health command, that command may provide a more meaningful check.

### Exec command

Kubelet runs a command inside the container. Exit code `0` means success; any other exit code means failure. The command runs in the container's environment, so the executable and script must exist in the image. Keep the command quick and lightweight because it runs repeatedly.

```yaml
readinessProbe:
  exec:
    command:
      - sh
      - -c
      - test -s /var/run/app/ready
  periodSeconds: 5
  timeoutSeconds: 2
```

**Real example:** A worker writes `/var/run/app/ready` after it has loaded its job configuration. The check succeeds only when the file exists and is non-empty. If the image is distroless and has no shell or `test` executable, this command will fail; use an HTTP, TCP, or gRPC check, or include an appropriate health-check binary.

### gRPC

Kubernetes can call the standard gRPC health checking protocol. The application must implement that protocol. The optional `service` distinguishes health checks, such as `api` for liveness and `readiness` for readiness. The port must be a numeric port; named ports, TLS settings, and authentication parameters are not supported by this probe mechanism.

```yaml
livenessProbe:
  grpc:
    port: 50051
    service: liveness
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

**Real example:** A gRPC order service implements the gRPC health checking protocol on port `50051`. It can report the process as healthy for liveness and report the `orders` service as serving only after its request handlers are ready.

## Complete Deployment Example

This Deployment demonstrates how the three purposes are commonly combined for a web API. The container must actually serve the shown endpoints on port `8080`; probe paths are application-specific.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
        - name: api
          image: example.com/orders-api:1.4.2
          ports:
            - name: http
              containerPort: 8080
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
            successThreshold: 1
```

The named port `http` is declared in `ports` and reused by the HTTP checks. The port declaration documents the container port and makes the name reusable; it does not expose the application outside the Pod. A Service can select the Deployment's ready Pods by label:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-api
spec:
  selector:
    app: orders-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

When one Pod's readiness check fails, it is excluded from ready Service endpoints. When its liveness check fails repeatedly, kubelet restarts its container. The Deployment maintains the requested replica count and creates replacement Pods when needed.

## Common Probe Settings

These timing and threshold fields can be set on a probe:

| Field                 | Meaning                                                                                                                                                      | Default |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------: |
| `initialDelaySeconds` | Seconds after the container starts before the first probe. A startup probe is often a better fit for slow startup.                                           |     `0` |
| `periodSeconds`       | Seconds between probe attempts.                                                                                                                              |    `10` |
| `timeoutSeconds`      | How long one probe may take before it is treated as failed.                                                                                                  |     `1` |
| `failureThreshold`    | Consecutive failures before the probe is considered failed. For liveness or startup, this leads to a restart; for readiness, it marks the container unready. |     `3` |
| `successThreshold`    | Consecutive successes needed to be considered successful again. Must be `1` for startup and liveness; readiness may use a larger value.                      |     `1` |

An optional `terminationGracePeriodSeconds` can control how long kubelet waits for a container to shut down after a liveness or startup failure before forcefully terminating it. If omitted, the Pod's termination grace period is used.

Do not tune intervals to be extremely frequent without a reason. Probe commands consume resources, and overly strict timeouts can mark a busy but functioning application unhealthy. Account for normal startup time and load when choosing thresholds.

## Where Probes Fit in Kubernetes

- **Pod templates:** In Deployments, StatefulSets, DaemonSets, Jobs, and other controllers, probes go inside each container in `spec.template.spec.containers`. For a standalone Pod, they go in `spec.containers`.
- **Kubelet and container lifecycle:** Kubelet runs checks for containers on its node. Startup and liveness outcomes can lead to a container restart according to the Pod's restart policy.
- **Services and traffic:** Readiness affects whether a Pod is included as a ready endpoint for Services selecting it. It does not itself create or expose a Service.
- **Rollouts:** Deployment controllers use Pod readiness when determining whether updated replicas are available. Readiness failures can delay rollout progress.
- **Not image health checks:** A Docker image's `HEALTHCHECK` is not a substitute for declaring Kubernetes probes in the Pod specification.

## Apply and Inspect

Apply a manifest and inspect the Pod:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

`kubectl describe pod` shows probe failures and recent events, such as connection refused, timeout, or a non-successful HTTP response. Check the container logs as well:

```bash
kubectl logs <pod-name> -c api
kubectl logs <pod-name> -c api --previous
```

Use `--previous` to inspect the logs from the prior container instance after a restart. To inspect Service endpoints and confirm readiness affects traffic:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=orders-api
```

## Common Mistakes

1. **Using the same check for every purpose:** A liveness check should detect a stuck process; a readiness check should detect whether requests can be served. They can use different endpoints.
2. **Checking a remote dependency with liveness:** A temporary database outage can trigger synchronized restarts and make recovery harder. Prefer readiness for dependencies that determine whether this instance can serve traffic.
3. **Using a URL or port the container does not serve:** The `path`, `port`, scheme, and protocol must match the running application. A `containerPort` declaration alone does not make a process listen there.
4. **Setting a short liveness delay for a slow application:** Use a startup probe to allow initialization, then let liveness detect failures after startup.
5. **Assuming readiness restarts containers:** Readiness controls traffic eligibility; it does not restart the container.
6. **Using an expensive exec check:** A command that performs a full database scan or other heavy work can add load every probe interval. Health checks should be bounded and fast.
7. **Ignoring probe events:** A failing probe is a symptom. Inspect events, logs, ports, paths, command availability, and timeout settings before increasing thresholds.

## Quick Mental Model

- **Startup:** Has initialization finished?
- **Liveness:** Should Kubernetes restart this container?
- **Readiness:** Should this Pod receive traffic right now?
- **Mechanism:** How does Kubernetes ask the question: HTTP, TCP, exec, or gRPC?
