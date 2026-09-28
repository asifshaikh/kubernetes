# Module 10 — Pod Lifecycle and Networking

## Learning Objectives

By the end of this module, students should be able to:

- Explain the Kubernetes Pod lifecycle.
- Understand Pod phases and container states.
- Explain restart policies and CrashLoopBackOff.
- Understand Init Containers.
- Understand startup, readiness, and liveness probes.
- Explain Pod IPs and Pod networking.
- Explain communication between containers in the same Pod.
- Explain Pod-to-Pod and Service networking.
- Understand Kubernetes DNS and CoreDNS.
- Troubleshoot common lifecycle and networking problems.
- Understand the AWS/EKS networking connection.
- Answer common Pod lifecycle and networking interview questions.

---

# 1. What Happens to a Pod?

A Pod does not simply have two states such as Running and Stopped.

A simplified lifecycle is:

```text
Pod Created
     |
     v
Pending
     |
     v
Running
     |
     +------------------+
     |                  |
     v                  v
Succeeded           Failed
```

The important distinction is:

> A Pod phase describes the overall Pod state, while container states describe what individual containers are doing.

---

# 2. Pod Lifecycle Mental Model

Think about a food delivery order:

```text
Order Created
     |
     v
Restaurant Preparing
     |
     v
Out for Delivery
     |
     +----------+
     |          |
     v          v
Delivered   Cancelled
```

Similarly:

```text
Pod Created
     |
     v
Pending
     |
     v
Running
     |
     +----------+
     |          |
     v          v
Succeeded    Failed
```

Kubernetes continuously observes the workload and takes actions according to the desired state.

---

# 3. Pod Phases

The main Pod phases are:

- `Pending`
- `Running`
- `Succeeded`
- `Failed`
- `Unknown`

## Pending

The Pod has been accepted by Kubernetes but is not yet fully running.

Possible reasons:

- Waiting for scheduling
- Image is being pulled
- Waiting for resources
- Volumes are being prepared
- Scheduling constraints prevent placement

Check:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Pay particular attention to:

```text
Events
```

## Running

A Pod is `Running` when it has been scheduled to a node and its containers have been started.

Important:

> Running does not necessarily mean the application is ready to receive traffic.

A Pod can be:

```text
Running
```

while its application is still starting or not ready.

## Succeeded

A Pod enters `Succeeded` when all containers terminate successfully and will not be restarted.

This is common with Jobs and batch workloads.

## Failed

A Pod enters `Failed` when its containers have terminated and at least one terminated unsuccessfully, or the Pod was terminated in a failure condition.

Investigate:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

## Unknown

`Unknown` means Kubernetes cannot obtain reliable information about the Pod's state.

Possible causes include node failure, kubelet failure, or communication problems between the node and control plane.

---

# 4. Pod Phase vs Container State

## Pod Phase

```text
Pending
Running
Succeeded
Failed
Unknown
```

## Container State

A container can be:

```text
Waiting
Running
Terminated
```

Example:

```text
Pod
 |
 +-- Phase: Running
 |
 +-- Container
       |
       +-- State: Running
```

Do not confuse Pod phase with container state.

---

# 5. Container States

## Waiting

The container has not started running yet.

Examples:

```text
ImagePullBackOff
ErrImagePull
CrashLoopBackOff
ContainerCreating
```

Inspect:

```bash
kubectl describe pod <pod-name>
```

## Running

The container is currently executing.

## Terminated

The container has finished execution.

It may have:

```text
Exit Code: 0
```

or a non-zero exit code.

For example:

```text
Reason: Completed
Exit Code: 0
```

means successful termination.

---

# 6. Restart Policy

Pod restart policies include:

```text
Always
OnFailure
Never
```

Example:

```yaml
restartPolicy: OnFailure
```

## Always

Containers are restarted when they terminate.

## OnFailure

Restart when the container terminates unsuccessfully.

## Never

Do not restart the container after termination.

For many long-running workloads managed by Deployments, the effective Pod restart policy is `Always`.

---

# 7. Restart Policy Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-demo
spec:
  restartPolicy: OnFailure

  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Running..."
          exit 1
```

Apply:

```bash
kubectl apply -f restart-demo.yaml
```

Check:

```bash
kubectl get pod restart-demo
```

You may see the restart count increase because the process exits unsuccessfully.

---

# 8. CrashLoopBackOff

A common Kubernetes status is:

```text
CrashLoopBackOff
```

It means a container is repeatedly failing and Kubernetes is backing off before attempting another restart.

Typical causes:

- Application crashes
- Incorrect command
- Missing environment variable
- Missing configuration
- Dependency unavailable
- Incorrect file path
- Application exits immediately

Troubleshoot:

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```

The `--previous` option is especially useful when the container has already restarted.

---

# 9. Practical CrashLoopBackOff Lab

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Application starting"
          sleep 5
          echo "Application crashed"
          exit 1
```

Apply:

```bash
kubectl apply -f crash-demo.yaml
```

Watch:

```bash
kubectl get pod crash-demo -w
```

Inspect:

```bash
kubectl logs crash-demo
kubectl logs crash-demo --previous
kubectl describe pod crash-demo
```

---

# 10. Init Containers

An Init Container runs before the main application containers.

Architecture:

```text
Pod
 |
 +-- Init Container
 |
 +-- Init Container
 |
 +-- Application Container
```

Init Containers must complete successfully before normal containers start.

Common uses:

- Wait for a dependency
- Download configuration
- Prepare files
- Run initialization logic
- Perform setup tasks

---

# 11. Init Container Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Initializing..."
          sleep 5
          echo "Initialization complete"

  containers:
    - name: app
      image: nginx:1.27
```

Apply:

```bash
kubectl apply -f init-demo.yaml
```

Inspect:

```bash
kubectl get pod init-demo
kubectl describe pod init-demo
kubectl logs init-demo -c init
```

Analogy:

```text
Prepare kitchen
      |
      v
Check equipment
      |
      v
Prepare ingredients
      |
      v
Open restaurant
```

The preparation steps are like Init Containers.

---

# 12. Startup, Readiness and Liveness Probes

There are three important health probes:

```text
Startup Probe
Readiness Probe
Liveness Probe
```

They answer different questions.

### Startup Probe

> Has the application finished starting?

### Readiness Probe

> Is the application ready to receive traffic?

### Liveness Probe

> Is the application still healthy enough to continue running?

---

# 13. Why Probes Matter

Imagine a Java application that takes 60 seconds to start.

The container may technically be running after a few seconds, but the application may not be ready.

Without readiness handling:

```text
Service
   |
   +---- New Pod
           |
           | Application still starting
           |
           X
```

With readiness handling:

```text
Service
   |
   +---- Ready Pod
   +---- Ready Pod

New Pod
   |
   | Not Ready
   |
   X  <-- no normal Service traffic
```

---

# 14. Readiness Probe

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

If the probe fails, Kubernetes marks the Pod as not ready.

The Pod can remain:

```text
Running
```

while:

```text
READY = 0/1
```

and it is normally removed from ready Service endpoints.

---

# 15. Liveness Probe

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
```

If the liveness probe repeatedly fails, kubelet can restart the affected container.

---

# 16. Startup Probe

Startup probes are useful for slow-starting applications.

Example:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

A common conceptual flow is:

```text
Startup probe
      |
      v
Application starts
      |
      v
Readiness determines traffic eligibility
      |
      v
Liveness monitors ongoing health
```

---

# 17. Probe Types

## HTTP

```yaml
httpGet:
  path: /health
  port: 8080
```

## TCP

```yaml
tcpSocket:
  port: 8080
```

## Command

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

Choose the probe type appropriate for the application.

---

# 18. Complete Probe Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-demo
spec:
  replicas: 2

  selector:
    matchLabels:
      app: health-demo

  template:
    metadata:
      labels:
        app: health-demo

    spec:
      containers:
        - name: app
          image: nginx:1.27

          ports:
            - containerPort: 80

          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30
            periodSeconds: 5

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
```

Apply:

```bash
kubectl apply -f health-demo.yaml
```

Inspect:

```bash
kubectl describe pod <pod-name>
```

---

# 19. Important Probe Mistake

Do not use an aggressive liveness probe to determine application readiness.

If an application takes 90 seconds to start but liveness checking starts after 5 seconds, Kubernetes may restart the application before it can finish starting.

Use:

```text
Startup probe → slow startup
Readiness probe → traffic eligibility
Liveness probe → ongoing health
```

---

# 20. Pod Networking

Every Pod receives its own IP address in a typical Kubernetes cluster networking model.

Example:

```text
Pod A → 10.244.1.10
Pod B → 10.244.1.11
Pod C → 10.244.2.10
```

A major Kubernetes networking principle is:

> Pods should be able to communicate with other Pods without requiring NAT between their Pod IPs in the normal cluster networking model.

The exact implementation depends on the cluster's CNI.

---

# 21. Pod IPs Are Ephemeral

Suppose:

```text
Pod A
IP = 10.244.1.10
```

Pod A is deleted.

A replacement may receive:

```text
Pod A'
IP = 10.244.1.15
```

Therefore:

```text
Do NOT build application discovery around individual Pod IPs.
```

Use:

```text
Service
```

for stable application connectivity.

---

# 22. Communication Between Containers in the Same Pod

Containers inside the same Pod share the Pod's network namespace.

Example:

```text
Pod IP = 10.244.1.10

Container A → port 8080
Container B → port 9090
```

Container A can reach Container B using:

```text
localhost:9090
```

and Container B can reach Container A using:

```text
localhost:8080
```

provided the applications are listening appropriately.

Important:

> Containers in the same Pod share the same network namespace, so they cannot independently bind the same IP/port combination.

---

# 23. Pod-to-Pod Communication

Suppose:

```text
Pod A
10.244.1.10

Pod B
10.244.2.20
```

Pod A can communicate with Pod B using its Pod IP in a correctly configured cluster network.

Conceptually:

```text
Pod A
10.244.1.10
    |
    | network
    v
Pod B
10.244.2.20
```

The CNI plugin implements the cluster networking model.

Examples:

- Amazon VPC CNI
- Calico
- Cilium
- Flannel

---

# 24. Pod-to-Pod Communication Across Nodes

Example:

```text
Node 1
  |
  +-- Pod A
      10.244.1.10

Node 2
  |
  +-- Pod B
      10.244.2.20
```

Traffic:

```text
Pod A
  |
  v
Node networking / CNI
  |
  v
Node 2 networking / CNI
  |
  v
Pod B
```

The exact implementation depends on the CNI.

In Amazon EKS, the Amazon VPC CNI integrates Pod networking with the AWS VPC networking model.

---

# 25. Service Networking

A Service provides a stable virtual endpoint.

```text
Client Pod
    |
    | http://backend-service
    v
Service
    |
    +------ Pod A
    +------ Pod B
    +------ Pod C
```

The client does not need to know individual Pod IPs.

---

# 26. DNS and Service Discovery

Suppose:

```text
Service:
backend-service
```

A frontend can use:

```text
http://backend-service
```

instead of:

```text
http://10.244.1.20
```

If the backend Service is in another namespace:

```text
http://backend-service.backend.svc.cluster.local
```

This makes application-to-application communication stable.

---

# 27. Network Troubleshooting Lab

Deploy a backend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2

  selector:
    matchLabels:
      app: backend

  template:
    metadata:
      labels:
        app: backend

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend

  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f backend.yaml
kubectl apply -f backend-service.yaml
```

Test:

```bash
kubectl run network-test   --image=curlimages/curl   --rm   -it   -- sh
```

Inside:

```bash
curl http://backend-service
```

---

# 28. Test DNS

Inside the test Pod:

```bash
nslookup backend-service
```

Then:

```bash
nslookup backend-service.default.svc.cluster.local
```

And:

```bash
curl http://backend-service.default.svc.cluster.local
```

If `nslookup` is not available in the image, use another debugging image or suitable DNS tools.

---

# 29. Break Service Connectivity

Change the Service selector:

```yaml
selector:
  app: wrong
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

DNS may still resolve the Service name.

This demonstrates:

> DNS resolution and application connectivity are different troubleshooting layers.

Check:

```bash
kubectl get endpoints backend-service
kubectl get endpointslices
```

---

# 30. Service Troubleshooting Workflow

When:

```bash
curl http://backend-service
```

fails, investigate:

```text
1. Does the Pod exist?
       |
       v
2. Is the Pod Running?
       |
       v
3. Is the Pod Ready?
       |
       v
4. Does the Service exist?
       |
       v
5. Does the selector match Pod labels?
       |
       v
6. Does the Service have endpoints?
       |
       v
7. Is targetPort correct?
       |
       v
8. Is the application listening?
       |
       v
9. Does DNS resolve?
       |
       v
10. Is NetworkPolicy blocking traffic?
       |
       v
11. Is the CNI/networking layer healthy?
```

---

# 31. Useful Networking Commands

Pods:

```bash
kubectl get pods -o wide
```

Services:

```bash
kubectl get svc
```

Endpoints:

```bash
kubectl get endpoints
```

EndpointSlices:

```bash
kubectl get endpointslices
```

Pod labels:

```bash
kubectl get pods --show-labels
```

Pod details:

```bash
kubectl describe pod <pod-name>
```

Service details:

```bash
kubectl describe service <service-name>
```

CoreDNS:

```bash
kubectl get pods -n kube-system
```

Test from inside the cluster:

```bash
kubectl run network-test   --image=curlimages/curl   --rm   -it   -- sh
```

---

# 32. NetworkPolicy

NetworkPolicy can control which Pods are allowed to communicate.

Conceptually:

```text
Frontend
   |
   | allowed
   v
Backend
```

But:

```text
Random Pod
   |
   | denied
   v
Backend
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

This allows ingress to backend Pods from Pods labeled:

```text
app=frontend
```

NetworkPolicy behavior depends on whether the cluster networking implementation supports and enforces it.

---

# 33. AWS EKS Networking

A simplified EKS architecture is:

```text
AWS VPC
 |
 +---------------------------+
 |                           |
 |   EKS Cluster             |
 |                           |
 |  Node 1                   |
 |   +-- Pod                 |
 |   +-- Pod                 |
 |                           |
 |  Node 2                   |
 |   +-- Pod                 |
 |   +-- Pod                 |
 |                           |
 +---------------------------+
```

Important concepts include:

- VPC
- Subnets
- Security Groups
- Route Tables
- Elastic Network Interfaces
- Amazon VPC CNI
- EKS worker nodes
- Services
- Load balancers

---

# 34. Why AWS Networking Matters

A Kubernetes application may need to access:

```text
RDS
S3
ElastiCache
External APIs
```

The Pod's network path interacts with the AWS VPC architecture.

Conceptually:

```text
Pod
 |
 v
Node/VPC networking
 |
 +---- RDS
 |
 +---- AWS services
 |
 +---- Internet/NAT
```

The exact path depends on the AWS service and network configuration.

---

# 35. Mini Project — Frontend to Backend

Build:

```text
Frontend Deployment
        |
        v
Frontend Pods
        |
        | HTTP
        v
Backend Service
        |
        v
Backend Deployment
        |
   +----+----+
   |         |
   v         v
Backend   Backend
 Pod        Pod
```

Requirements:

- Frontend: 2 replicas
- Backend: 2 replicas
- Backend Service: ClusterIP
- Backend port: 80
- Test frontend/client → backend Service
- Use DNS instead of Pod IPs

Verify:

```bash
kubectl get pods -o wide
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```

---

# 36. Advanced Mini Project — Health-Aware Backend

Create a backend Deployment with:

- 3 replicas
- Readiness probe
- Liveness probe
- Startup probe
- ClusterIP Service

Architecture:

```text
Client
  |
  v
Backend Service
  |
  +---- Ready Pod 1
  +---- Ready Pod 2
  +---- Ready Pod 3
```

Experiment:

1. Deploy the application.
2. Confirm all Pods are Ready.
3. Make one Pod fail its readiness check.
4. Observe the Pod remain Running but become NotReady.
5. Check Service endpoints.
6. Observe that the unready Pod is removed from normal ready Service endpoints.
7. Restore health.
8. Observe it become Ready again.

This demonstrates the difference between:

```text
Running
```

and:

```text
Ready
```

---

# 37. Common Mistakes

## Mistake 1 — Assuming Running means Ready

A Pod can be Running while its application is not ready.

Use readiness probes.

## Mistake 2 — Using liveness as readiness

Liveness determines ongoing health. Readiness determines traffic eligibility.

## Mistake 3 — Forgetting startup probes

Slow applications can be restarted prematurely if liveness checks begin too early.

## Mistake 4 — Using Pod IPs directly

Pod IPs can change.

Use Services for stable application discovery.

## Mistake 5 — Assuming DNS failure whenever curl fails

DNS may resolve correctly while the Service has no endpoints.

Separate:

```text
DNS
Service
Endpoints
Application
NetworkPolicy
CNI
```

## Mistake 6 — Ignoring `--previous`

For restart loops:

```bash
kubectl logs <pod> --previous
```

is often extremely useful.

---

# 38. Interview Questions and Answers

## Q1. What are the main Pod phases?

`Pending`, `Running`, `Succeeded`, `Failed`, and `Unknown`.

## Q2. Difference between Pod phase and container state?

Pod phase describes the overall Pod lifecycle. Container state describes an individual container as `Waiting`, `Running`, or `Terminated`.

## Q3. What does CrashLoopBackOff mean?

A container is repeatedly failing and Kubernetes is backing off before another restart attempt.

## Q4. How do you debug CrashLoopBackOff?

```bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

## Q5. What is an Init Container?

A container that runs before the main application containers and must complete successfully before they start.

## Q6. What is a readiness probe?

It determines whether an application is ready to receive traffic.

## Q7. What is a liveness probe?

It determines whether a container is healthy enough to keep running; repeated failures can cause a restart.

## Q8. What is a startup probe?

It gives slow-starting applications time to initialize before normal liveness/readiness behavior is relied upon.

## Q9. Can a Pod be Running but not Ready?

Yes.

For example:

```text
STATUS: Running
READY: 0/1
```

## Q10. Can containers in the same Pod communicate using localhost?

Yes. Containers in the same Pod share a network namespace.

## Q11. Can Pods on different nodes communicate?

Yes, in a correctly configured cluster network, subject to the CNI and NetworkPolicy configuration.

## Q12. Why shouldn't applications use Pod IPs for service discovery?

Pod IPs are ephemeral and can change when Pods are replaced.

## Q13. What provides stable application discovery?

Kubernetes Services and DNS.

## Q14. What is NetworkPolicy?

A Kubernetes API mechanism for defining allowed Pod ingress and/or egress traffic, assuming the networking implementation supports and enforces it.

## Q15. What is the purpose of a CNI?

The Container Network Interface is the standard interface used by Kubernetes networking implementations to configure Pod/container networking.

## Q16. What is CoreDNS?

CoreDNS is commonly used to provide Kubernetes cluster DNS and Service discovery.

## Q17. What does `restartPolicy: OnFailure` mean?

The container can be restarted when it terminates unsuccessfully, subject to the Pod and workload context.

## Q18. Why is `kubectl logs --previous` useful?

It retrieves logs from the previous terminated container instance, which is valuable during restart loops.

## Q19. What happens when a Pod becomes NotReady?

It is normally removed from the set of ready Service endpoints used for ordinary Service traffic.

## Q20. Difference between readiness and liveness?

Readiness:

```text
Should this Pod receive traffic?
```

Liveness:

```text
Is this container healthy enough to keep running?
```

---

# 39. Interview Scenario — Running but Not Ready

Suppose:

```text
NAME       READY   STATUS
backend    0/1     Running
```

Is the application necessarily down?

Not necessarily.

The container may be running while its readiness probe is failing.

Investigate:

```bash
kubectl describe pod backend
```

Check:

```text
Conditions
Events
Readiness probe
```

---

# 40. Interview Scenario — CrashLoopBackOff

You see:

```text
backend   0/1   CrashLoopBackOff
```

Check:

```bash
kubectl logs backend
kubectl logs backend --previous
kubectl describe pod backend
```

Then investigate:

- Exit code
- Application error
- Environment variables
- Configuration
- Command/entrypoint
- Dependencies
- Probe configuration

---

# 41. Interview Scenario — Service Works by IP but Not Name

Suppose:

```bash
curl http://10.244.1.20
```

works, but:

```bash
curl http://backend-service
```

fails.

Investigate:

```bash
nslookup backend-service
kubectl get svc backend-service
kubectl get endpoints backend-service
kubectl get endpointslices
kubectl get pods -n kube-system
```

Possible causes:

- DNS problem
- Incorrect Service name
- Wrong namespace
- CoreDNS problem
- Service configuration problem

---

# 42. Practical Revision Exercise

Deploy a backend:

```text
Deployment:
replicas = 3
```

Add:

```text
startupProbe
readinessProbe
livenessProbe
```

Create:

```text
ClusterIP Service
```

Then:

1. Delete a Pod.
2. Observe replacement.
3. Scale to five replicas.
4. Break the readiness endpoint.
5. Observe `READY` change.
6. Check Service endpoints.
7. Restore the endpoint.
8. Break the application so it crashes.
9. Observe `CrashLoopBackOff`.
10. Use `kubectl logs --previous`.
11. Restore the application.
12. Test Service DNS.

---

# 43. Homework

## Beginner

1. Explain the five Pod phases.
2. Create a Pod with `restartPolicy: OnFailure`.
3. Create an intentionally failing Pod.
4. Observe `CrashLoopBackOff`.
5. Use `kubectl logs --previous`.
6. Create an Init Container.

## Intermediate

1. Create a Deployment with three replicas.
2. Add a readiness probe.
3. Add a liveness probe.
4. Add a startup probe.
5. Create a ClusterIP Service.
6. Test DNS from another Pod.
7. Inspect EndpointSlices.

## Advanced

Create:

```text
Frontend
   |
   v
Backend Service
   |
   v
Backend Pods
```

Requirements:

- Frontend: 2 replicas
- Backend: 3 replicas
- Backend readiness probe
- Backend liveness probe
- Backend startup probe
- ClusterIP Service
- DNS-based communication
- Network troubleshooting exercise
- Delete a backend Pod
- Break readiness
- Break liveness
- Observe the effects

---

# 44. Module Summary

Lifecycle:

```text
Pod Phase
   |
   +-- Pending
   +-- Running
   +-- Succeeded
   +-- Failed
   +-- Unknown
```

Container state:

```text
Waiting
Running
Terminated
```

Health checks:

```text
Startup
   ↓
Readiness
   ↓
Liveness
```

Networking:

```text
Pod
 |
 +-- Pod IP
 |
 +-- Containers share network namespace
 |
 +-- Communicates with other Pods
 |
 +-- Service provides stable endpoint
 |
 +-- DNS provides service discovery
 |
 +-- CNI implements networking
```

Troubleshooting:

```text
Pod status
    ↓
Container state
    ↓
Logs
    ↓
Events
    ↓
Readiness
    ↓
Service
    ↓
Endpoints
    ↓
DNS
    ↓
NetworkPolicy
    ↓
CNI / Node / Cloud networking
```

---

# Quick Revision Cheat Sheet

| Concept | Remember |
|---|---|
| Pending | Pod accepted but not fully running |
| Running | Pod scheduled and containers started |
| Succeeded | Containers completed successfully |
| Failed | Pod completed with failure |
| Unknown | Reliable Pod state unavailable |
| Waiting | Container has not started |
| Running | Container is executing |
| Terminated | Container has stopped |
| CrashLoopBackOff | Container repeatedly fails and restart attempts are backed off |
| Init Container | Runs before application containers |
| Startup Probe | Handles slow application startup |
| Readiness Probe | Determines traffic eligibility |
| Liveness Probe | Detects ongoing unhealthy containers |
| Pod IP | Ephemeral network identity |
| Service | Stable endpoint for Pods |
| CoreDNS | Cluster DNS/service discovery |
| CNI | Implements Pod networking |
| NetworkPolicy | Controls allowed Pod traffic |
| `kubectl logs --previous` | Logs from previous terminated container |
| `restartPolicy` | Controls container restart behavior at Pod level |

---

# Module 10 Complete

## Next Module — Module 11: ReplicaSets and Deployments

Topics:

- ReplicaSet architecture
- Desired vs current state
- Deployment controller behavior
- Rolling updates
- Recreate strategy
- `maxSurge`
- `maxUnavailable`
- Rollout history
- Rollbacks
- Revision management
- Deployment scaling
- Deployment troubleshooting
- Practical exercises
- AWS/EKS deployment patterns
- Interview scenarios
