# Module 9 — Pods, Deployments and Services

## Learning Objectives

By the end of this module, students should be able to:

- Explain what a Pod is and why Kubernetes uses Pods.
- Understand the relationship between Pods, Deployments, ReplicaSets, and Services.
- Create and manage Pods using YAML and kubectl.
- Create Deployments with multiple replicas.
- Understand how Kubernetes replaces failed Pods.
- Create Services and understand ClusterIP, NodePort, and LoadBalancer.
- Understand `port` vs `targetPort`.
- Use labels and selectors to connect Services to Pods.
- Troubleshoot common Pod, Deployment, and Service problems.
- Deploy a simple application and test service discovery.
- Answer common Kubernetes interview questions.

---

# 1. The Big Picture

A useful beginner mental model is:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
    |
    v
Containers
```

A Service provides stable networking to the Pods:

```text
                 +----------------+
                 |   Deployment   |
                 +-------+--------+
                         |
                         v
                 +---------------+
                 |  ReplicaSet   |
                 +-------+-------+
                         |
              +----------+----------+
              |          |           |
              v          v           v
           +-----+    +-----+     +-----+
           | Pod |    | Pod |     | Pod |
           +-----+    +-----+     +-----+
              ^          ^           ^
              |          |           |
              +----------+-----------+
                         |
                     Service
                         |
                       Client
```

> A Deployment manages Pods, while a Service provides a stable network endpoint for reaching Pods.

---

# 2. What Is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod contains one or more containers that share:

- Network namespace
- Pod IP
- Storage volumes
- Lifecycle

The most common pattern is:

```text
Pod
└── Application Container
```

A Pod can also contain multiple cooperating containers:

```text
Pod
├── Application container
└── Sidecar container
```

For beginners, think:

> One Pod usually represents one application workload.

This is a mental model, not a strict rule. Kubernetes allows multiple tightly coupled containers in one Pod.

---

# 3. Real-World Analogy — Pod

Imagine a hotel room.

The hotel room is the **Pod**.

Inside the room you may have:

```text
Room
├── Person A
└── Person B
```

They share the same room and address.

Similarly, containers inside the same Pod share the Pod's network namespace and can communicate using `localhost`.

Example:

```text
Container A → localhost:8080
Container B → localhost:9090
```

---

# 4. Creating Your First Pod

Create `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Check:

```bash
kubectl get pods
```

Inspect:

```bash
kubectl describe pod nginx-pod
kubectl get pod nginx-pod -o wide
kubectl logs nginx-pod
```

Execute a shell:

```bash
kubectl exec -it nginx-pod -- /bin/bash
```

If bash is unavailable:

```bash
kubectl exec -it nginx-pod -- /bin/sh
```

Delete:

```bash
kubectl delete pod nginx-pod
```

---

# 5. Why Don't We Usually Create Pods Directly?

Suppose you manually create:

```text
Pod A
Pod B
Pod C
```

If Pod B disappears, a standalone Pod does not provide the replica-management behavior you normally want for a production stateless application.

Kubernetes provides controllers such as:

- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob

For typical stateless applications:

> Deployment is the common choice.

---

# 6. What Is a Deployment?

A **Deployment** is a Kubernetes controller that manages a desired state for a set of Pods.

For example:

```yaml
replicas: 3
```

means Kubernetes should maintain three matching Pods.

The simplified hierarchy is:

```text
Deployment
    |
    v
ReplicaSet
    |
    +---- Pod
    +---- Pod
    +---- Pod
```

Deployments also support:

- Rolling updates
- Rollbacks
- Scaling
- Declarative application management

---

# 7. Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Inspect:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

---

# 8. Understanding the Deployment YAML

## `apiVersion`

```yaml
apiVersion: apps/v1
```

Specifies the Kubernetes API version for the Deployment.

## `kind`

```yaml
kind: Deployment
```

Specifies the resource type.

## `metadata`

```yaml
metadata:
  name: nginx-deployment
```

Defines metadata such as the resource name.

## `replicas`

```yaml
replicas: 3
```

Defines the desired number of replicas.

## `selector`

```yaml
selector:
  matchLabels:
    app: nginx
```

Identifies the Pods managed by the Deployment.

## Pod template

```yaml
template:
  metadata:
    labels:
      app: nginx
```

Defines the Pods that the Deployment creates.

The selector must match the labels in the Pod template.

---

# 9. Labels

Labels are key-value metadata attached to Kubernetes objects.

Example:

```yaml
labels:
  app: nginx
  environment: production
  team: platform
```

Query by label:

```bash
kubectl get pods -l app=nginx
```

Labels are heavily used by:

- Services
- Deployments
- Selectors
- Scheduling
- Resource organization

---

# 10. What Is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of matching Pods are running.

For example:

```text
Desired: 3
Current: 2

ReplicaSet:
Create 1 Pod
```

Or:

```text
Desired: 3
Current: 3

No change required
```

The normal relationship is:

```text
Deployment
    |
    v
ReplicaSet
    |
    +---- Pod
    +---- Pod
    +---- Pod
```

You normally manage the Deployment rather than directly managing the ReplicaSet.

---

# 11. Self-Healing Lab

Deploy:

```bash
kubectl apply -f deployment.yaml
```

List Pods:

```bash
kubectl get pods
```

Delete one:

```bash
kubectl delete pod <pod-name>
```

Watch:

```bash
kubectl get pods -w
```

The ReplicaSet notices that the current number is below the desired number and creates a replacement.

Conceptually:

```text
Desired replicas = 3
        |
        v
Pod deleted
        |
        v
Current replicas = 2
        |
        v
ReplicaSet creates replacement
        |
        v
Current replicas = 3
```

---

# 12. Scaling a Deployment

Scale to five:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Check:

```bash
kubectl get pods
```

Scale down:

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

Declaratively, change:

```yaml
spec:
  replicas: 5
```

and run:

```bash
kubectl apply -f deployment.yaml
```

---

# 13. What Is a Service?

Pods are ephemeral.

Their IP addresses can change when Pods are replaced.

Example:

```text
Pod A → 10.244.1.10
Pod B → 10.244.1.11
Pod C → 10.244.2.10
```

After Pod A is replaced:

```text
New Pod → 10.244.1.15
```

Applications should therefore generally not depend on individual Pod IPs.

A **Service** provides a stable network endpoint for a group of Pods.

---

# 14. Real-World Service Analogy

Imagine a restaurant.

Customers don't call the chef's personal phone.

They call the restaurant's main number:

```text
Customer
   |
   v
Restaurant phone number
   |
   +---- Chef A
   +---- Chef B
   +---- Chef C
```

The restaurant number stays stable even when employees change.

Similarly:

```text
Client
   |
   v
Service
   |
   +---- Pod A
   +---- Pod B
   +---- Pod C
```

---

# 15. Service Selectors

Pods:

```yaml
metadata:
  labels:
    app: nginx
```

Service:

```yaml
selector:
  app: nginx
```

The Service selects Pods whose labels match the selector.

```text
Service selector
       |
       v
Pod labels
```

This relationship is one of the most important concepts in Kubernetes networking.

---

# 16. ClusterIP Service

`ClusterIP` is the default Service type and is primarily used for internal cluster communication.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80

  type: ClusterIP
```

Apply:

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get service
```

---

# 17. `port` vs `targetPort`

Consider:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Flow:

```text
Client
   |
   | :80
   v
Service
   |
   | :8080
   v
Pod
```

### `port`

The port exposed by the Service.

### `targetPort`

The port on the selected Pod to which traffic is sent.

They can be the same:

```yaml
port: 80
targetPort: 80
```

Or different:

```yaml
port: 80
targetPort: 8080
```

---

# 18. EndpointSlices

Kubernetes uses EndpointSlices to track the backend endpoints associated with Services.

Inspect:

```bash
kubectl get endpoints
kubectl get endpointslices
```

Conceptually:

```text
Service
   |
   v
EndpointSlice
   |
   +---- Pod IP 1
   +---- Pod IP 2
   +---- Pod IP 3
```

If a Service has no usable endpoints, investigate:

- Service selector
- Pod labels
- Pod readiness
- EndpointSlices
- NetworkPolicy
- CNI/networking

---

# 19. Testing a ClusterIP Service

Run a temporary client Pod:

```bash
kubectl run curl-pod   --image=curlimages/curl   --rm   -it   -- sh
```

Inside:

```bash
curl http://nginx-service
```

You should receive the nginx response.

You do not need to know the individual Pod IP.

---

# 20. Kubernetes DNS

Services receive DNS names inside the cluster.

For:

```text
Service: nginx-service
Namespace: default
```

A full DNS name is commonly:

```text
nginx-service.default.svc.cluster.local
```

Inside the same namespace:

```bash
curl http://nginx-service
```

usually works.

Kubernetes DNS is commonly provided by **CoreDNS**.

Check:

```bash
kubectl get pods -n kube-system
```

---

# 21. NodePort

A NodePort exposes a Service through a port on each node.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx

  type: NodePort

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f nodeport.yaml
```

Check:

```bash
kubectl get service nginx-nodeport
```

Conceptually:

```text
External Client
      |
      v
NodeIP:30080
      |
      v
Service
      |
      v
Pods
```

---

# 22. LoadBalancer

A Service can use:

```yaml
type: LoadBalancer
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  selector:
    app: nginx

  type: LoadBalancer

  ports:
    - port: 80
      targetPort: 80
```

On a cloud provider, the Kubernetes/cloud integration can provision or integrate with an external load-balancing resource.

Conceptually:

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Kubernetes Service
   |
   +---- Pod
   +---- Pod
   +---- Pod
```

The exact AWS behavior depends on the EKS configuration and AWS load-balancing integration being used.

---

# 23. Service Types

| Type | Typical purpose |
|---|---|
| ClusterIP | Internal cluster communication |
| NodePort | Expose through node ports |
| LoadBalancer | Integrate with external/cloud load balancing |
| ExternalName | DNS-based mapping to an external name |

Default:

```yaml
type: ClusterIP
```

---

# 24. Complete Deployment + Service Example

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web

  ports:
    - port: 80
      targetPort: 80

  type: ClusterIP
```

Apply:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Inspect:

```bash
kubectl get deployment
kubectl get replicasets
kubectl get pods
kubectl get service
kubectl get endpoints
kubectl get endpointslices
```

---

# 25. Break-and-Fix Service Lab

First inspect labels:

```bash
kubectl get pods --show-labels
```

Pods should have:

```text
app=web
```

Change the Service selector from:

```yaml
selector:
  app: web
```

to:

```yaml
selector:
  app: wrong-app
```

Apply:

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get endpoints web-service
```

There should be no matching backend endpoints.

Why?

```text
Service selector:
app=wrong-app

Pod label:
app=web
```

Fix:

```yaml
selector:
  app: web
```

Apply again:

```bash
kubectl apply -f service.yaml
```

Then:

```bash
kubectl get endpoints web-service
kubectl get endpointslices
```

---

# 26. Rolling Updates

Change the image:

```yaml
image: nginx:1.27
```

to another explicit version, for example:

```yaml
image: nginx:1.28
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Watch:

```bash
kubectl rollout status deployment/web
kubectl get pods -w
```

Kubernetes gradually replaces old Pods with new Pods according to the Deployment's rollout strategy.

---

# 27. Rollback

View history:

```bash
kubectl rollout history deployment/web
```

Rollback:

```bash
kubectl rollout undo deployment/web
```

Check:

```bash
kubectl rollout status deployment/web
```

---

# 28. Useful Deployment Commands

```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl describe deployment nginx
kubectl scale deployment nginx --replicas=5
kubectl rollout status deployment nginx
kubectl rollout history deployment nginx
kubectl rollout undo deployment nginx
kubectl rollout restart deployment nginx
kubectl delete deployment nginx
```

---

# 29. Useful Service Commands

```bash
kubectl get services
kubectl describe service web-service
kubectl get endpoints
kubectl get endpointslices
kubectl delete service web-service
```

---

# 30. AWS + Kubernetes Connection

A common EKS workflow is:

```text
Dockerfile
    |
    v
Docker Image
    |
    v
Amazon ECR
    |
    v
EKS Deployment
    |
    v
Pods
    |
    v
Kubernetes Service
```

A simplified external architecture:

```text
                    Internet
                       |
                       v
              AWS Load Balancer
                       |
                       v
               Kubernetes Service
                       |
          +------------+------------+
          |            |            |
          v            v            v
       Pod 1         Pod 2        Pod 3
```

The exact AWS implementation depends on whether you use a LoadBalancer Service, Ingress, and the relevant AWS load-balancing controller/integration.

---

# 31. Docker vs Kubernetes Mental Model

Docker:

```bash
docker run nginx
```

You directly ask Docker to run a container.

Kubernetes:

```bash
kubectl apply -f deployment.yaml
```

You declare the desired Kubernetes state.

Kubernetes then manages:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
    ↓
Containers
```

---

# 32. Mini Project — Kubernetes Web Application

## Objective

Deploy a simple highly available web application.

Requirements:

- 1 Deployment
- 3 Pods
- 1 ClusterIP Service
- Service-to-Pod communication
- Self-healing
- Scaling
- Service troubleshooting

## Step 1 — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3

  selector:
    matchLabels:
      app: web-app

  template:
    metadata:
      labels:
        app: web-app

    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

## Step 2 — Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app

  ports:
    - port: 80
      targetPort: 80
```

## Step 3 — Deploy

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## Step 4 — Verify

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
kubectl get service
kubectl get endpoints
```

## Step 5 — Test

```bash
kubectl run test-client   --image=curlimages/curl   --rm   -it   -- sh
```

Inside:

```bash
curl http://web-app-service
```

## Step 6 — Test Self-Healing

```bash
kubectl delete pod <pod-name>
kubectl get pods -w
```

Verify that the Deployment returns to three replicas.

## Step 7 — Scale

```bash
kubectl scale deployment web-app --replicas=5
kubectl get pods
```

## Step 8 — Break the Service

Change:

```yaml
selector:
  app: web-app
```

to:

```yaml
selector:
  app: broken
```

Apply and inspect:

```bash
kubectl get endpoints web-app-service
```

Fix the selector and verify the endpoints return.

---

# 33. Troubleshooting Guide

## Pod is Pending

```bash
kubectl describe pod <pod-name>
```

Check Events.

Possible causes:

- Insufficient resources
- Scheduling constraints
- Node taints
- No suitable nodes

## CrashLoopBackOff

```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

Possible causes:

- Application crashes
- Incorrect command
- Missing configuration
- Missing environment variables
- Dependency unavailable

## ImagePullBackOff

```bash
kubectl describe pod <pod-name>
```

Possible causes:

- Wrong image name
- Wrong image tag
- Private registry authentication issue
- Registry/network issue

## Service Has No Endpoints

```bash
kubectl get pods --show-labels
kubectl describe service <service-name>
kubectl get endpoints <service-name>
kubectl get endpointslices
```

Common cause:

```text
Service selector != Pod labels
```

## Service Exists but Application Is Not Reachable

Check:

```text
1. Pod running?
2. Pod ready?
3. Service exists?
4. Selector correct?
5. Endpoints exist?
6. targetPort correct?
7. Application listening on that port?
8. NetworkPolicy blocking traffic?
9. CNI/networking healthy?
```

---

# 34. Common Beginner Mistakes

## Mistake 1 — Treating Pod IP as permanent

Pods are replaceable. Use Services for stable connectivity.

## Mistake 2 — Confusing `port` and `targetPort`

Remember:

```text
Service port → targetPort → Pod
```

## Mistake 3 — Incorrect Service selector

Example:

```yaml
selector:
  app: backend
```

while Pods have:

```yaml
labels:
  app: api
```

The Service won't select those Pods.

## Mistake 4 — Thinking Deployment directly runs containers

Simplified hierarchy:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pod
   ↓
Container
```

## Mistake 5 — Thinking `containerPort` exposes a Pod externally

This:

```yaml
ports:
  - containerPort: 80
```

documents the intended container port. It does not by itself create a Service or external exposure.

## Mistake 6 — Using `latest` everywhere

For reproducible deployments, prefer explicit tags or immutable image digests.

Example:

```yaml
image: nginx:1.27
```

rather than:

```yaml
image: nginx:latest
```

---

# 35. Interview Questions and Answers

## Q1. What is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It contains one or more containers that share networking and storage resources and are scheduled together.

## Q2. Why does Kubernetes use Pods?

Pods provide a higher-level execution unit for one or more tightly coupled containers and define their shared networking, storage, and lifecycle context.

## Q3. What is a Deployment?

A Deployment manages a desired state for stateless application Pods and provides replica management, rolling updates, and rollback functionality.

## Q4. What is a ReplicaSet?

A ReplicaSet maintains a specified number of matching Pod replicas.

## Q5. What happens when a Deployment-managed Pod is deleted?

The ReplicaSet associated with the Deployment detects that the current replica count is below the desired count and creates a replacement Pod.

## Q6. Why do we need Services?

Pods are ephemeral and their IPs can change. A Service provides a stable endpoint and selects the appropriate backend Pods.

## Q7. What is ClusterIP?

ClusterIP is the default Service type and provides an internal stable virtual endpoint for accessing selected Pods.

## Q8. What is NodePort?

NodePort exposes a Service through a port on each node.

## Q9. What is a LoadBalancer Service?

It requests integration with an external load-balancing mechanism, such as a cloud provider's load balancer.

## Q10. What is the difference between `port` and `targetPort`?

`port` is the Service port. `targetPort` is the destination port on the selected Pod.

## Q11. How does a Service find Pods?

It uses a label selector. Matching Pods become Service endpoints, subject to readiness and other endpoint conditions.

## Q12. Can a Pod have multiple containers?

Yes. Multiple containers can share the Pod's network namespace and volumes.

## Q13. Does deleting a Deployment delete its Pods?

Normally, yes. The Deployment owns its ReplicaSets, which in turn own the Pods, so deleting the Deployment normally results in its managed workload being removed.

## Q14. Is a Service the same thing as a load balancer?

No. A Service is a Kubernetes abstraction providing stable access to a set of Pods. Some Service types can integrate with an external load balancer.

## Q15. Why are labels important?

Labels allow Kubernetes selectors to identify and group resources.

## Q16. What happens if a Service selector matches no Pods?

The Service has no matching backend endpoints, so requests cannot be delivered to the intended Pods.

## Q17. How do you scale a Deployment?

```bash
kubectl scale deployment <name> --replicas=5
```

or update the YAML and apply it.

## Q18. How do you rollback a Deployment?

```bash
kubectl rollout undo deployment <name>
```

## Q19. What is a rolling update?

A rolling update gradually replaces old application Pods with new Pods according to the Deployment rollout strategy.

## Q20. What is EndpointSlice?

EndpointSlice is a Kubernetes API resource used to represent network endpoints associated with a Service in scalable groups.

---

# 36. Interview Scenario

Suppose:

```text
Deployment:
replicas = 3

Service selector:
app=backend
```

Pods:

```text
Pod 1 → app=backend
Pod 2 → app=backend
Pod 3 → app=frontend
```

Question:

How many Pods are selected?

Answer:

```text
2 Pods
```

Only Pods with:

```text
app=backend
```

match the Service selector.

---

# 37. Another Interview Scenario

Given:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

and the application listens on:

```text
8080
```

Client traffic flows:

```text
Client
  |
  | :80
  v
Service
  |
  | :8080
  v
Pod
```

The client uses the Service port; the Service forwards to the target port.

---

# 38. Important Mental Model

Remember:

```text
POD
↓
Runs application container(s)

REPLICASET
↓
Maintains desired Pod count

DEPLOYMENT
↓
Manages ReplicaSets and application rollouts

SERVICE
↓
Provides stable networking to Pods
```

And:

```text
Service
   |
   | selector
   v
Pod labels
   |
   v
Endpoints / EndpointSlices
   |
   v
Pod IP + targetPort
```

---

# 39. Homework

## Beginner

1. Create an nginx Pod.
2. Inspect its logs.
3. Execute a shell inside it.
4. Delete the Pod.
5. Create an nginx Deployment with three replicas.
6. Scale it to five.
7. Scale it back to two.

## Intermediate

1. Create a Deployment with three Pods.
2. Create a ClusterIP Service.
3. Test the Service from another Pod.
4. Change the Service selector.
5. Observe the failure.
6. Fix the selector.
7. Inspect EndpointSlices.

## Advanced

Create:

```text
Frontend Deployment
    |
Frontend Service
    |
Backend Deployment
    |
Backend Service
```

Requirements:

- Frontend: 2 replicas
- Backend: 3 replicas
- Services use selectors
- Test frontend → backend communication using the backend Service DNS name
- Delete a backend Pod and verify replacement
- Scale backend from 3 to 5 replicas
- Perform a rolling image update
- Roll back the update

---

# 40. Module Summary

The core relationships are:

```text
Deployment
     |
     v
ReplicaSet
     |
     v
Pods
     ^
     |
Service
```

The core networking relationship is:

```text
Service
   |
   | selector
   v
Pod labels
   |
   v
EndpointSlices
   |
   v
Pod IP + targetPort
```

If you understand these relationships, you have the foundation for:

- Pod lifecycle
- Service discovery
- Load balancing
- Deployments
- Scaling
- Kubernetes networking
- Ingress
- EKS application deployment

---

# Quick Revision Cheat Sheet

| Concept | Remember |
|---|---|
| Pod | Smallest deployable Kubernetes unit |
| Container | Runs inside a Pod |
| ReplicaSet | Maintains desired Pod count |
| Deployment | Manages ReplicaSets and rollouts |
| Service | Stable endpoint for Pods |
| ClusterIP | Internal Service |
| NodePort | Exposes Service through node ports |
| LoadBalancer | Integrates with external/cloud load balancing |
| Label | Key-value metadata |
| Selector | Finds matching resources |
| `port` | Service port |
| `targetPort` | Pod destination port |
| EndpointSlice | Represents Service backend endpoints |
| CoreDNS | Provides Kubernetes DNS |
| Rolling update | Gradually updates application Pods |
| Rollback | Returns a Deployment to an earlier revision |

---

# Next Module

## Module 10 — Pod Lifecycle and Networking

Topics will include:

- Pod lifecycle phases
- Pod conditions
- Container states
- Restart policies
- Init containers
- Liveness, readiness, and startup probes
- Pod IPs
- Pod-to-Pod communication
- Container-to-container communication
- Networking inside Pods
- Practical debugging
- AWS/EKS networking connection
- Interview scenarios
