# Module 5 — Kubernetes Control Plane Components: Deep Dive

## Learning Objectives

By the end of this module, students should be able to:

- Explain the purpose of the Kubernetes control plane.
- Explain kube-apiserver, etcd, kube-scheduler and kube-controller-manager in detail.
- Understand how these components interact.
- Explain authentication, authorization and admission at a high level.
- Understand reconciliation and controller loops.
- Trace what happens when `kubectl apply` is executed.
- Troubleshoot basic control-plane-related problems.
- Answer control-plane interview questions confidently.

---

# 1. What Is the Control Plane?

The Kubernetes control plane is the management layer of the cluster.

It is responsible for things such as:

```text
Receiving API requests
Storing cluster state
Scheduling Pods
Running controllers
Maintaining desired state
```

Think of it as the **management and decision-making system** of Kubernetes.

```text
                    Kubernetes Cluster
                           |
              +------------+------------+
              |                         |
              v                         v
        CONTROL PLANE              WORKER NODES
              |                         |
       Makes decisions             Runs workloads
       Stores state                    |
       Reconciles state               Pods
```

---

# 2. Main Control Plane Components

The core components are:

```text
Control Plane
│
├── kube-apiserver
├── etcd
├── kube-scheduler
└── kube-controller-manager
```

There are other components and add-ons in a real cluster, but these four are the foundation.

---

# 3. kube-apiserver

## What Is It?

The Kubernetes API Server exposes the Kubernetes API.

It is the primary interface through which:

- `kubectl`
- Kubernetes controllers
- Scheduler
- Operators
- Other clients

interact with the cluster.

Think:

```text
API Server = Front door of Kubernetes
```

---

# 4. API Server Analogy

Imagine a bank.

Customers don't directly modify the bank's database.

Instead:

```text
Customer
   |
   v
Bank Counter
   |
   v
Bank Systems
```

Similarly:

```text
User
 |
kubectl
 |
 v
API Server
 |
 v
Kubernetes State
```

The API Server acts as the central API gateway into Kubernetes.

---

# 5. What Happens When You Run kubectl?

Suppose:

```bash
kubectl get pods
```

Conceptually:

```text
kubectl
   |
   | HTTPS API request
   v
API Server
   |
   v
Authenticate
   |
   v
Authorize
   |
   v
Admission (where applicable)
   |
   v
Read cluster state
   |
   v
Return response
```

The API Server does not itself "run the Pod."

It handles the API interaction.

---

# 6. Authentication

Authentication answers:

> **Who are you?**

Examples:

```text
User
Service Account
Certificate
OIDC identity
Cloud identity integration
```

Conceptually:

```text
Request
   |
   v
Who are you?
   |
   v
Authenticated identity
```

---

# 7. Authorization

Authorization answers:

> **Are you allowed to perform this action?**

For example:

```text
User: alice

Can Alice:
GET Pods?       YES
DELETE Pods?    NO
CREATE Secrets? NO
```

Kubernetes commonly uses RBAC for authorization.

RBAC means:

> Role-Based Access Control

We will study RBAC in a later module.

---

# 8. Admission

After authentication and authorization, Kubernetes can apply admission controls to API requests.

Admission can:

```text
Validate requests
Modify requests
Enforce policies
```

Conceptually:

```text
Request
   |
Authentication
   |
Authorization
   |
Admission
   |
API operation
```

Admission controllers can enforce organizational rules.

For example:

```text
All containers must have resource limits.
```

A policy can reject a workload that does not comply.

---

# 9. etcd

`etcd` is a distributed key-value store used by Kubernetes to persist cluster state.

Think:

```text
etcd = Kubernetes cluster state store
```

It contains information about Kubernetes objects and other important cluster state.

Examples include:

```text
Pods
Deployments
Services
Nodes
ConfigMaps
Secrets
Namespaces
```

The exact storage representation is internal; users normally interact with these objects through the Kubernetes API rather than directly editing etcd.

---

# 10. Important: etcd Is Not Your Application Database

Suppose your application uses PostgreSQL.

```text
Application
    |
    v
PostgreSQL
    |
    +-- Users
    +-- Orders
    +-- Payments
```

Kubernetes has:

```text
Kubernetes
    |
    v
etcd
    |
    +-- Cluster state
```

Therefore:

```text
PostgreSQL -> Application data

etcd -> Kubernetes state
```

Do not confuse them.

---

# 11. Why Is etcd So Important?

Imagine the control plane loses the information describing:

```text
What Deployments exist?
What Services exist?
What Nodes exist?
What configuration exists?
```

The cluster would have a serious state-management problem.

Therefore etcd is a critical component.

In production environments, etcd availability, backup and recovery are important operational concerns for self-managed control planes.

---

# 12. kube-scheduler

The Scheduler decides which node should run a newly scheduled Pod.

Suppose:

```text
Pod: payment-api
```

Available nodes:

```text
Node 1
Node 2
Node 3
```

The Scheduler evaluates the available nodes and chooses an appropriate one.

```text
              Payment Pod
                   |
                   v
               Scheduler
              /    |    \
             /     |     \
         Node 1  Node 2  Node 3
                    |
                    v
                Selected
```

---

# 13. Scheduler Does Not Run Containers

This is an important interview point.

Incorrect:

```text
Scheduler
   |
   v
Runs container
```

Better mental model:

```text
Scheduler
   |
   v
Chooses node for Pod
   |
   v
Kubelet on selected node
   |
   v
Container runtime
   |
   v
Container
```

---

# 14. How Does the Scheduler Choose a Node?

The scheduler considers scheduling constraints and suitability.

Examples:

```text
CPU availability
Memory availability
Pod resource requests
Node selectors
Affinity
Anti-affinity
Taints and tolerations
Topology constraints
Scheduling policies
```

Example:

```text
GPU Pod
   |
   v
Requires GPU node
   |
   v
Scheduler
   |
   v
GPU-capable Node
```

---

# 15. Resource Requests and Scheduling

Suppose a Pod requests:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The scheduler uses resource requests when determining whether a node is suitable.

If:

```text
Node A -> insufficient available requested capacity
Node B -> enough capacity
```

the scheduler can choose:

```text
Node B
```

Important:

> Requests influence scheduling. Limits primarily constrain runtime resource usage rather than directly determining scheduling placement.

---

# 16. kube-controller-manager

The Controller Manager runs Kubernetes controller processes.

A controller follows a basic pattern:

```text
Observe
  |
  v
Compare desired vs current
  |
  v
Take corrective action
  |
  v
Observe again
```

This is called:

> **Reconciliation**

---

# 17. Desired State vs Current State

Suppose a Deployment says:

```text
replicas: 3
```

Current state:

```text
Pod 1
Pod 2
```

The controller sees:

```text
Desired = 3
Current = 2
```

It works toward:

```text
Desired = 3
Current = 3
```

This is one of the most important Kubernetes concepts.

---

# 18. Controller Loop

Think of a thermostat.

You set:

```text
Desired temperature = 24°C
```

Current temperature:

```text
28°C
```

The thermostat takes action.

Later:

```text
Current = 24°C
```

It continues observing.

Kubernetes controllers behave similarly:

```text
Desired State
      |
      v
Controller
      |
      v
Observe Current State
      |
      v
Calculate Difference
      |
      v
Take Action
      |
      v
Observe Again
```

---

# 19. Controllers Are Specialized

The Controller Manager coordinates multiple controller processes.

Examples include controllers associated with:

```text
Deployments/ReplicaSets
Nodes
Jobs
Namespaces
Service accounts
and other Kubernetes resources
```

Students do not need to memorize every controller at this stage.

Understand the central principle:

> Controllers continuously reconcile resources toward their desired state.

---

# 20. Complete Control Plane Interaction

Now combine the components.

```text
                         kubectl
                            |
                            v
                     kube-apiserver
                       /    |    \
                      /     |     \
                     v      v      v
                   etcd  Scheduler Controllers
                     |       |         |
                     |       |         |
                     +-------+---------+
                             |
                             v
                        Worker Nodes
```

This is a simplified conceptual diagram.

---

# 21. Example: Creating a Deployment

Suppose we apply:

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
          image: nginx:latest
```

Run:

```bash
kubectl apply -f deployment.yaml
```

What happens?

---

# 22. Step 1 — kubectl Sends the Request

```text
kubectl
   |
   v
API Server
```

The request contains the desired Deployment object.

---

# 23. Step 2 — API Request Processing

The API Server processes the request through the relevant request pipeline.

Conceptually:

```text
Request
   |
Authentication
   |
Authorization
   |
Admission
   |
API processing
```

If accepted, the object becomes part of cluster state.

---

# 24. Step 3 — State Is Persisted

The API Server persists the relevant state in etcd.

Conceptually:

```text
API Server
    |
    v
etcd

Deployment:
  name = web
  replicas = 3
```

---

# 25. Step 4 — Controller Notices the Deployment

The Deployment controller sees the desired object.

It determines that the required ReplicaSet/Pods need to exist.

Conceptually:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
3 Pods
```

---

# 26. Step 5 — Scheduler Selects Nodes

The new Pods need nodes.

The scheduler evaluates the Pods and available nodes.

```text
Pod 1 -> Node A
Pod 2 -> Node B
Pod 3 -> Node C
```

The actual result depends on cluster configuration and scheduling constraints.

---

# 27. Step 6 — Kubelet Runs the Pod

On each selected node:

```text
Kubelet
   |
   v
Container Runtime
   |
   v
NGINX Container
```

The kubelet works to ensure the Pod is running.

---

# 28. Step 7 — Status Comes Back

The node reports status through Kubernetes mechanisms.

Conceptually:

```text
Pod running
    |
    v
Kubelet
    |
    v
API Server
    |
    v
Cluster state
```

Now:

```text
Desired:
3 Pods

Actual:
3 Running Pods
```

The system has converged.

---

# 29. What If a Pod Dies?

Suppose:

```text
web-1   X
web-2
web-3
```

Desired:

```text
3 Pods
```

Current:

```text
2 Pods
```

The controller detects the difference.

```text
Desired = 3
Current = 2
```

It creates a replacement Pod.

```text
web-new
```

Eventually:

```text
web-1
web-2
web-3
```

or a different set of Pod names representing three replicas.

---

# 30. What If a Node Dies?

Suppose:

```text
Node A X
Node B
Node C
```

Pods previously running on Node A may become unavailable.

Controllers and node-related control-plane logic can react according to Kubernetes behavior and configured policies.

If the workload is managed by a controller and sufficient healthy capacity exists, replacement Pods can be scheduled elsewhere.

This is why multiple nodes and replicas are important for resilient applications.

---

# 31. Practical Lab — Observe Controllers

Create:

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
          image: nginx:latest
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get deployment
kubectl get replicasets
kubectl get pods
```

You should see the relationship:

```text
Deployment
    |
    v
ReplicaSet
    |
    +-- Pod
    +-- Pod
    +-- Pod
```

---

# 32. Practical Lab — Demonstrate Reconciliation

Delete one Pod:

```bash
kubectl delete pod <pod-name>
```

Immediately run:

```bash
kubectl get pods
```

You should see Kubernetes working toward three replicas again.

Watch continuously:

```bash
kubectl get pods -w
```

This is one of the best beginner demonstrations of Kubernetes reconciliation.

---

# 33. Practical Lab — Observe Scheduling

Run:

```bash
kubectl get pods -o wide
```

You can see the node assigned to each Pod.

Example:

```text
NAME       READY   STATUS    NODE
web-a      1/1     Running   node1
web-b      1/1     Running   node2
web-c      1/1     Running   node1
```

The exact placement depends on the cluster.

---

# 34. Practical Lab — Inspect Events

Run:

```bash
kubectl describe pod <pod-name>
```

Look at:

```text
Events
```

Events can help explain things such as:

```text
Scheduled
Pulled
Created
Started
FailedScheduling
```

Events are extremely useful during troubleshooting.

---

# 35. Troubleshooting Scenario — Pod Pending

Suppose:

```bash
kubectl get pods
```

shows:

```text
web-xxxxx   0/1   Pending
```

First inspect:

```bash
kubectl describe pod web-xxxxx
```

Look at:

```text
Events
```

Possible reasons can include:

```text
Insufficient CPU
Insufficient memory
Taints
Affinity rules
No suitable node
Storage constraints
```

The correct troubleshooting method is:

```text
Observe
  |
  v
Describe
  |
  v
Read Events
  |
  v
Identify constraint
  |
  v
Fix configuration/capacity
```

---

# 36. Troubleshooting Scenario — ImagePullBackOff

Suppose:

```text
STATUS
ImagePullBackOff
```

Check:

```bash
kubectl describe pod <pod-name>
```

Possible causes:

```text
Wrong image name
Wrong image tag
Private registry authentication problem
Registry unavailable
Network issue
```

This is usually not a scheduler problem.

---

# 37. Troubleshooting Scenario — CrashLoopBackOff

Suppose:

```text
STATUS
CrashLoopBackOff
```

Check:

```bash
kubectl logs <pod-name>
```

Also:

```bash
kubectl describe pod <pod-name>
```

Possible causes:

```text
Application crash
Incorrect configuration
Missing environment variable
Bad command/arguments
Dependency unavailable
```

Again, distinguish:

```text
Scheduling problem
vs
Application runtime problem
```

---

# 38. Useful Control Plane Commands

Students should become comfortable with:

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get replicasets
kubectl describe pod <name>
kubectl logs <name>
kubectl get events
kubectl api-resources
kubectl get --raw='/readyz?verbose'
```

The exact availability of diagnostic endpoints and commands can vary by Kubernetes distribution and permissions.

---

# 39. Mini Project — Trace an Application From Command to Container

## Goal

Students must explain the complete path.

Start with:

```bash
kubectl apply -f deployment.yaml
```

Draw:

```text
kubectl
   |
   v
API Server
   |
   +--> Authentication
   |
   +--> Authorization
   |
   +--> Admission
   |
   v
etcd
   |
   v
Controller
   |
   v
ReplicaSet
   |
   v
Pods
   |
   v
Scheduler
   |
   v
Worker Node
   |
   v
Kubelet
   |
   v
Container Runtime
   |
   v
Container
```

Students should explain each arrow in their own words.

---

# 40. Mini Project — Break and Diagnose

Create a Deployment using an intentionally invalid image:

```yaml
containers:
  - name: web
    image: nginx:this-tag-does-not-exist
```

Apply:

```bash
kubectl apply -f broken.yaml
```

Check:

```bash
kubectl get pods
```

Then:

```bash
kubectl describe pod <pod-name>
```

and:

```bash
kubectl logs <pod-name>
```

Students should determine why the application did not start.

Then fix the image:

```yaml
image: nginx:latest
```

Apply again:

```bash
kubectl apply -f broken.yaml
```

Verify:

```bash
kubectl get pods
```

---

# 41. Interview Questions

## Q1. What is the Kubernetes control plane?

**Answer:**

The control plane is the management layer of Kubernetes. It processes API requests, stores cluster state, schedules Pods and runs controllers that reconcile desired and actual state.

---

## Q2. What does kube-apiserver do?

**Answer:**

It exposes the Kubernetes API and acts as the primary interface for clients and Kubernetes components to interact with cluster state.

---

## Q3. What is etcd?

**Answer:**

etcd is a distributed key-value store used by Kubernetes to persist cluster state.

---

## Q4. What does the Scheduler do?

**Answer:**

The Scheduler selects suitable nodes for Pods that need scheduling based on resource requirements, constraints and scheduling policies.

---

## Q5. Does the Scheduler start containers?

**Answer:**

No. The Scheduler selects a node. The kubelet on that node works with the container runtime to run the Pod's containers.

---

## Q6. What is kube-controller-manager?

**Answer:**

It runs Kubernetes controller processes that continuously reconcile actual cluster state toward the desired state.

---

## Q7. What is reconciliation?

**Answer:**

Reconciliation is the process of comparing desired state with observed actual state and taking corrective actions so that actual state converges toward desired state.

---

## Q8. What is authentication?

**Answer:**

Authentication determines the identity making a request.

---

## Q9. What is authorization?

**Answer:**

Authorization determines whether the authenticated identity is permitted to perform a requested action.

---

## Q10. What is admission?

**Answer:**

Admission is the stage where Kubernetes can validate and/or mutate API requests before they are persisted and acted upon.

---

## Q11. Is etcd a normal application database?

**Answer:**

No. etcd stores Kubernetes cluster state. Application data should normally be stored in an appropriate application database or storage system.

---

## Q12. What happens when a Deployment has 3 replicas and one Pod is deleted?

**Answer:**

The Deployment's controller machinery detects that fewer replicas exist than desired and works to create a replacement Pod.

---

## Q13. What happens when a Pod is Pending?

**Answer:**

Investigate why it has not been scheduled or started. A good first step is:

```bash
kubectl describe pod <pod-name>
```

and inspect Events.

---

## Q14. What is the difference between API Server and etcd?

**Answer:**

The API Server provides the Kubernetes API and handles API requests. etcd is the persistent key-value store used for Kubernetes cluster state.

---

# 42. Interview Scenario

### Question

A developer says:

> "I created a Deployment with 3 replicas, but one Pod crashed. Who creates the replacement?"

### Answer

The controller machinery associated with the Deployment/ReplicaSet observes that the actual number of Pods is below the desired number and works to create a replacement.

---

### Question

A Pod is Pending. Which component should you think about first?

### Answer

Think about scheduling and inspect the Pod's Events:

```bash
kubectl describe pod <pod-name>
```

The Scheduler is responsible for selecting a suitable node, but Pending can have several causes, so inspect evidence rather than guessing.

---

### Question

The API Server is unavailable. Can normal `kubectl` cluster operations work?

### Answer

Most Kubernetes API operations require access to the API Server, so an unavailable API Server severely affects cluster management. Existing containers may continue running depending on the failure and node/runtime state, but normal control-plane operations cannot proceed normally.

---

# 43. Common Beginner Mistakes

## Mistake 1

> Scheduler runs containers.

Incorrect.

Scheduler selects a node.

---

## Mistake 2

> etcd stores application data.

Incorrect.

etcd stores Kubernetes cluster state.

---

## Mistake 3

> Controller Manager directly runs containers.

Incorrect.

Controllers reconcile resources. Node components and container runtimes participate in actually running containers.

---

## Mistake 4

> API Server is just for kubectl.

Incorrect.

Many Kubernetes components and clients interact through the Kubernetes API.

---

## Mistake 5

> Authentication and authorization are the same.

Incorrect.

```text
Authentication -> Who are you?

Authorization -> What are you allowed to do?
```

---

# 44. The Most Important Diagram

Memorize the flow, not just the names:

```text
                     kubectl
                        |
                        v
                 kube-apiserver
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
        Auth         Admission      etcd
       checks                       state
          |
          v
      API request
                        |
                        v
                 Controllers
                        |
                        v
                   ReplicaSet
                        |
                        v
                      Pods
                        |
                        v
                   Scheduler
                        |
                        v
                   Worker Node
                        |
                     kubelet
                        |
                Container Runtime
                        |
                        v
                    Container
```

Remember:

```text
API Server -> API
etcd       -> State
Scheduler  -> Placement
Controllers-> Reconciliation
Kubelet    -> Node execution
Runtime    -> Containers
```

---

# 45. Module Summary

The four major control-plane components:

```text
kube-apiserver
    |
    +-- Kubernetes API

etcd
    |
    +-- Cluster state

kube-scheduler
    |
    +-- Pod placement

kube-controller-manager
    |
    +-- Reconciliation
```

The central Kubernetes loop is:

```text
Desired State
      |
      v
API Server
      |
      v
Persisted State
      |
      v
Controllers
      |
      v
Actual State Changes
      |
      v
Observe Again
      |
      +---------> Desired State
```

This reconciliation model is one of the foundations of Kubernetes.

---

# 46. Homework

## Theory

Answer:

1. What is the Kubernetes control plane?
2. What is kube-apiserver?
3. What is etcd?
4. What does kube-scheduler do?
5. What does kube-controller-manager do?
6. What is reconciliation?
7. Authentication vs authorization?
8. What is admission?
9. Why is etcd critical?
10. Does Scheduler run containers?
11. Does Kubernetes require Docker Engine?
12. What happens when a Deployment loses one replica?

## Practical

### Lab 1

Create a 3-replica NGINX Deployment.

```bash
kubectl apply -f deployment.yaml
kubectl get deployment
kubectl get replicasets
kubectl get pods -o wide
```

### Lab 2

Delete one Pod.

```bash
kubectl delete pod <pod-name>
```

Watch:

```bash
kubectl get pods -w
```

Explain the reconciliation process.

### Lab 3

Deploy an invalid image.

```yaml
image: nginx:this-tag-does-not-exist
```

Diagnose it using:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Lab 4

Write a diagram showing what happens after:

```bash
kubectl apply -f deployment.yaml
```

Your diagram must include:

```text
kubectl
API Server
etcd
Controller
ReplicaSet
Scheduler
Kubelet
Container Runtime
Pod
```

---

# Next Module

## Module 6 — Kubernetes Worker Node Components: Deep Dive

We will study:

- Worker node architecture
- kubelet
- kube-proxy
- Container Runtime
- CRI
- Pods on a node
- Node lifecycle
- Node conditions
- Node labels
- Taints and tolerations
- Resource requests and limits
- Node troubleshooting
- Practical labs
- Interview questions
- Mini project: inspect and troubleshoot a Kubernetes worker node
