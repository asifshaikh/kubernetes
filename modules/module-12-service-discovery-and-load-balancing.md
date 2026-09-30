# Module 12 — Service Discovery and Load Balancing in Kubernetes

## Learning Objectives

By the end of this module, students should be able to:

- Explain why Kubernetes Services are required.
- Understand Kubernetes service discovery.
- Explain ClusterIP, NodePort, LoadBalancer, and ExternalName Services.
- Understand Kubernetes DNS and CoreDNS.
- Explain EndpointSlices.
- Understand how kube-proxy participates in Service networking.
- Understand internal and external traffic.
- Understand session affinity and headless Services.
- Troubleshoot Service and DNS problems.
- Understand AWS Load Balancer integration with EKS.
- Build a practical frontend → backend application using Kubernetes Services.

---

# 1. The Problem: Pods Keep Changing

Suppose we have three backend Pods:

```text
backend-pod-1 → 10.244.1.10
backend-pod-2 → 10.244.2.15
backend-pod-3 → 10.244.3.21
```

A frontend should not normally depend directly on these IP addresses.

What happens if:

- a Pod crashes?
- Kubernetes replaces it?
- the Pod moves to another node?
- the application scales from 3 Pods to 10?

Pod IP addresses can change.

Kubernetes Services solve this problem by providing a stable network endpoint for a group of Pods.

```text
Frontend
   |
   v
Service
   |
   +----> Pod 1
   +----> Pod 2
   +----> Pod 3
```

---

# 2. What Is a Kubernetes Service?

A Service provides a stable network endpoint for a set of Pods.

For example:

```text
backend-service
```

can represent:

```text
backend-pod-1
backend-pod-2
backend-pod-3
```

The frontend can call:

```bash
curl http://backend-service
```

instead of using a Pod IP.

The Pods can change while the Service remains the stable access point.

---

# 3. Real-World Analogy

Imagine a company with 10 employees.

Customers should not call a particular employee's personal phone number because that employee may leave or become unavailable.

Instead, customers call:

```text
Customer Support: 1800-123-456
```

The company routes the call to an available employee.

In Kubernetes:

```text
Service = Customer Support number
Pods    = Employees
```

---

# 4. Service Discovery

Service discovery means finding the network location of a service without manually knowing the IP addresses of individual application instances.

Kubernetes commonly provides this through:

```text
Service + CoreDNS
```

For example:

```bash
curl http://backend
```

The application does not need to know the backend Pod IP.

---

# 5. Kubernetes DNS

Kubernetes normally provides DNS through CoreDNS.

A Service DNS name commonly follows:

```text
<service>.<namespace>.svc.cluster.local
```

Example:

```text
backend.default.svc.cluster.local
```

Inside the same namespace, this can usually be shortened to:

```text
backend
```

Across namespaces:

```text
backend.production
```

or:

```text
backend.production.svc.cluster.local
```

---

# 6. CoreDNS

CoreDNS provides DNS-based service discovery inside the cluster.

Simplified flow:

```text
Application Pod
      |
      | DNS query: backend
      v
   CoreDNS
      |
      v
Service DNS information
      |
      v
Backend Service
```

Check CoreDNS:

```bash
kubectl get pods -n kube-system
```

You may see Pods such as:

```text
coredns-xxxxx
coredns-yyyyy
```

---

# 7. Service Selectors

A Service normally uses a selector to identify its backend Pods.

Pod:

```yaml
metadata:
  labels:
    app: backend
```

Service:

```yaml
selector:
  app: backend
```

Therefore:

```text
Service
   |
   | selector: app=backend
   |
   +----> Pod A: app=backend
   +----> Pod B: app=backend
   +----> Pod C: app=backend
```

If the selector doesn't match the Pod labels, the Service will not have the expected endpoints.

---

# 8. ClusterIP

ClusterIP is the default Service type.

Example:

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
      targetPort: 5000
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc
```

Example:

```text
NAME      TYPE        CLUSTER-IP      PORT(S)
backend   ClusterIP   10.96.120.10    80/TCP
```

ClusterIP is normally used for internal cluster communication.

---

# 9. `port` vs `targetPort`

Consider:

```yaml
ports:
  - port: 80
    targetPort: 5000
```

`port` is the port exposed by the Service.

`targetPort` is the destination port on the selected Pod.

Traffic:

```text
Client
  |
  | :80
  v
Service
  |
  | forwards to :5000
  v
Backend Pod
```

They do not have to be the same.

---

# 10. EndpointSlices

A Service needs to track the actual backend network endpoints.

Modern Kubernetes uses EndpointSlices for this.

Check them:

```bash
kubectl get endpointslices
```

For a specific Service:

```bash
kubectl get endpointslices   -l kubernetes.io/service-name=backend
```

Conceptually:

```text
Service
   |
   v
EndpointSlice
   |
   +----> 10.244.1.10:5000
   +----> 10.244.2.15:5000
   +----> 10.244.3.21:5000
```

When a matching Pod disappears, its endpoint can be removed. When a new matching Pod becomes ready, its endpoint can be added.

---

# 11. Service Load Balancing

Suppose a Service has three backend Pods:

```text
Backend Pod 1
Backend Pod 2
Backend Pod 3
```

A client connects to the Service:

```text
Client
   |
   v
Service
   |
   +----> Pod 1
   +----> Pod 2
   +----> Pod 3
```

The Service provides a stable destination while Kubernetes networking mechanisms direct traffic toward available endpoints.

The exact packet path depends on the cluster networking implementation, kube-proxy mode, CNI, and cloud integration.

Remember:

> Service = stable access point; Kubernetes networking = mechanism that reaches the selected endpoints.

---

# 12. kube-proxy

`kube-proxy` is a node-level Kubernetes component involved in implementing Service networking.

Depending on the environment, it can use mechanisms such as:

- iptables
- IPVS
- nftables

Conceptually:

```text
Client Pod
    |
    v
Service IP
    |
    v
Node networking rules
    |
    v
Backend endpoint
```

Important:

> kube-proxy does not run your application containers.

The kubelet and container runtime are responsible for running containers on worker nodes.

---

# 13. NodePort

NodePort exposes a Service through a port on each node.

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
      targetPort: 8080
      nodePort: 30080
```

Traffic:

```text
Client
  |
  v
Node IP:30080
  |
  v
Service
  |
  v
Web Pod:8080
```

The commonly used NodePort range is:

```text
30000-32767
```

unless the cluster is configured differently.

Check:

```bash
kubectl get svc
```

---

# 14. When Is NodePort Useful?

NodePort is useful for:

- learning
- development
- simple environments
- exposing a Service through node addresses
- situations where another load balancer sits in front of the nodes

For many production cloud architectures, applications are exposed through cloud load-balancing or Ingress rather than relying directly on arbitrary node ports.

---

# 15. LoadBalancer

A Service with:

```yaml
type: LoadBalancer
```

requests external load-balancing functionality from the environment.

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
      targetPort: 8080
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
   +----> Pod
   +----> Pod
   +----> Pod
```

The exact infrastructure depends on the cloud provider and controller.

---

# 16. Service Types Comparison

| Type | Typical Reachability | Common Use |
|---|---|---|
| ClusterIP | Inside cluster | Internal services |
| NodePort | Node IP + port | Development/simple external access |
| LoadBalancer | External load balancer | Cloud-facing applications |
| ExternalName | DNS alias | Referring to an external DNS name |

Mental model:

```text
ClusterIP
    ↓
Internal

NodePort
    ↓
Node address + port

LoadBalancer
    ↓
External/cloud load balancer
```

---

# 17. Internal vs External Traffic

Consider an ecommerce application:

```text
Internet
   |
   v
Frontend
   |
   v
Backend Service
   |
   v
Backend Pods
   |
   v
Database
```

A possible Kubernetes design is:

```text
Frontend → externally exposed Service
Backend  → ClusterIP
Database → internal Service
```

Only components that actually require external access should normally be exposed externally.

---

# 18. Service Discovery Example

Create a backend Deployment:

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
          image: nginx
          ports:
            - containerPort: 80
```

Create its Service:

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
```

Apply:

```bash
kubectl apply -f backend.yaml
kubectl apply -f backend-service.yaml
```

Inspect:

```bash
kubectl get pods
kubectl get svc
kubectl get endpointslices
```

---

# 19. Test DNS From Another Pod

Run a temporary debugging Pod:

```bash
kubectl run dns-test   --image=busybox:1.36   -it --rm   --restart=Never -- sh
```

Inside:

```bash
nslookup backend
```

Then:

```bash
wget -qO- http://backend
```

The flow is:

```text
dns-test Pod
    |
    | DNS lookup: backend
    v
CoreDNS
    |
    v
Backend Service
    |
    v
Backend endpoint
```

---

# 20. Practical Lab — Break the Service Selector

Check labels:

```bash
kubectl get pods --show-labels
```

Suppose the Pods have:

```text
app=backend
```

Edit the Service:

```bash
kubectl edit svc backend
```

Change:

```yaml
selector:
  app: backend
```

to:

```yaml
selector:
  app: wrong
```

Check:

```bash
kubectl get endpointslices
```

The Service should no longer have the expected backend endpoints.

Test:

```bash
kubectl run client   --image=busybox:1.36   -it --rm   --restart=Never -- sh
```

Inside:

```bash
wget -qO- http://backend
```

Fix the selector:

```yaml
selector:
  app: backend
```

Then check:

```bash
kubectl get endpointslices
```

This is one of the most important Service troubleshooting exercises.

---

# 21. Headless Services

A headless Service does not receive a normal ClusterIP.

Use:

```yaml
clusterIP: None
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

Headless Services are useful when applications need to discover individual endpoints directly.

Common use cases include:

- StatefulSets
- databases
- distributed systems
- systems requiring endpoint discovery

Conceptually:

```text
database.default.svc.cluster.local
          |
          +----> Pod 1 IP
          +----> Pod 2 IP
          +----> Pod 3 IP
```

---

# 22. Session Affinity

Sometimes an application wants requests from the same client to preferentially reach the same backend endpoint.

Kubernetes Services support:

```yaml
sessionAffinity: ClientIP
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  sessionAffinity: ClientIP
  ports:
    - port: 80
      targetPort: 8080
```

This is sometimes called sticky behavior.

Applications should ideally avoid depending on one particular Pod. Shared session state can instead be stored in systems such as Redis or a database.

---

# 23. Service Without a Selector

A Service can exist without a selector.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  ports:
    - port: 443
```

Endpoints can then be managed separately.

This can be useful when the target is outside the cluster or when custom endpoint management is required.

---

# 24. Kubernetes Microservice Architecture

A typical application can look like:

```text
Frontend Pod
     |
     | http://auth
     v
Auth Service
     |
     +------> Auth Pod
     +------> Auth Pod

Frontend Pod
     |
     | http://cart
     v
Cart Service
     |
     +------> Cart Pod
     +------> Cart Pod

Cart Pod
     |
     | http://payment
     v
Payment Service
     |
     +------> Payment Pod
```

Each Service provides a stable name for its microservice.

---

# 25. AWS EKS and Load Balancing

In Amazon EKS, Kubernetes Services can integrate with AWS networking and load-balancing components.

A simplified architecture is:

```text
Internet
   |
   v
AWS Load Balancer
   |
   v
Kubernetes Service
   |
   v
Pods
```

AWS environments can use load-balancing technologies such as:

- Application Load Balancer (ALB)
- Network Load Balancer (NLB)

HTTP/HTTPS routing commonly uses an AWS load-balancer controller with Kubernetes Ingress resources, while `Service` type `LoadBalancer` is another way to request external load-balancing functionality.

The exact behavior depends on the AWS controller, annotations, Service configuration, networking mode, and cluster setup.

---

# 26. AWS EKS Internal Microservice Example

Imagine:

```text
Internet
   |
   v
Frontend
   |
   v
Auth Service
   |
   v
User Service
   |
   v
Database
```

Possible Kubernetes design:

```text
Frontend Service
    ↓
Frontend Pods

Auth Service
    ↓
Auth Pods

User Service
    ↓
User Pods

Database Service
    ↓
Database Pods / external database
```

Internal services can communicate through DNS:

```text
auth
user
orders
payments
```

For example:

```text
http://auth
http://user
```

---

# 27. Service Troubleshooting Workflow

When a Service doesn't work, follow a fixed sequence.

## Step 1 — Check Pods

```bash
kubectl get pods
```

Check:

```text
STATUS
READY
RESTARTS
```

## Step 2 — Check Labels

```bash
kubectl get pods --show-labels
```

## Step 3 — Check Service

```bash
kubectl describe svc backend
```

Compare the Service selector with Pod labels.

## Step 4 — Check EndpointSlices

```bash
kubectl get endpointslices
```

If there are no matching endpoints, investigate:

- selector
- labels
- readiness
- namespace
- Service configuration

## Step 5 — Check DNS

```bash
nslookup backend
```

If DNS fails:

```bash
kubectl get pods -n kube-system
```

Inspect CoreDNS.

## Step 6 — Test the Service

```bash
wget -qO- http://backend
```

or:

```bash
curl http://backend
```

## Step 7 — Test the Application

Check:

```bash
kubectl logs <pod>
kubectl describe pod <pod>
```

Also verify that the application is listening on the expected port.

---

# 28. Common Service Problems

## Problem 1 — Wrong Selector

Service:

```yaml
selector:
  app: api
```

Pod:

```yaml
labels:
  app: backend
```

Result:

```text
No matching endpoints
```

Fix:

```yaml
selector:
  app: backend
```

---

## Problem 2 — Wrong targetPort

Application listens on:

```text
5000
```

Service:

```yaml
targetPort: 8080
```

Result:

```text
Connection failure
```

Fix:

```yaml
targetPort: 5000
```

---

## Problem 3 — Wrong Namespace

Service exists in:

```text
production
```

A client in another namespace tries:

```bash
curl http://backend
```

Use:

```bash
curl http://backend.production
```

---

## Problem 4 — Application Not Listening

A Pod can be Running while its application is not listening on the expected port.

Check:

```bash
kubectl logs <pod>
kubectl exec -it <pod> -- sh
```

---

# 29. Service vs Ingress

A Service provides stable access to a group of Pods.

Ingress provides HTTP/HTTPS routing into Services through an Ingress controller.

Example:

```text
Internet
    |
    v
Ingress
    |
    +---- /api → backend-service
    |
    +---- /    → frontend-service
```

Ingress will be covered in depth in a later module.

---

# 30. Service vs EndpointSlice

### Service

Provides:

- stable virtual endpoint
- DNS name
- port definition
- selector
- networking abstraction

### EndpointSlice

Represents:

- backend network endpoints
- endpoint addresses
- ports
- endpoint conditions

Simplified:

```text
Service
   |
   v
EndpointSlices
   |
   +----> Pod endpoint
   +----> Pod endpoint
   +----> Pod endpoint
```

---

# 31. Service vs Deployment

A Deployment manages application Pods.

A Service provides stable networking to those Pods.

```text
Deployment
    |
    +----> Pod
    +----> Pod
    +----> Pod

Service
    |
    +----> Pod
    +----> Pod
    +----> Pod
```

Mental model:

```text
Deployment = How many application instances should exist?

Service = How do clients reliably reach those instances?
```

---

# 32. Practical Lab — NodePort

Use:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
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
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc backend
```

For local clusters such as kind, host access may require port mappings. Minikube can simplify access with:

```bash
minikube service backend
```

---

# 33. Practical Lab — LoadBalancer

Change the Service to:

```yaml
type: LoadBalancer
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc backend
```

On a cloud cluster, an external load balancer may be provisioned depending on the cloud integration/controller.

On a local cluster, an external address may not automatically appear.

---

# 34. AWS Mini Project — E-Commerce Service Discovery

Build:

```text
                Internet
                   |
                   v
            Frontend Service
                   |
        +----------+----------+
        |          |          |
        v          v          v
      Auth       Product     Cart
     Service     Service    Service
        |          |          |
        v          v          v
      Pods        Pods       Pods
```

## Technologies

Use:

- Docker
- Kubernetes
- Amazon ECR
- Amazon EKS
- Kubernetes Services
- CoreDNS
- Deployments
- ConfigMaps
- Secrets

## Services

Create:

```text
auth-service
product-service
cart-service
frontend-service
```

Internal communication:

```text
frontend → auth-service
frontend → product-service
frontend → cart-service
```

Each service uses Kubernetes DNS.

---

# 35. AWS Mini Project Architecture

A possible architecture:

```text
Internet
   |
   v
AWS Load Balancer
   |
   v
Frontend Service
   |
   v
Frontend Pods
   |
   v
Backend ClusterIP Service
   |
   v
Backend Pods
   |
   v
Redis ClusterIP Service
   |
   v
Redis Pods
```

Only the frontend needs external exposure.

Backend and Redis remain internal.

---

# 36. Mini Project — Service Discovery Dashboard

Build:

```text
Frontend
Backend
Redis
```

Architecture:

```text
                 User
                  |
                  v
           Frontend Service
                  |
                  v
            Frontend Pods
                  |
                  v
           Backend Service
                  |
          +-------+-------+
          |               |
          v               v
     Backend Pods       Redis Service
                            |
                            v
                       Redis Pod
```

## Requirements

Create Docker images for:

```text
frontend
backend
```

Push them to Docker Hub or Amazon ECR.

Create:

```text
frontend-deployment.yaml
frontend-service.yaml
backend-deployment.yaml
backend-service.yaml
redis-deployment.yaml
redis-service.yaml
```

Service discovery requirements:

```text
frontend → backend-service
backend → redis-service
```

Do not use Pod IPs.

Scale backend:

```bash
kubectl scale deployment backend --replicas=5
```

Verify:

```bash
kubectl get pods
kubectl get endpointslices
```

Delete a backend Pod:

```bash
kubectl delete pod <backend-pod>
```

Observe:

```bash
kubectl get pods -w
```

Then test the frontend again.

---

# 37. Interview Questions and Answers

## Q1. What is a Kubernetes Service?

A Service provides a stable network endpoint for accessing a set of Pods.

## Q2. Why do we need Services?

Pod IPs are dynamic. Services provide stable access even when Pods are replaced or rescheduled.

## Q3. What is the default Service type?

```text
ClusterIP
```

## Q4. What is ClusterIP?

ClusterIP provides an internal virtual IP for accessing a Service inside the cluster.

## Q5. What is NodePort?

NodePort exposes a Service through a port on each node.

## Q6. What is LoadBalancer?

LoadBalancer requests external load-balancing functionality from the cluster environment or cloud integration.

## Q7. What is the difference between `port` and `targetPort`?

`port` is the Service port. `targetPort` is the destination port on the selected Pod.

Example:

```yaml
port: 80
targetPort: 5000
```

means:

```text
Client → Service:80 → Pod:5000
```

## Q8. What is CoreDNS?

CoreDNS provides DNS-based service discovery inside a Kubernetes cluster.

## Q9. What is a Service selector?

A selector identifies the Pods associated with the Service.

```yaml
selector:
  app: backend
```

## Q10. What happens if a Service selector doesn't match any Pods?

The Service has no matching endpoints, so traffic cannot reach the intended Pods through that Service.

## Q11. What are EndpointSlices?

EndpointSlices are Kubernetes API objects used to represent sets of network endpoints associated with Services.

## Q12. What is kube-proxy?

kube-proxy is a node-level component involved in implementing Kubernetes Service networking.

## Q13. What is a headless Service?

A Service with:

```yaml
clusterIP: None
```

It does not provide the normal virtual ClusterIP and can be used for direct endpoint discovery.

## Q14. Why are headless Services useful with StatefulSets?

They can provide DNS-based discovery of individual Pods, which is useful for stateful and distributed applications.

## Q15. Can a Service exist without a selector?

Yes. Endpoints can be managed separately for appropriate use cases.

## Q16. What is session affinity?

Session affinity can cause requests from the same client IP to preferentially reach the same Service endpoint.

Example:

```yaml
sessionAffinity: ClientIP
```

## Q17. What happens when a Pod behind a Service dies?

Its endpoint is removed from the active endpoint set and traffic can be directed to other available endpoints.

## Q18. How does a Pod discover another Service?

Usually through Kubernetes DNS.

Example:

```bash
curl http://backend
```

## Q19. How do you troubleshoot a Service with no traffic?

Check:

```bash
kubectl get pods --show-labels
kubectl describe svc <service>
kubectl get endpointslices
kubectl get pods
kubectl logs <pod>
```

Then test DNS and connectivity from another Pod.

## Q20. What is the difference between Service and Ingress?

A Service provides stable access to Pods. Ingress provides HTTP/HTTPS routing into Services through an Ingress controller.

---

# 38. Interview Scenario

### Question

Your Deployment has three Pods, but your Service has no endpoints. What do you check?

### Answer

Check:

1. Pod labels
2. Service selector
3. Namespace
4. Pod readiness
5. EndpointSlices

Commands:

```bash
kubectl get pods --show-labels
kubectl describe svc backend
kubectl get endpointslices
```

---

# 39. Interview Scenario

### Question

Your Service works, but requests fail with connection refused. What could be wrong?

Possible causes:

- incorrect `targetPort`
- application isn't listening
- application listens on a different port
- NetworkPolicy blocks traffic
- Pod isn't ready
- application crashed

Check:

```bash
kubectl logs <pod>
kubectl describe pod <pod>
kubectl describe svc <service>
```

---

# 40. Interview Scenario

### Question

Why should a frontend call `backend-service` instead of a backend Pod IP?

Because Pod IPs can change.

A Service provides:

- stable network identity
- stable DNS
- endpoint tracking
- access to multiple backend replicas

---

# 41. Interview Scenario

### Question

You scale a Deployment from 3 to 10 replicas. Do you need to manually update the Service?

Normally no.

If the new Pods have labels matching the Service selector, Kubernetes updates the endpoint information automatically.

---

# 42. Common Beginner Mistakes

### Mistake 1 — Hard-coding Pod IPs

Bad:

```text
http://10.244.2.15:5000
```

Better:

```text
http://backend-service:5000
```

### Mistake 2 — Confusing `port` and `targetPort`

Remember:

```text
Service port → Pod port
```

### Mistake 3 — Forgetting selectors

A Service without matching endpoints cannot route traffic to the intended Pods.

### Mistake 4 — Using NodePort everywhere

NodePort is useful, but it isn't automatically the right architecture for every application.

### Mistake 5 — Making every microservice public

Internal microservices should generally use internal communication unless external access is required.

### Mistake 6 — Assuming Running means Ready

A Pod can be Running while its application is not ready to serve requests.

Readiness probes are important for reliable traffic routing.

---

# 43. Important Commands Cheat Sheet

## Services

```bash
kubectl get svc
kubectl get svc -A
kubectl describe svc <name>
kubectl delete svc <name>
```

## EndpointSlices

```bash
kubectl get endpointslices
kubectl describe endpointslice <name>
```

## Pods

```bash
kubectl get pods
kubectl get pods --show-labels
kubectl describe pod <name>
kubectl logs <name>
```

## DNS Testing

```bash
kubectl run dns-test   --image=busybox:1.36   -it --rm   --restart=Never -- sh
```

Inside:

```bash
nslookup backend
wget -qO- http://backend
```

## Scaling

```bash
kubectl scale deployment backend --replicas=5
```

---

# 44. Three-Tier Practical Exercise

Build:

```text
                 Client
                   |
                   v
             Frontend Service
                   |
                   v
             Frontend Pods
                   |
                   v
             Backend Service
                   |
                   v
             Backend Pods
                   |
                   v
             Database Service
                   |
                   v
             Database
```

Requirements:

### Frontend

- Deployment
- 2 replicas
- ClusterIP or LoadBalancer depending on environment

### Backend

- Deployment
- 3 replicas
- ClusterIP

### Database

- Service
- database workload or managed database

### Test

Frontend calls:

```text
http://backend-service
```

Backend calls:

```text
http://database-service
```

Do not hard-code Pod IP addresses.

---

# 45. Homework

## Beginner

1. Create a Deployment with 3 nginx replicas.
2. Create a ClusterIP Service.
3. Resolve the Service using DNS.
4. Curl the Service from another Pod.

## Intermediate

1. Break the Service selector.
2. Diagnose why traffic stops.
3. Fix the selector.
4. Convert the Service to NodePort.
5. Scale the Deployment from 3 to 5 replicas.
6. Observe EndpointSlices.

## Advanced

Build:

```text
Frontend
   |
Backend
   |
Redis
```

Requirements:

- Docker images
- Kubernetes Deployments
- Kubernetes Services
- CoreDNS service discovery
- backend scaling
- Pod failure testing
- AWS ECR
- AWS EKS
- external frontend access
- internal backend and Redis access

Document the architecture and explain every network hop.

---

# 46. Quick Revision

Remember:

```text
Pod
 ↓
Dynamic IP
 ↓
Service
 ↓
Stable IP + DNS
 ↓
EndpointSlices
 ↓
Backend Pods
```

Service types:

```text
ClusterIP
   ↓
Internal

NodePort
   ↓
Node IP + Port

LoadBalancer
   ↓
External/cloud load balancer

ExternalName
   ↓
DNS alias to an external name
```

DNS:

```text
backend
backend.namespace
backend.namespace.svc.cluster.local
```

Troubleshooting:

```text
Pod healthy?
    ↓
Labels correct?
    ↓
Service selector correct?
    ↓
EndpointSlices populated?
    ↓
DNS works?
    ↓
Port/targetPort correct?
    ↓
Application listening?
    ↓
NetworkPolicy/CNI/cloud networking okay?
```

---

# 47. Key Takeaways

1. Pods are temporary; Services provide stable access.
2. Service discovery lets applications communicate without hard-coded Pod IPs.
3. CoreDNS provides Kubernetes DNS-based discovery.
4. Service selectors determine which Pods receive traffic.
5. EndpointSlices track Service endpoints.
6. ClusterIP is the default internal Service type.
7. NodePort exposes a Service through node ports.
8. LoadBalancer integrates Service exposure with external/cloud load-balancing infrastructure.
9. Headless Services support direct endpoint discovery.
10. Session affinity can provide client-IP-based sticky behavior.
11. Services are fundamental to Kubernetes microservice communication.
12. EKS can integrate Kubernetes Services with AWS load-balancing and networking components.
13. Good troubleshooting starts with Pods → labels → Service → EndpointSlices → DNS → ports → application → network.

---

# 48. Final Mental Model

Think of a Kubernetes Service as the stable front desk of your application:

```text
                 Service
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
      Pod A       Pod B       Pod C
```

Pods can:

- die
- restart
- move
- scale
- change IPs

The Service remains the stable entry point.

For microservices:

```text
Frontend
   |
   | DNS
   v
Backend Service
   |
   +----> Backend Pod
   +----> Backend Pod
   +----> Backend Pod
```

The combination of:

```text
Service
+
DNS
+
EndpointSlices
+
Kubernetes networking
```

forms the foundation of Kubernetes service discovery and load balancing.

---

# Next Module

## Module 13 — Managed vs Self-Hosted Kubernetes

Topics:

- What is managed Kubernetes?
- What is self-hosted Kubernetes?
- Control-plane responsibilities
- Worker-node responsibilities
- Managed vs self-hosted comparison
- AWS EKS architecture
- EKS control plane
- EKS managed node groups
- EKS Auto Mode concepts
- Operational responsibility
- Cost considerations
- Security responsibilities
- When to choose each approach
- Practical EKS architecture exercise
- Interview questions
