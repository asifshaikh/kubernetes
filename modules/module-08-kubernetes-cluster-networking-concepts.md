# Module 8 — Kubernetes Cluster Networking Concepts

## Learning Objectives

By the end of this module, students should be able to:

- Explain why Kubernetes networking is required.
- Understand the Kubernetes networking model.
- Explain Pod IP addresses and Pod-to-Pod communication.
- Understand Services and Service types.
- Understand Kubernetes DNS and service discovery.
- Explain kube-proxy and CNI.
- Understand EndpointSlices.
- Understand basic NetworkPolicy concepts.
- Troubleshoot common Kubernetes networking problems.
- Connect Kubernetes networking with AWS VPC and EKS.
- Build and troubleshoot a small Kubernetes networking project.

---

# 1. Why Does Kubernetes Need Networking?

Imagine an application:

```text
Frontend
   |
   v
Backend
   |
   v
Database
```

Each component may run in different Pods:

```text
Frontend Pod
     |
     v
Backend Pod
     |
     v
Database Pod
```

These Pods need to communicate.

At the same time:

- Users need to access applications.
- Frontend needs to access backend.
- Backend needs to access database.
- Pods may be created and destroyed.
- Pods can move between Nodes.
- Applications need stable ways to find each other.

Therefore Kubernetes needs a networking model that handles dynamic workloads.

---

# 2. Real-World Analogy

Imagine a large apartment complex.

```text
Apartment Complex
 |
 +-- Building A
 |    +-- Apartment 101
 |    +-- Apartment 102
 |
 +-- Building B
      +-- Apartment 201
      +-- Apartment 202
```

Every apartment has an address.

Similarly:

```text
Cluster
 |
 +-- Node 1
 |    +-- Pod A -> Pod IP
 |    +-- Pod B -> Pod IP
 |
 +-- Node 2
      +-- Pod C -> Pod IP
      +-- Pod D -> Pod IP
```

But Pods are temporary.

A Pod might have:

```text
Old Pod -> 10.244.1.5
```

and its replacement might have:

```text
New Pod -> 10.244.1.8
```

Therefore applications should generally not depend on a specific Pod IP.

This is where **Services** become important.

---

# 3. Kubernetes Networking Model

Kubernetes follows several important networking principles:

1. Each Pod receives its own IP address.
2. Pods can communicate with other Pods through the cluster network.
3. Nodes can communicate with Pods.
4. Containers within the same Pod share the Pod network namespace.
5. Kubernetes Services provide stable endpoints for groups of Pods.

Conceptually:

```text
Pod A
10.244.1.10
     |
     | Cluster Network
     |
     v
Pod B
10.244.2.15
```

The exact implementation is provided by the cluster's networking solution/CNI.

---

# 4. Pod IP Address

When a Pod is created, the cluster networking system assigns it an IP.

Example:

```text
frontend-pod -> 10.244.1.10
backend-pod  -> 10.244.2.15
```

Check Pod IPs:

```bash
kubectl get pods -o wide
```

Example:

```text
NAME       READY   STATUS    IP            NODE
frontend   1/1     Running   10.244.1.10   worker-01
backend    1/1     Running   10.244.2.15   worker-02
```

Pod IPs are generally ephemeral.

Do not design applications around permanently assigning a particular Pod IP.

---

# 5. Why Direct Pod IP Communication Is Not Enough

Suppose:

```text
Frontend
   |
   v
10.244.1.10
Backend
```

The backend Pod crashes.

Kubernetes creates a replacement:

```text
Old Backend:
10.244.1.10

New Backend:
10.244.2.20
```

The frontend should not need to know the new address.

Instead:

```text
Frontend
    |
    v
Backend Service
    |
    +---- Backend Pod
    |
    +---- Backend Pod
    |
    +---- Backend Pod
```

The Service gives the application a stable endpoint.

---

# 6. What Is a Kubernetes Service?

A **Service** is a Kubernetes abstraction that provides a stable network endpoint for a group of Pods.

Example:

```text
                 backend-service
                       |
              +--------+--------+
              |        |        |
              v        v        v
           Pod A     Pod B     Pod C
```

Pods can change while the Service remains stable.

A Service generally uses a selector to identify backend Pods.

---

# 7. Labels and Service Selectors

Suppose backend Pods have:

```yaml
labels:
  app: backend
```

The Service can select them:

```yaml
selector:
  app: backend
```

Conceptually:

```text
Service
 selector:
   app=backend
       |
       v
+------+------+------+
|      |      |      |
Pod A  Pod B  Pod C
```

This is why labels and selectors are fundamental to Kubernetes networking.

---

# 8. ClusterIP

**ClusterIP** is the default Kubernetes Service type.

It provides an internal virtual IP reachable from within the cluster.

Example:

```text
Frontend Pod
     |
     v
Backend Service
ClusterIP: 10.x.x.x
     |
     +---- Backend Pod
     +---- Backend Pod
     +---- Backend Pod
```

Example YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Check:

```bash
kubectl get svc
```

---

# 9. port vs targetPort

This is one of the most common beginner questions.

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Traffic conceptually flows:

```text
Client
  |
  | Service port 80
  v
Service
  |
  | targetPort 8080
  v
Pod
  |
  v
Application listening on 8080
```

Remember:

```text
port       = Service port
targetPort = destination port on the selected Pod
```

---

# 10. NodePort

A **NodePort** Service exposes a Service through a port on each eligible Node.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Conceptually:

```text
Client
   |
   v
NodeIP:30080
   |
   v
Service
   |
   v
Web Pods
```

Check:

```bash
kubectl get svc web
```

NodePort is useful for learning and some environments. Cloud production architectures often use cloud LoadBalancers or Ingress-based solutions instead.

---

# 11. LoadBalancer

A Service of type `LoadBalancer` requests an external/cloud load-balancing mechanism when supported by the environment.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

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

In AWS EKS, AWS integrations can provision/configure load-balancing infrastructure depending on the Service and controller configuration.

---

# 12. Service Types Summary

| Type | Purpose |
|---|---|
| ClusterIP | Internal cluster access |
| NodePort | Expose through a Node port |
| LoadBalancer | Request an external/cloud load balancer |
| ExternalName | DNS-style alias to an external hostname |

Default Service type:

```text
ClusterIP
```

---

# 13. Kubernetes DNS

Hardcoding Service IP addresses is inconvenient.

Kubernetes provides DNS-based service discovery through its cluster DNS system.

Instead of:

```text
10.96.45.12
```

applications can usually communicate using:

```text
backend
```

Inside the same namespace, a client can typically use:

```text
http://backend
```

A fully qualified Service DNS name commonly looks like:

```text
backend.default.svc.cluster.local
```

Where:

```text
backend
   = Service name

default
   = Namespace

svc
   = Service DNS category

cluster.local
   = Cluster DNS domain
```

The cluster DNS domain can be customized.

---

# 14. Service Discovery Example

Suppose:

```text
Namespace: production

Service:
backend
```

Frontend can call:

```text
http://backend
```

Conceptually:

```text
Frontend Pod
     |
     | DNS query
     v
Cluster DNS
     |
     v
backend
     |
     v
Service
     |
     v
Backend Pods
```

This avoids hardcoding changing Pod addresses.

---

# 15. CoreDNS

Kubernetes clusters commonly use **CoreDNS** for cluster DNS.

CoreDNS helps resolve names such as:

```text
backend.default.svc.cluster.local
```

Check DNS Pods:

```bash
kubectl get pods -n kube-system
```

You may see Pods with names containing:

```text
coredns
```

You can also try:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

The exact labels can vary by Kubernetes distribution.

---

# 16. CNI — Container Network Interface

**CNI** stands for **Container Network Interface**.

CNI provides a standard mechanism for configuring networking for containers/Pods.

Conceptually:

```text
Kubernetes
    |
    v
Container Runtime
    |
    v
CNI Plugin
    |
    +--> Pod network namespace
    +--> Pod IP
    +--> Routes
    +--> Connectivity
```

The CNI implementation handles much of the low-level Pod networking setup.

---

# 17. Common CNI Implementations

Examples include:

- Calico
- Cilium
- Flannel
- Amazon VPC CNI

Different CNIs provide different capabilities.

Depending on the implementation, networking may include:

- Pod networking
- Routing
- Network policy enforcement
- Load-balancing-related features
- Observability features
- Advanced traffic control

Do not assume every Kubernetes cluster uses the same CNI.

---

# 18. AWS VPC CNI

Amazon EKS commonly uses the **Amazon VPC CNI plugin**.

A simplified model:

```text
AWS VPC
 |
 +-- Subnet
      |
      +-- Node
      |    |
      |    +-- Pod IP
      |    +-- Pod IP
      |
      +-- Node
           |
           +-- Pod IP
           +-- Pod IP
```

Because Pod networking integrates with AWS VPC networking, EKS networking troubleshooting may require understanding:

- VPC CIDRs
- Subnets
- Route tables
- ENIs
- Available IP addresses
- Security Groups

---

# 19. kube-proxy and Services

Historically, kube-proxy has been responsible for programming node networking rules that help implement Kubernetes Service traffic handling.

Conceptually:

```text
Client Pod
    |
    v
Service Virtual IP
    |
    v
Node networking
    |
    +---- Pod A
    +---- Pod B
    +---- Pod C
```

Depending on the Kubernetes networking configuration, Service traffic can be implemented using different mechanisms.

Do not assume kube-proxy is the only possible way every modern Kubernetes environment handles Service networking.

---

# 20. EndpointSlices

A Service needs to know which backend endpoints currently exist.

Kubernetes uses **EndpointSlices** to represent groups of network endpoints associated with Services.

Conceptually:

```text
Service
   |
   v
EndpointSlice
   |
   +-- 10.244.1.10:8080
   +-- 10.244.1.11:8080
   +-- 10.244.2.15:8080
```

Inspect:

```bash
kubectl get endpointslices
```

For a particular Service:

```bash
kubectl get endpointslices   -l kubernetes.io/service-name=backend
```

This is useful when troubleshooting:

> "The Service exists, but why isn't traffic reaching my Pods?"

---

# 21. NetworkPolicy

A **NetworkPolicy** expresses rules controlling allowed network traffic to and/or from selected Pods, when the cluster networking implementation supports and enforces NetworkPolicy.

Imagine:

```text
Frontend
   |
   | allowed
   v
Backend
   |
   | allowed
   v
Database
```

But:

```text
Untrusted workload
       X
       |
       v
Database
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

This expresses the intent:

> Allow ingress to backend Pods from Pods labeled `app=frontend`.

Important:

> NetworkPolicy is an API object; enforcement depends on the cluster's networking implementation.

---

# 22. Same-Pod Networking

Containers in the same Pod share the Pod network namespace.

Therefore:

```text
Pod
 |
 +-- Container A
 |
 +-- Container B
```

can communicate through:

```text
localhost
```

For example:

```text
Container A
     |
     | localhost:8080
     v
Container B
```

This is different from two separate Pods.

---

# 23. Pod-to-Pod Communication

Suppose:

```text
Pod A
10.244.1.10

Pod B
10.244.2.20
```

Pod A can communicate with Pod B through the cluster network.

The CNI implementation provides the underlying connectivity.

You can create a temporary debugging Pod:

```bash
kubectl run network-test --rm -it   --image=busybox:1.36 -- /bin/sh
```

Inside:

```sh
ping <pod-ip>
```

Depending on the image and cluster, `ping` may not be available.

---

# 24. DNS Testing

Create a temporary debugging Pod:

```bash
kubectl run dns-test --rm -it   --image=busybox:1.36 -- /bin/sh
```

Inside:

```sh
nslookup kubernetes.default
```

Test a Service:

```sh
nslookup backend
```

Test the full name:

```sh
nslookup backend.default.svc.cluster.local
```

If your chosen debugging image does not contain `nslookup`, use another diagnostic image/tool.

---

# 25. Practical Lab 1 — Create Backend Pods

Create `backend.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:alpine
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f backend.yaml
```

Check:

```bash
kubectl get pods -o wide
```

---

# 26. Practical Lab 2 — Create a ClusterIP Service

Create `backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc backend
```

---

# 27. Practical Lab 3 — Inspect Service Endpoints

Run:

```bash
kubectl get endpoints backend
```

Also:

```bash
kubectl get endpointslices
```

The EndpointSlices should contain endpoints corresponding to the selected backend Pods.

This teaches an important debugging technique:

> A Service with no endpoints often means its selector does not match the intended Pods or there are no eligible Ready endpoints.

---

# 28. Practical Lab 4 — Test Service DNS

Create:

```bash
kubectl run dns-test --rm -it   --image=busybox:1.36 -- /bin/sh
```

Inside:

```sh
nslookup backend
```

Then:

```sh
wget -qO- http://backend
```

if supported by the image.

Exit:

```sh
exit
```

---

# 29. Practical Lab 5 — Test Service Connectivity

Run:

```bash
kubectl run curl-test --rm -it   --image=curlimages/curl -- sh
```

Inside:

```sh
curl http://backend
```

You should receive the nginx response if the Service and Pods are functioning.

Exit:

```sh
exit
```

---

# 30. Practical Lab 6 — Break the Service Selector

Current backend Pods have:

```yaml
labels:
  app: backend
```

Change the Service selector to:

```yaml
selector:
  app: wrong
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get endpoints backend
kubectl get endpointslices
```

There should be no intended backend endpoints.

Test from a debugging Pod:

```bash
curl http://backend
```

The request should fail or have no backend response.

Fix:

```yaml
selector:
  app: backend
```

Apply again:

```bash
kubectl apply -f backend-service.yaml
```

Test again.

---

# 31. Practical Lab 7 — NodePort

Create:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: backend
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
kubectl get svc backend-nodeport
```

Inspect Node IPs:

```bash
kubectl get nodes -o wide
```

In a local environment, access:

```text
NodeIP:30080
```

if your cluster networking permits it.

---

# 32. Practical Lab 8 — Inspect CoreDNS

Run:

```bash
kubectl get pods -n kube-system
```

Find CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Inspect logs if required:

```bash
kubectl logs -n kube-system <coredns-pod>
```

Do not assume the exact Pod name or label is identical across every Kubernetes distribution.

---

# 33. Practical Lab 9 — Identify the CNI

Start with:

```bash
kubectl get pods -n kube-system
```

Look for networking components.

Depending on your cluster, you may see components associated with:

```text
calico
cilium
flannel
aws-node
```

In EKS using Amazon VPC CNI, `aws-node` Pods are commonly present in `kube-system`.

---

# 34. Network Troubleshooting Workflow

Suppose:

```text
Frontend cannot reach Backend
```

Use a structured approach.

### Step 1 — Is the backend running?

```bash
kubectl get pods -l app=backend
```

### Step 2 — Is it Ready?

```bash
kubectl get pods -l app=backend
```

### Step 3 — Does the Service exist?

```bash
kubectl get svc backend
```

### Step 4 — Does the Service have endpoints?

```bash
kubectl get endpoints backend
kubectl get endpointslices
```

### Step 5 — Does the selector match?

```bash
kubectl describe svc backend
kubectl get pods --show-labels
```

### Step 6 — Does DNS resolve?

From a debugging Pod:

```bash
nslookup backend
```

### Step 7 — Can the client connect?

```bash
curl http://backend
```

### Step 8 — Check NetworkPolicies

```bash
kubectl get networkpolicy
```

### Step 9 — Check application port

Make sure the application is actually listening on the expected port.

### Step 10 — Investigate CNI/node/cloud networking

If the above looks correct, investigate the CNI, Node networking and, in EKS, AWS VPC networking.

---

# 35. Common Networking Problems

## Problem 1 — Service has no endpoints

Possible cause:

```text
Service selector != Pod labels
```

Check:

```bash
kubectl describe svc <service>
kubectl get pods --show-labels
```

---

## Problem 2 — DNS does not resolve

Check:

```bash
kubectl get pods -n kube-system
```

Look for CoreDNS.

Then:

```bash
kubectl logs -n kube-system <coredns-pod>
```

---

## Problem 3 — Connection refused

Possible causes:

- Application is not running.
- Application listens on another port.
- `targetPort` is incorrect.
- Application is not Ready.
- Application is bound incorrectly.

---

## Problem 4 — Connection timeout

Possible causes:

- NetworkPolicy.
- CNI/routing problem.
- Node networking problem.
- Cloud firewall/security group.
- Network ACL.
- Application not responding.

---

## Problem 5 — Service exists but traffic doesn't reach Pods

Check:

```bash
kubectl get endpoints <service>
```

If there are no endpoints, investigate:

- Service selector.
- Pod labels.
- Pod readiness.
- EndpointSlice state.

---

# 36. AWS EKS Networking Architecture

A simplified architecture:

```text
                         Internet
                            |
                            v
                    AWS Load Balancer
                            |
                            v
                          EKS
                            |
                 +----------+----------+
                 |                     |
              Node 1                Node 2
                 |                     |
            +----+----+           +----+----+
            |         |           |         |
          Pod A     Pod B       Pod C     Pod D
```

Important AWS networking concepts:

- VPC
- Subnets
- Route tables
- Security Groups
- Network ACLs
- Elastic Network Interfaces
- Pod IP allocation
- Private/public networking

---

# 37. EKS Networking Troubleshooting Example

Suppose:

```text
Backend Pod -> Database
```

fails.

Potential layers:

```text
Application
    |
    v
Pod
    |
    v
Service / DNS
    |
    v
CNI
    |
    v
Node
    |
    v
AWS VPC
    |
    v
Security Group / Network ACL / Routes
    |
    v
Database
```

Use this layered model instead of randomly changing AWS security settings.

---

# 38. Mini Project — Three-Tier Kubernetes Networking

## Objective

Build and test:

```text
Frontend
   |
   v
Backend Service
   |
   v
Backend Pods
```

### Step 1 — Backend Deployment

Create a Deployment with:

```text
3 replicas
label: app=backend
container port: 80
```

### Step 2 — Backend Service

Create a ClusterIP Service:

```text
Service name: backend
selector: app=backend
port: 80
targetPort: 80
```

### Step 3 — Test Pod

Run:

```bash
kubectl run curl-test --rm -it   --image=curlimages/curl -- sh
```

Inside:

```sh
curl http://backend
```

### Step 4 — DNS

```sh
nslookup backend
```

### Step 5 — Break the Service

Change:

```yaml
selector:
  app: wrong
```

Observe:

```bash
kubectl get endpoints backend
kubectl get endpointslices
```

### Step 6 — Fix the Service

Restore:

```yaml
selector:
  app: backend
```

### Step 7 — Add NetworkPolicy

Create a policy allowing backend traffic only from Pods labeled:

```text
app=frontend
```

### Step 8 — Document the traffic path

Students should draw:

```text
Frontend Pod
     |
     v
Cluster DNS
     |
     v
Backend Service
     |
     v
EndpointSlice
     |
     v
Backend Pod
```

---

# 39. Interview Questions and Answers

## Q1. What is Kubernetes networking?

**Answer:** Kubernetes networking is the set of mechanisms that allow Pods, Nodes, Services and external clients to communicate according to the Kubernetes networking model.

## Q2. Does every Pod get an IP?

**Answer:** Under the standard Kubernetes networking model, each Pod receives its own IP address.

## Q3. Why shouldn't applications depend on Pod IPs?

**Answer:** Pod IPs are generally ephemeral. Pods can be recreated and receive different IP addresses.

## Q4. What is a Service?

**Answer:** A Service provides a stable network endpoint for a group of Pods selected by labels.

## Q5. What is ClusterIP?

**Answer:** ClusterIP is the default Service type and provides an internal virtual IP for access within the cluster.

## Q6. What is the difference between `port` and `targetPort`?

**Answer:**

- `port` = Service port.
- `targetPort` = destination port on the selected Pod.

## Q7. What is NodePort?

**Answer:** NodePort exposes a Service through a port on each eligible Node.

## Q8. What is LoadBalancer?

**Answer:** LoadBalancer requests an external/cloud load-balancing mechanism when supported by the environment.

## Q9. What is CNI?

**Answer:** CNI stands for Container Network Interface. It provides a standard mechanism for configuring container/Pod networking.

## Q10. What is CoreDNS?

**Answer:** CoreDNS is commonly used as the Kubernetes cluster DNS service and provides DNS-based service discovery.

## Q11. What does kube-proxy do?

**Answer:** kube-proxy has historically been used to configure node networking rules that implement Kubernetes Service traffic handling. The exact implementation depends on the cluster configuration.

## Q12. What is an EndpointSlice?

**Answer:** EndpointSlices represent groups of network endpoints associated with Services and provide a scalable way to track backend endpoints.

## Q13. What is NetworkPolicy?

**Answer:** NetworkPolicy expresses rules controlling allowed network traffic to and/or from selected Pods when the cluster networking implementation supports and enforces it.

## Q14. How do you troubleshoot a Service that isn't working?

Start with:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
kubectl get pods --show-labels
kubectl describe svc <service>
```

Then test DNS and connectivity from a debugging Pod.

## Q15. How do Pods on different Nodes communicate?

**Answer:** The cluster's networking/CNI implementation provides the underlying connectivity between Pod networks across Nodes.

---

# 40. Common Beginner Mistakes

### Mistake 1 — Using Pod IPs as permanent addresses

Pod IPs can change.

Use Services for stable application access.

### Mistake 2 — Confusing `port` and `targetPort`

Remember:

```text
port       -> Service
targetPort -> Pod
```

### Mistake 3 — Forgetting Service selectors

A Service needs a selector that matches the intended backend Pods.

### Mistake 4 — Assuming a ClusterIP Service is Internet-accessible

ClusterIP is internal.

External access generally requires mechanisms such as NodePort, LoadBalancer or Ingress.

### Mistake 5 — Assuming every Kubernetes cluster uses the same CNI

Different distributions and environments can use different networking implementations.

### Mistake 6 — Debugging only the application

Networking failures can exist at multiple layers:

```text
Application
Pod
Service
DNS
CNI
Node
Cloud Network
```

### Mistake 7 — Assuming kube-proxy is the entire Kubernetes networking system

Kubernetes networking involves multiple components and layers, including CNI, Services, DNS and potentially kube-proxy or alternative implementations.

---

# 41. Quick Revision

```text
                    External User
                          |
                          v
                 LoadBalancer / Ingress
                          |
                          v
                       Service
                          |
                 +--------+--------+
                 |        |        |
                 v        v        v
               Pod A    Pod B    Pod C
```

Internal communication:

```text
Frontend Pod
     |
     | http://backend
     v
Cluster DNS
     |
     v
Backend Service
     |
     v
EndpointSlice
     |
     v
Backend Pods
```

Networking components:

```text
Kubernetes
 |
 +-- CNI
 |    |
 |    +-- Pod networking
 |
 +-- Service
 |    |
 |    +-- Stable endpoint
 |
 +-- CoreDNS
 |    |
 |    +-- Service discovery
 |
 +-- kube-proxy / equivalent
 |    |
 |    +-- Service traffic handling
 |
 +-- NetworkPolicy
      |
      +-- Traffic control
```

---

# 42. Homework

1. Explain the Kubernetes networking model.
2. Explain why Pods need IP addresses.
3. Explain why Pod IPs should not be treated as permanent.
4. Explain Kubernetes Services.
5. Explain ClusterIP.
6. Explain NodePort.
7. Explain LoadBalancer.
8. Explain `port` vs `targetPort`.
9. Explain CoreDNS.
10. Explain CNI.
11. Find which CNI your Kubernetes cluster uses.
12. Create a Deployment and ClusterIP Service.
13. Test the Service from another Pod.
14. Break the Service selector and troubleshoot it.
15. Inspect EndpointSlices.
16. Create a NetworkPolicy that limits backend access.
17. Draw the networking path from frontend Pod to backend Pod.
18. Explain the difference between Kubernetes networking and AWS VPC networking.

---

# Module 8 Takeaway

The most important concepts are:

```text
Pod
 |
 +-- Gets a Pod IP
 |
 +-- Communicates over cluster networking

Service
 |
 +-- Stable endpoint
 |
 +-- Selects Pods using labels

CoreDNS
 |
 +-- Service discovery

CNI
 |
 +-- Provides Pod networking

kube-proxy / equivalent
 |
 +-- Helps implement Service traffic handling

NetworkPolicy
 |
 +-- Controls allowed traffic
```

Use this troubleshooting sequence:

```text
Pod healthy?
     |
     v
Service exists?
     |
     v
Selector correct?
     |
     v
Endpoints exist?
     |
     v
DNS resolves?
     |
     v
Network connection works?
     |
     v
NetworkPolicy allows traffic?
     |
     v
CNI / Node / Cloud network healthy?
```

Once this layered model is understood, Kubernetes networking becomes much easier to troubleshoot.

---

# Next Module

## Module 9 — Pods, Deployments and Services

Topics:

- Pods in depth.
- Pod specification.
- Container configuration.
- Pod labels and selectors.
- Deployments.
- ReplicaSets.
- Services.
- Deployment-to-Service architecture.
- Rolling updates.
- Rollbacks.
- Scaling.
- Health checks.
- Practical application deployment.
- AWS/EKS deployment.
- Mini project.
- Interview questions.
- Common mistakes.
