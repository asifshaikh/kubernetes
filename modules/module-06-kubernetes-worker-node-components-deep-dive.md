# Module 6 — Kubernetes Worker Node Components: Deep Dive

## Learning Objectives

By the end of this module, students should be able to:

- Explain what a Kubernetes Worker Node is.
- Explain kubelet, kube-proxy, Container Runtime and CRI.
- Understand how Pods are scheduled onto Worker Nodes.
- Use node labels, nodeSelector, taints and tolerations.
- Understand CPU/memory requests and limits.
- Inspect node health and Pod placement.
- Troubleshoot Pending Pods and unhealthy nodes.
- Connect Kubernetes Worker Nodes to AWS EKS.

---

## 1. What Is a Worker Node?

A **Worker Node** is a machine where Kubernetes actually runs application workloads.

Think of a Kubernetes cluster like a company:

- **Control Plane** = management team that makes decisions.
- **Worker Nodes** = production machines that do the work.
- **Pod** = application unit.
- **kubelet** = worker/node supervisor.
- **Container Runtime** = software that actually runs containers.
- **kube-proxy** = node networking component.

The Control Plane says:

> "Run this Pod on Worker Node 2."

Worker Node 2 then makes sure that Pod actually runs.

```text
Kubernetes Cluster
       |
       +--------------------+
       |                    |
 Control Plane          Worker Node
 "Decides"              "Runs"
                            |
              +-------------+-------------+
              |             |             |
           kubelet      kube-proxy   Container Runtime
                                         |
                                         v
                                        Pods
```

---

# 2. Worker Node Architecture

A typical Worker Node contains:

```text
+------------------------------------------------+
|                 Worker Node                    |
|                                                |
|  +-------------+     +----------------------+  |
|  |   kubelet   |     |     kube-proxy       |  |
|  +-------------+     +----------------------+  |
|          |                    |                |
|          +----------+---------+                |
|                     |                          |
|             Container Runtime                 |
|          containerd / CRI-O / etc.             |
|                     |                          |
|             +-------+-------+                  |
|             |       |       |                  |
|            Pod     Pod     Pod                 |
|             |       |       |                  |
|         Container Container Container          |
+------------------------------------------------+
```

The key components are:

1. kubelet
2. kube-proxy
3. Container Runtime
4. CRI
5. Pods
6. Node CPU, memory, disk and networking

---

# 3. kubelet

## Definition

**kubelet is the agent running on every Worker Node that makes sure Pods assigned to that node are running correctly.**

### Real-world analogy

Imagine a restaurant manager receives an order:

> "Prepare three pizzas."

The manager:

1. Reads the order.
2. Checks the kitchen.
3. Starts the required work.
4. Watches the workers.
5. Reports status.

kubelet performs a similar role for Pods on a Worker Node.

### kubelet responsibilities

kubelet:

- Registers the node with the Kubernetes API Server.
- Watches for Pods assigned to its node.
- Requests the Container Runtime to start and stop containers.
- Monitors Pod/container health.
- Reports node and Pod status to the API Server.
- Manages configured volume mounts.
- Participates in Pod lifecycle management.

### Important distinction

The kubelet does **not** choose which node should run a Pod.

The **Scheduler** makes that decision.

```text
Scheduler:
"Pod should run on Worker Node 2."

kubelet on Worker Node 2:
"Okay, I will make sure this Pod runs."
```

---

# 4. Container Runtime

A **Container Runtime** is software responsible for running containers.

Common Kubernetes-compatible runtimes include:

- containerd
- CRI-O

Modern Kubernetes does not require Docker Engine.

Docker can still be used to build images:

```text
Dockerfile
   |
docker build
   |
Container Image
   |
Registry
   |
Kubernetes
   |
containerd / CRI-O
   |
Container
```

The important distinction is:

> Docker can build and package an image, while the Kubernetes node needs a compatible runtime to run containers.

---

# 5. CRI — Container Runtime Interface

**CRI is the interface through which kubelet communicates with a compatible Container Runtime.**

Conceptually:

```text
kubelet
   |
   | CRI
   v
Container Runtime
   |
   +-- containerd
   +-- CRI-O
   |
   v
Containers
```

### Interview definition

> CRI is a Kubernetes interface that allows kubelet to communicate with compatible container runtimes.

This abstraction means kubelet does not need to understand the internal implementation of every runtime.

---

# 6. kube-proxy

**kube-proxy is a node-level networking component that helps implement Kubernetes Service networking.**

Suppose we have:

```text
Frontend Pods
      |
      v
Service: backend
      |
      v
Backend Pods
```

A Service provides a stable virtual endpoint even though backend Pods can be created, deleted or replaced.

kube-proxy helps configure node networking rules so Service traffic can reach appropriate backend Pods.

Depending on the Kubernetes environment/configuration, Service traffic handling may involve mechanisms such as:

- iptables
- IPVS
- nftables

### Important distinction

kube-proxy is **not** an Ingress Controller.

- **Service** = stable networking abstraction.
- **kube-proxy** = helps implement Service traffic handling on nodes.
- **Ingress Controller** = handles HTTP/HTTPS routing into the cluster.

---

# 7. Pods on Worker Nodes

A Pod is Kubernetes' smallest deployable unit.

A Pod can contain one or more containers:

```text
Pod
 |
 +-- Container 1
 |
 +-- Container 2
```

Containers in the same Pod share:

- Network namespace.
- Pod IP.
- `localhost`.
- Volumes when configured.

Example:

```text
+-----------------------------+
|            Pod              |
|                             |
|  +---------+  +----------+  |
|  |  nginx  |  | sidecar  |  |
|  +---------+  +----------+  |
|                             |
|       Pod IP: 10.x.x.x      |
+-----------------------------+
```

Most applications use one primary application container per Pod.

---

# 8. How a Pod Reaches a Worker Node

Consider a Deployment creating a Pod:

```text
1. kubectl apply
       |
       v
2. API Server
       |
       v
3. Deployment Controller
       |
       v
4. ReplicaSet
       |
       v
5. Pod created
       |
       v
6. Scheduler selects Worker Node
       |
       v
7. kubelet sees assigned Pod
       |
       v
8. kubelet talks to Container Runtime
       |
       v
9. Image is pulled
       |
       v
10. Container starts
       |
       v
11. kubelet monitors it
       |
       v
12. Status reported to API Server
```

This flow is extremely important for troubleshooting and interviews.

---

# 9. Node Registration

When kubelet starts, it communicates with the Kubernetes API Server and registers the node.

Check nodes:

```bash
kubectl get nodes
```

Example:

```text
NAME           STATUS   ROLES           AGE   VERSION
control-plane  Ready    control-plane   20m   v1.xx.x
worker-01      Ready    <none>          18m   v1.xx.x
worker-02      Ready    <none>          18m   v1.xx.x
```

Get detailed information:

```bash
kubectl describe node worker-01
```

---

# 10. Node Conditions

Node conditions describe the health of a node.

Common conditions:

- Ready
- MemoryPressure
- DiskPressure
- PIDPressure
- NetworkUnavailable

Check them:

```bash
kubectl describe node <node-name>
```

Example:

```text
Conditions:
  Type             Status
  ----             ------
  MemoryPressure   False
  DiskPressure     False
  PIDPressure      False
  Ready            True
```

### What does Ready=True mean?

It generally means Kubernetes considers the node healthy and available to run Pods.

---

# 11. Node Labels

Labels are key-value metadata attached to Kubernetes objects.

Add a label:

```bash
kubectl label node worker-01 workload=backend
```

Check labels:

```bash
kubectl get nodes --show-labels
```

Now the node has:

```text
workload=backend
```

A Pod can request that label:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:
  nodeSelector:
    workload: backend
  containers:
    - name: nginx
      image: nginx:alpine
```

This tells Kubernetes:

> Schedule this Pod only onto a node with `workload=backend`.

---

# 12. Taints and Tolerations

Labels can help select nodes.

Taints work in the opposite direction:

> "Keep Pods away from this node unless they explicitly tolerate the taint."

Apply a taint:

```bash
kubectl taint nodes worker-01 dedicated=database:NoSchedule
```

A normal Pod should not be scheduled there.

A Pod can tolerate it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: database
      effect: NoSchedule

  containers:
    - name: database
      image: postgres:16
```

### Simple analogy

```text
Node:
"Database team only."

Taint:
"Do not enter unless authorized."

Toleration:
"This employee is authorized."
```

### Very important

A toleration does **not** force a Pod onto a node.

It only means the Pod is allowed to run on a node with the matching taint.

---

# 13. nodeSelector vs Taints/Tolerations

These concepts are often confused.

### nodeSelector

```text
"Put me on nodes matching this label."
```

### Taint

```text
"Keep Pods away from this node."
```

### Toleration

```text
"This Pod is allowed onto a tainted node."
```

You can combine them:

```yaml
nodeSelector:
  workload: database

tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
```

Now the Pod both:

- Selects database nodes.
- Is permitted to run on the tainted database node.

---

# 14. Resource Requests and Limits

Worker Nodes have finite resources.

Example:

```text
Worker Node
-----------------------
CPU:    4 cores
Memory: 8 GiB
```

Pods consume these resources.

## Requests

A request tells Kubernetes approximately:

> "Reserve/consider this amount when deciding where I can run."

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

`500m` means 0.5 CPU core.

## Limits

A limit specifies a maximum resource amount for the container:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

Complete example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

### Interview distinction

**Requests influence scheduling.**

**Limits constrain runtime resource usage.**

For CPU, exceeding a CPU limit generally results in throttling.

For memory, exceeding a memory limit can result in the container being terminated due to an out-of-memory condition.

---

# 15. How the Scheduler Uses Resources

Suppose:

```text
Node 1:
CPU = 4
Available allocatable capacity = 2

Node 2:
CPU = 4
Available allocatable capacity = 0.5
```

A Pod requests:

```text
CPU = 1
```

Node 1 is a possible candidate based on CPU request, while Node 2 may not be.

The scheduler considers many factors, including:

- Resource requests.
- Node selectors.
- Node affinity.
- Taints and tolerations.
- Topology constraints.
- Scheduling policies/plugins.

So scheduling is much more than simply:

> "Find a node with free CPU."

---

# 16. Node Affinity

Node affinity provides more expressive node selection.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: workload
                operator: In
                values:
                  - backend

  containers:
    - name: nginx
      image: nginx:alpine
```

This means:

> The Pod must be scheduled onto a node labeled `workload=backend`.

---

# 17. Worker Node Failure

Imagine:

```text
Cluster
+----------+----------+
|                     |
Worker 1              Worker 2
  DOWN                  UP
```

Kubernetes detects that Worker 1 is unhealthy.

For replicated workloads such as a Deployment, Kubernetes can recreate/reschedule Pods according to the desired state if suitable cluster capacity exists.

Conceptually:

```text
Before:
Worker 1 -> Pod A
Worker 2 -> Pod B

Worker 1 fails

After reconciliation:
Worker 2 -> Pod A
Worker 2 -> Pod B
```

But remember:

> Kubernetes does not magically guarantee application high availability.

You still need appropriate:

- Replica counts.
- Cluster capacity.
- Scheduling configuration.
- Health checks.
- Storage architecture.
- Networking.

---

# 18. Practical Lab 1 — Inspect Worker Nodes

Run:

```bash
kubectl get nodes
```

Then:

```bash
kubectl get nodes -o wide
```

And:

```bash
kubectl describe node <node-name>
```

Find:

1. Node IP.
2. Kubernetes version.
3. Container runtime.
4. Allocatable CPU.
5. Allocatable memory.
6. Node labels.
7. Node conditions.
8. Whether the node is Ready.

---

# 19. Practical Lab 2 — Schedule a Pod Using a Label

Label a node:

```bash
kubectl label node <node-name> workload=backend
```

Verify:

```bash
kubectl get nodes --show-labels
```

Create `backend-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:
  nodeSelector:
    workload: backend
  containers:
    - name: nginx
      image: nginx:alpine
```

Apply:

```bash
kubectl apply -f backend-pod.yaml
```

Check placement:

```bash
kubectl get pod backend-pod -o wide
```

---

# 20. Practical Lab 3 — Taint a Node

Apply:

```bash
kubectl taint nodes <node-name> dedicated=database:NoSchedule
```

Create a normal Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: normal-pod
spec:
  containers:
    - name: nginx
      image: nginx:alpine
```

Apply:

```bash
kubectl apply -f normal-pod.yaml
```

If there is no other suitable node, the Pod may remain:

```text
Pending
```

Inspect:

```bash
kubectl describe pod normal-pod
```

Check the Events section.

Remove the taint:

```bash
kubectl taint nodes <node-name> dedicated=database:NoSchedule-
```

---

# 21. Practical Lab 4 — Resource Requests and Limits

Create `resource-demo.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

Apply:

```bash
kubectl apply -f resource-demo.yaml
```

Check:

```bash
kubectl get pods -o wide
```

Inspect:

```bash
kubectl describe pod <pod-name>
```

---

# 22. Practical Lab 5 — Troubleshoot a Pending Pod

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      resources:
        requests:
          memory: "999Gi"
```

Apply:

```bash
kubectl apply -f impossible-pod.yaml
```

Check:

```bash
kubectl get pod impossible-pod
```

It should remain Pending in a normal small cluster.

Now inspect:

```bash
kubectl describe pod impossible-pod
```

Look at:

```text
Events
```

The events should help explain why the scheduler cannot find a suitable node.

### Troubleshooting rule

Do not stop at:

```bash
kubectl get pods
```

For a Pending Pod, immediately try:

```bash
kubectl describe pod <pod-name>
```

---

# 23. Practical Lab 6 — Find Pod-to-Node Placement

Run:

```bash
kubectl get pods -o wide
```

Example:

```text
NAME             READY   STATUS    NODE
frontend-abc     1/1     Running   worker-01
frontend-def     1/1     Running   worker-02
backend-abc      1/1     Running   worker-01
```

This answers:

> "Where is my application actually running?"

---

# 24. Worker Node Troubleshooting Checklist

### Step 1 — Check nodes

```bash
kubectl get nodes
```

### Step 2 — Describe the node

```bash
kubectl describe node <node-name>
```

### Step 3 — Check conditions

Look for:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
```

### Step 4 — Check Pods

```bash
kubectl get pods -A -o wide
```

### Step 5 — Check events

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

### Step 6 — Inspect node services

On a Linux worker node, depending on the environment:

```bash
systemctl status kubelet
systemctl status containerd
```

Logs:

```bash
journalctl -u kubelet
journalctl -u containerd
```

These commands require access to the actual Worker Node.

---

# 25. AWS EKS Connection

In Amazon EKS, worker capacity can be provided through approaches such as:

- EKS Managed Node Groups.
- Self-managed nodes.
- AWS-managed compute capabilities such as EKS Auto Mode, depending on the cluster setup.

For a traditional EKS Managed Node Group architecture:

```text
AWS
 |
 +-- EKS Control Plane
 |
 +-- VPC
      |
      +-- Subnets
           |
           +-- EC2 Worker Nodes
                 |
                 +-- kubelet
                 +-- kube-proxy
                 +-- Container Runtime
                 +-- Pods
```

EKS removes much of the control-plane management burden, but your workloads still require suitable compute capacity.

EKS worker infrastructure can integrate with:

- EC2.
- IAM.
- VPC networking.
- EBS/EFS storage.
- AWS Load Balancers.

These integrations become important in later modules.

---

# 26. Mini Project — Worker Node Investigation

## Objective

Practice determining **where Pods run and why they run there**.

### Task 1

Create a 3-replica Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: worker-demo
  template:
    metadata:
      labels:
        app: worker-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

### Task 2

Find where Pods are running:

```bash
kubectl get pods -o wide
```

### Task 3

Inspect all nodes:

```bash
kubectl get nodes -o wide
```

### Task 4

Describe a node:

```bash
kubectl describe node <node-name>
```

### Task 5

Add:

```text
workload=frontend
```

to one node.

### Task 6

Create a Pod with:

```yaml
nodeSelector:
  workload: frontend
```

### Task 7

Taint that node:

```bash
kubectl taint nodes <node-name> dedicated=frontend:NoSchedule
```

Observe what happens to Pods that do not tolerate the taint.

### Task 8

Add a matching toleration and test again.

---

# 27. Interview Questions and Answers

## Q1. What is a Kubernetes Worker Node?

**Answer:** A Worker Node is a machine that runs Kubernetes workloads. It normally runs kubelet, a Container Runtime and node networking functionality such as kube-proxy.

## Q2. What does kubelet do?

**Answer:** kubelet is the node agent responsible for ensuring Pods assigned to its node are running and for reporting node/Pod status to the API Server.

## Q3. Does kubelet schedule Pods?

**Answer:** No. The Scheduler selects a node. kubelet manages Pods assigned to that node.

## Q4. What is a Container Runtime?

**Answer:** Software that runs containers, such as containerd or CRI-O.

## Q5. Does Kubernetes require Docker?

**Answer:** No. Kubernetes can use CRI-compatible runtimes such as containerd and CRI-O. Docker can still be used to build container images.

## Q6. What is CRI?

**Answer:** CRI stands for Container Runtime Interface. It provides the interface through which kubelet communicates with a compatible Container Runtime.

## Q7. What does kube-proxy do?

**Answer:** kube-proxy is a node networking component that helps implement Kubernetes Service networking and direct Service traffic toward backend Pods.

## Q8. What is a node label?

**Answer:** A key-value metadata pair attached to a node. Scheduling rules such as nodeSelector and node affinity can use node labels.

## Q9. What is a taint?

**Answer:** A taint marks a node so Pods without a matching toleration are prevented from being scheduled there, depending on the taint effect.

## Q10. What is a toleration?

**Answer:** A toleration allows a Pod to be scheduled onto a node with a matching taint. It does not by itself force the Pod onto that node.

## Q11. What is the difference between requests and limits?

**Answer:**

- Request: resource amount considered during scheduling.
- Limit: maximum runtime resource amount imposed on a container.

## Q12. Why is a Pod Pending?

Possible causes include:

- Insufficient CPU or memory.
- Node selector mismatch.
- Node affinity rules.
- Untolerated taints.
- Topology constraints.
- Other scheduling restrictions.

Use:

```bash
kubectl describe pod <pod-name>
```

and inspect Events.

## Q13. What happens when a Worker Node fails?

**Answer:** Kubernetes detects node health changes and controllers such as Deployments can recreate/reschedule workloads according to desired state when suitable capacity and scheduling conditions exist.

## Q14. Can multiple containers exist in one Pod?

**Answer:** Yes. Containers in a Pod share the Pod network namespace and can share volumes. Multi-container Pods are useful for tightly coupled containers such as an application and sidecar.

## Q15. How do you find which node is running a Pod?

```bash
kubectl get pods -o wide
```

---

# 28. Common Beginner Mistakes

### Mistake 1: Thinking kubelet chooses the node

Incorrect.

The Scheduler chooses the node.

### Mistake 2: Thinking Kubernetes equals Docker

Incorrect.

Docker is a container ecosystem/tooling platform. Kubernetes is a container orchestration platform.

### Mistake 3: Thinking a toleration means "run here"

Incorrect.

A toleration means the Pod is allowed to run on a node with the matching taint.

### Mistake 4: Ignoring resource requests

Requests are central to understanding scheduling and Pending Pods.

### Mistake 5: Checking only Pod status

Instead of only:

```bash
kubectl get pods
```

learn to use:

```bash
kubectl describe pod <pod>
kubectl get events -A
kubectl get pods -o wide
kubectl describe node <node>
```

---

# 29. Quick Revision

```text
Worker Node
 |
 +-- kubelet
 |     |
 |     +-- Watches assigned Pods
 |     +-- Talks to API Server
 |     +-- Talks to Container Runtime
 |     +-- Reports status
 |
 +-- Container Runtime
 |     |
 |     +-- Runs containers
 |     +-- containerd / CRI-O
 |
 +-- kube-proxy
 |     |
 |     +-- Helps implement Service networking
 |
 +-- Pods
       |
       +-- Containers
```

### Scheduling concepts

```text
nodeSelector
     |
     +--> Select nodes by labels

nodeAffinity
     |
     +--> Advanced node selection

taint
     |
     +--> Keep Pods away

toleration
     |
     +--> Allow Pod onto tainted nodes

resource request
     |
     +--> Influences scheduling
```

---

# 30. Homework

1. Explain kubelet in your own words.
2. Explain CRI and why Kubernetes uses it.
3. Explain containerd vs Docker Engine conceptually.
4. Explain kube-proxy.
5. Create a node label and schedule a Pod using nodeSelector.
6. Create a taint and observe a Pod becoming Pending.
7. Add a toleration and schedule the Pod.
8. Create a Deployment with CPU/memory requests.
9. Use `kubectl describe pod` to find a scheduling problem.
10. Explain what happens from Scheduler selection until a container starts.

---

# Module 6 Takeaway

The most important mental model is:

> **The Control Plane decides where a workload should run; the Worker Node executes and monitors it.**

Remember:

```text
Scheduler
    |
    v
Worker Node
    |
    v
kubelet
    |
    v
Container Runtime
    |
    v
Container
    |
    v
Application
```

For Service networking:

```text
Service
   |
   v
Node networking
   |
   v
Backend Pods
```

Once you understand the Worker Node, troubleshooting becomes much easier because you can reason about:

1. Where the Pod was scheduled.
2. Whether the node has enough resources.
3. Whether scheduling rules permit placement.
4. Whether kubelet is healthy.
5. Whether the Container Runtime can start the container.
6. Whether networking and storage are functioning.

---

# Next Module

## Module 7 — Kubernetes API and kubectl Basics

Topics:

- Kubernetes API.
- API Server interaction.
- kubectl.
- Kubernetes resources.
- YAML manifests.
- Imperative vs declarative commands.
- `kubectl get`.
- `kubectl describe`.
- `kubectl logs`.
- `kubectl exec`.
- `kubectl apply`.
- `kubectl delete`.
- API groups and versions.
- Practical API exploration.
- Interview questions.
- Hands-on labs.
