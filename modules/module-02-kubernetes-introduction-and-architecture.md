# Module 2 — Introduction to Kubernetes and Kubernetes Architecture

## Learning Objectives

By the end of this module, you should be able to:

- Define Kubernetes in simple and interview-ready language.
- Explain what a Kubernetes cluster is.
- Explain control plane and worker nodes.
- Explain API Server, etcd, Scheduler and Controller Manager.
- Explain Kubelet, Kube Proxy and container runtime.
- Understand the basic flow of `kubectl apply`.
- Create a local Kubernetes cluster.
- Create and inspect your first Pod.

---

# 1. What Is Kubernetes?

## Simple Definition

Kubernetes is an open-source platform used to **deploy, manage, scale and operate containerized applications**.

A simpler definition for beginners:

> Kubernetes is a system that manages containers across one or more machines and continuously tries to keep your applications running in the state you requested.

Docker helps us run containers.

Kubernetes helps us manage many containerized applications in a cluster.

---

# 2. Why Is Kubernetes Called K8s?

The word:

```text
Kubernetes
```

has 10 letters.

The abbreviation:

```text
K8s
```

means:

```text
K + 8 letters + s
```

So:

```text
Kubernetes = K8s
```

You may hear both terms in real-world DevOps environments.

---

# 3. What Is a Kubernetes Cluster?

A Kubernetes cluster is a collection of machines that work together to run applications.

Conceptually:

```text
                 Kubernetes Cluster
                        |
          +-------------+-------------+
          |                           |
          v                           v
    Control Plane                 Worker Nodes
                                      |
                         +------------+------------+
                         |            |            |
                         v            v            v
                      Node 1       Node 2       Node 3
                         |            |            |
                        Pods         Pods         Pods
```

The cluster has two major areas:

```text
1. Control Plane
2. Worker Nodes
```

---

# 4. Control Plane vs Worker Node

Think about a company.

```text
                 Company
                    |
          Management / HQ
                    |
              Control Plane
                    |
          +---------+---------+
          |         |         |
        Worker    Worker    Worker
        Node      Node      Node
```

### Control Plane

The control plane makes decisions and maintains the desired state of the cluster.

### Worker Node

Worker nodes actually run application workloads.

A useful mental model:

> **Control Plane = brain/management**

> **Worker Node = machines doing the work**

---

# 5. Control Plane Components

The main control-plane components are:

```text
Control Plane
│
├── kube-apiserver
├── etcd
├── kube-scheduler
└── kube-controller-manager
```

There are also other cluster components and add-ons, but these four are the core control-plane components students should understand first.

---

# 6. kube-apiserver

The Kubernetes API Server is the main entry point to the Kubernetes API.

Almost everything that wants to interact with the Kubernetes control plane goes through the API Server.

For example:

```bash
kubectl get pods
```

The flow is conceptually:

```text
You
 |
 | kubectl get pods
 v
API Server
 |
 v
Cluster state
```

For an operation such as creating a workload:

```bash
kubectl apply -f deployment.yaml
```

the request reaches the API Server.

---

# 7. Real-World Analogy for API Server

Imagine a company reception desk.

Employees do not normally walk directly into the CEO's office and change company records.

They interact through the official interface:

```text
Employee
   |
   v
Reception / Front Desk
   |
   v
Company systems
```

Similarly:

```text
kubectl
   |
   v
API Server
   |
   v
Kubernetes control plane
```

The API Server provides the standard interface for Kubernetes operations.

---

# 8. etcd

`etcd` is a distributed key-value store used by Kubernetes to store important cluster state.

Think of it as the cluster's highly important database for Kubernetes state.

Conceptually it may contain information about:

```text
Pods
Deployments
Services
Nodes
ConfigMaps
Secrets
Cluster configuration
```

Important:

> etcd stores Kubernetes cluster state; it is not your application's normal database.

For example, if your application uses PostgreSQL:

```text
PostgreSQL
   |
   +-- Application data
```

while:

```text
etcd
   |
   +-- Kubernetes cluster state
```

---

# 9. Real-World Analogy for etcd

Imagine a company's official records room.

It contains information such as:

```text
Employees
Departments
Projects
Rules
Assignments
```

The control plane uses its stored state to know what the cluster currently looks like.

---

# 10. kube-scheduler

The Scheduler decides **which worker node should run a newly created Pod**.

Suppose Kubernetes needs to run:

```text
Pod A
```

and there are:

```text
Node 1
Node 2
Node 3
```

The scheduler evaluates available nodes and constraints and selects a suitable node.

Conceptually:

```text
             Pod A
               |
               v
          API Server
               |
               v
           Scheduler
          /    |    \
         /     |     \
      Node1  Node2  Node3
                |
                v
            Selected
```

The Scheduler does not itself run the container.

It chooses a node for the Pod.

---

# 11. Scheduler Analogy

Imagine a delivery company.

```text
Package
   |
   v
Dispatcher
   |
   +-- Driver A
   +-- Driver B
   +-- Driver C
```

The dispatcher decides which driver should receive the package.

Similarly:

```text
Pod
 |
 v
Scheduler
 |
 v
Suitable Worker Node
```

---

# 12. kube-controller-manager

Kubernetes uses controllers to continuously compare:

```text
Desired State
      vs
Current State
```

and take corrective action.

For example:

```text
Desired:
3 replicas

Current:
2 replicas
```

A controller notices the difference and works toward:

```text
Current:
3 replicas
```

This is called **reconciliation**.

---

# 13. Controller Analogy

Imagine a factory manager.

The target is:

```text
100 products
```

Current production:

```text
97 products
```

The manager notices the difference and tells the factory to produce more.

Kubernetes controllers work similarly:

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
Take Action
      |
      v
Desired State
```

---

# 14. Worker Node Components

A worker node commonly contains:

```text
Worker Node
│
├── kubelet
├── kube-proxy
└── Container Runtime
```

The exact implementation and networking behavior can vary by Kubernetes distribution and configuration, but these are the foundational components to learn.

---

# 15. kubelet

The kubelet is the main Kubernetes agent running on a worker node.

Its job includes:

- Receiving/observing Pod specifications assigned to its node.
- Ensuring the containers described by Pods are running.
- Reporting node and Pod status back to the control plane.

Conceptually:

```text
Control Plane
      |
      v
API Server
      |
      v
Kubelet
      |
      v
Container Runtime
      |
      v
Containers
```

---

# 16. Container Runtime

The container runtime is responsible for actually running containers.

Modern Kubernetes uses the **Container Runtime Interface (CRI)** to communicate with supported runtimes.

A common runtime is:

```text
containerd
```

Another runtime is:

```text
CRI-O
```

Important beginner point:

> Kubernetes does not require Docker Engine to run containers.

You may still build your application image with Docker:

```bash
docker build -t myapp:v1 .
```

and Kubernetes can run that image using a CRI-compatible runtime.

---

# 17. kube-proxy

`kube-proxy` is traditionally associated with implementing Service-related networking behavior on nodes.

It helps establish the network rules needed for traffic destined for Kubernetes Services to reach appropriate backend Pods.

A simplified picture:

```text
Client
  |
  v
Service
  |
  v
kube-proxy / networking rules
  |
  +------> Pod 1
  |
  +------> Pod 2
  |
  +------> Pod 3
```

Modern Kubernetes networking implementations can use different mechanisms, and some environments may replace or augment kube-proxy behavior.

For beginner learning, remember:

> kube-proxy is associated with Kubernetes Service networking on nodes.

---

# 18. Complete Architecture

Now combine everything.

```text
                         Kubernetes Cluster
                                |
             +------------------+------------------+
             |                                     |
             v                                     v
       CONTROL PLANE                          WORKER NODES
             |                                     |
      +------+------+                    +---------+---------+
      |      |      |                    |         |         |
      v      v      v                    v         v         v
 API Server etcd Scheduler             Node 1    Node 2    Node 3
      |              |                    |         |         |
      |              |                 Kubelet   Kubelet   Kubelet
      |              |                    |         |         |
      |              |                 Runtime   Runtime   Runtime
      |              |                    |         |         |
      |              +--------------------+---------+---------+
      |                                   |
      v                                   v
 Controller Manager                     Pods
```

---

# 19. What Happens When You Run kubectl apply?

This is an important interview and practical concept.

Suppose we have:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
```

We run:

```bash
kubectl apply -f pod.yaml
```

A simplified flow is:

```text
kubectl
   |
   v
API Server
   |
   v
Authentication / Authorization / Admission
   |
   v
Cluster State
   |
   v
Scheduler
   |
   v
Selected Worker Node
   |
   v
Kubelet
   |
   v
Container Runtime
   |
   v
NGINX Container
```

The API Server accepts and processes the API request.

The object is recorded as cluster state.

The scheduler chooses a suitable node if scheduling is required.

The kubelet on the selected node ensures the Pod is running.

The runtime creates the containers.

---

# 20. Important: Kubernetes Does Not Directly "Run Docker Commands"

A common beginner misconception is:

```text
kubectl
  |
  v
Docker CLI
  |
  v
docker run
```

That is not the right mental model.

Instead:

```text
kubectl
   |
   v
Kubernetes API
   |
   v
Kubernetes control plane
   |
   v
Kubelet
   |
   v
CRI
   |
   v
Container Runtime
```

---

# 21. Installing a Local Kubernetes Environment

For learning, students can use:

- Minikube
- kind
- Docker Desktop Kubernetes

For this course, **kind or Minikube** are good local learning choices.

The exact installation commands depend on the operating system and the tool version, so use the official documentation for the current installation procedure.

---

# 22. First Kubernetes Cluster With kind

If `kind` is installed:

```bash
kind create cluster --name learning
```

Check:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes
```

You should see a node in a ready state.

Example:

```text
NAME                    STATUS   ROLES
learning-control-plane  Ready    control-plane
```

Depending on your local configuration, kind may create a single-node learning cluster or a multi-node cluster if you provide a custom configuration.

---

# 23. First Pod

Create:

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

Save it as:

```text
pod.yaml
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Check:

```bash
kubectl get pods
```

Expected:

```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          ...
```

---

# 24. Inspect the Pod

Run:

```bash
kubectl describe pod nginx
```

This is one of the most useful troubleshooting commands.

It can show:

- Pod details.
- Node assignment.
- Container information.
- Events.
- Image.
- Status.
- Conditions.

---

# 25. View Pod Logs

Run:

```bash
kubectl logs nginx
```

For NGINX, logs may be empty until traffic reaches the server.

---

# 26. Execute a Command Inside the Container

Run:

```bash
kubectl exec -it nginx -- /bin/sh
```

Inside:

```bash
hostname
```

Then:

```bash
exit
```

This is conceptually similar to:

```bash
docker exec -it <container> sh
```

This is a useful Docker-to-Kubernetes connection.

---

# 27. Compare Docker and Kubernetes Commands

| Docker | Kubernetes |
|---|---|
| `docker ps` | `kubectl get pods` |
| `docker logs` | `kubectl logs` |
| `docker exec` | `kubectl exec` |
| `docker inspect` | `kubectl describe` |
| `docker stop` | `kubectl delete` for Kubernetes objects |
| Docker network | Kubernetes networking model |
| Docker Compose | Kubernetes workload/resource manifests |

These are not exact one-to-one equivalents, but the comparison helps Docker users transition to Kubernetes.

---

# 28. Delete the Pod

Run:

```bash
kubectl delete pod nginx
```

Then:

```bash
kubectl get pods
```

The Pod is gone.

This demonstrates an important lesson:

> A standalone Pod is not normally the right abstraction for maintaining an application in production.

Later we will use:

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pods
```

The Deployment/ReplicaSet combination can maintain the desired number of Pods.

---

# 29. Mini Project — Kubernetes Architecture Explorer

## Goal

Students should learn to inspect their cluster rather than just memorizing architecture diagrams.

### Step 1

Create a cluster.

```bash
kind create cluster --name learning
```

### Step 2

Check:

```bash
kubectl get nodes
```

### Step 3

Create the NGINX Pod.

```bash
kubectl apply -f pod.yaml
```

### Step 4

Inspect:

```bash
kubectl get pod nginx -o wide
```

Notice the node where the Pod is running.

### Step 5

Describe it:

```bash
kubectl describe pod nginx
```

Find:

```text
Node:
Image:
Status:
Events:
```

### Step 6

Enter the container:

```bash
kubectl exec -it nginx -- /bin/sh
```

### Step 7

Delete it:

```bash
kubectl delete pod nginx
```

### Student Question

Ask:

> "Who will recreate this Pod?"

Answer:

> Nothing will recreate it because it is a standalone Pod. We need a controller such as a Deployment to maintain replicas.

---

# 30. Mini Project — Observe Desired State

Create a Deployment:

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
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get pods
```

You should have approximately:

```text
web-xxxxx
web-yyyyy
web-zzzzz
```

Delete one:

```bash
kubectl delete pod <one-pod-name>
```

Immediately check:

```bash
kubectl get pods
```

A new Pod should be created by the Deployment's controller machinery.

This is your first practical demonstration of:

```text
Desired State
      ↓
3 replicas
      ↓
Controller observes actual state
      ↓
Pod deleted
      ↓
Only 2 remain
      ↓
Controller creates replacement
      ↓
3 replicas again
```

---

# 31. Interview Questions

## Q1. What is Kubernetes?

**Answer:**

Kubernetes is an open-source platform for automating the deployment, scaling and management of containerized applications across a cluster.

---

## Q2. What is a Kubernetes cluster?

**Answer:**

A Kubernetes cluster is a group of machines consisting of control-plane components and worker nodes that work together to manage and run containerized workloads.

---

## Q3. What is the control plane?

**Answer:**

The control plane contains the components responsible for managing the cluster, processing API requests, scheduling workloads, storing cluster state and running controllers that reconcile desired and actual state.

---

## Q4. What is a worker node?

**Answer:**

A worker node is a machine in the cluster where application workloads, such as Pods, are run.

---

## Q5. What does kube-apiserver do?

**Answer:**

The Kubernetes API Server exposes the Kubernetes API and acts as the central interface through which clients and Kubernetes components interact with cluster state.

---

## Q6. What is etcd?

**Answer:**

etcd is a distributed key-value store used by Kubernetes to persist cluster state.

---

## Q7. What does the Scheduler do?

**Answer:**

The Scheduler selects an appropriate worker node for Pods that need scheduling, based on resources, constraints and scheduling policies.

---

## Q8. What does kubelet do?

**Answer:**

kubelet is the node agent responsible for ensuring that Pods assigned to its node are running and for reporting node and workload status to the control plane.

---

## Q9. Does Kubernetes require Docker Engine?

**Answer:**

No. Kubernetes uses the Container Runtime Interface (CRI), and modern clusters commonly use runtimes such as containerd or CRI-O.

---

## Q10. What is kube-proxy?

**Answer:**

kube-proxy is traditionally responsible for implementing node-level networking rules associated with Kubernetes Services. Modern networking implementations may provide this functionality differently.

---

## Q11. What is reconciliation?

**Answer:**

Reconciliation is the continuous process of comparing desired state with actual state and taking actions to make actual state converge toward desired state.

---

## Q12. What happens when a standalone Pod is deleted?

**Answer:**

If it is not managed by a controller such as a Deployment, nothing automatically recreates it.

---

## Q13. What is the difference between a Pod and a container?

**Answer:**

A container is a process-isolated application environment. A Pod is Kubernetes' basic deployable unit and can contain one or more closely related containers that share networking and certain resources.

---

## Q14. What is the difference between control plane and worker node?

**Answer:**

The control plane manages the cluster and makes orchestration decisions, while worker nodes run application workloads.

---

# 32. Common Beginner Mistakes

### Mistake 1

Thinking:

```text
Kubernetes = Docker
```

Correction:

```text
Docker -> container platform
Kubernetes -> orchestration platform
```

They solve different layers of the problem.

### Mistake 2

Thinking the Scheduler creates containers.

Correction:

> The Scheduler selects a node for a Pod. The kubelet and container runtime on the node participate in actually running the workload.

### Mistake 3

Thinking etcd is the application's database.

Correction:

> etcd stores Kubernetes cluster state.

### Mistake 4

Thinking deleting a Pod always causes Kubernetes to recreate it.

Correction:

> Recreation depends on whether the Pod is managed by an appropriate controller.

---

# 33. Module Summary

Remember:

```text
Kubernetes Cluster
│
├── Control Plane
│   ├── API Server
│   ├── etcd
│   ├── Scheduler
│   └── Controller Manager
│
└── Worker Nodes
    ├── kubelet
    ├── kube-proxy
    ├── Container Runtime
    └── Pods
```

The core mental model:

```text
kubectl
   |
   v
API Server
   |
   +----> etcd
   |
   +----> Scheduler
   |
   +----> Controllers
              |
              v
         Worker Node
              |
            kubelet
              |
        Container Runtime
              |
              v
             Pod
```

---

# 34. Homework

## Theory

Answer:

1. What is Kubernetes?
2. What is K8s?
3. What is a cluster?
4. What is the control plane?
5. What is a worker node?
6. What is kube-apiserver?
7. What is etcd?
8. What is the Scheduler?
9. What does kubelet do?
10. What is a container runtime?
11. What is kube-proxy?
12. What is reconciliation?

## Practical

Create:

```text
3-node local Kubernetes cluster
```

Then:

1. Deploy an NGINX Pod.
2. Find the node on which it runs.
3. Inspect it with `kubectl describe`.
4. View logs.
5. Execute a shell inside it.
6. Delete the Pod.
7. Create a Deployment with 3 replicas.
8. Delete one Pod.
9. Observe the replacement.
10. Explain why the Deployment behaves differently from the standalone Pod.

---

# Next Module

## Module 3 — Kubernetes vs Docker Swarm vs AWS ECS

We will compare:

```text
Kubernetes
Docker Swarm
AWS ECS
```

using:

- Architecture
- Ease of use
- Scalability
- Networking
- Storage
- Security
- AWS integration
- Operational complexity
- Cost
- Real-world use cases

We will also build the same small application using different orchestration approaches and decide which platform is appropriate for different scenarios.
