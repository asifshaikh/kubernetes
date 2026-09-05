# Module 1 — Why Container Orchestration?

## Learning Objectives

By the end of this module, you should be able to:

- Explain what container orchestration means.
- Explain why Docker alone is not enough for large applications.
- Understand the problems Kubernetes solves.
- Explain scaling, self-healing, service discovery and load balancing at a high level.
- Compare manual Docker management with orchestration.

---

# 1. First: What Problem Does Docker Solve?

Before Kubernetes, understand Docker.

Suppose we have a Python application:

```text
Application
    |
    +-- Python
    +-- Python libraries
    +-- OS dependencies
```

Different machines may have different versions of Python or different libraries.

Docker packages the application and its dependencies into an image:

```text
Docker Image
    |
    +-- Application
    +-- Dependencies
    +-- Runtime
```

The image can then be started as a container:

```text
Docker Image
      |
      v
   Container
```

For example:

```bash
docker run -d -p 8080:80 nginx
```

Docker is very good at:

- Packaging applications.
- Running containers.
- Isolating applications.
- Managing images.
- Connecting containers through networks.
- Attaching storage volumes.

But Docker primarily solves the **container runtime and packaging problem**.

---

# 2. What Happens When the Application Becomes Large?

Imagine an online shopping application.

It has:

```text
Frontend
Backend
User Service
Payment Service
Order Service
Redis
Database
```

Initially, you might have:

```text
1 frontend container
1 backend container
1 user-service container
1 payment-service container
```

Everything looks easy.

But the application becomes popular.

Now you need:

```text
10 frontend containers
20 backend containers
5 user-service containers
10 payment-service containers
```

And these containers may run across several servers:

```text
                 Application
                     |
       +-------------+-------------+
       |             |             |
    Server 1      Server 2      Server 3
       |             |             |
   Containers     Containers     Containers
```

Now manually managing everything becomes difficult.

---

# 3. Problems With Manually Managing Containers

## Problem 1 — Container Failure

Suppose:

```text
Backend Container
       X
    CRASHED
```

A user may receive an error.

Someone has to notice the problem and start another container.

With orchestration:

```text
Desired:
3 backend containers

Current:
2 backend containers

Orchestrator:
"Only 2 are running. Start another."

Result:
3 backend containers
```

This is called **self-healing**.

---

# 4. Problem 2 — Scaling

Suppose your application normally receives:

```text
100 requests/second
```

You may need:

```text
3 backend containers
```

During a sale:

```text
10,000 requests/second
```

Three containers may not be enough.

You may need:

```text
3 replicas
      |
      v
10 replicas
```

An orchestrator can increase the number of application instances.

This is called **scaling**.

---

# 5. Problem 3 — Load Balancing

Suppose we have:

```text
Backend 1
Backend 2
Backend 3
```

Users should not all connect to Backend 1.

Traffic should be distributed:

```text
                 Users
                   |
                   v
             Load Balancer
              /     |     \
             v      v      v
          Backend Backend Backend
             1       2       3
```

An orchestrator can help provide service discovery and load balancing mechanisms.

---

# 6. Problem 4 — Server Failure

Imagine:

```text
Server 1
  |
  +-- Backend
  +-- Payment
```

Server 1 crashes:

```text
Server 1
   X
```

The application should continue running if enough capacity exists elsewhere.

A cluster orchestrator can detect unhealthy nodes and reschedule workloads when appropriate.

---

# 7. Problem 5 — Deploying New Versions

Suppose version 1 is running:

```text
backend:v1
backend:v1
backend:v1
```

You build:

```text
backend:v2
```

You do not want to immediately stop every v1 container.

A production orchestrator can perform controlled updates:

```text
v1  v1  v1
 |   |   |
 v   v   v

v2  v1  v1

v2  v2  v1

v2  v2  v2
```

This is the basic idea behind a **rolling update**.

---

# 8. Problem 6 — Service Discovery

Suppose the backend needs Redis.

You could manually configure:

```text
redis -> 10.20.30.40
```

But what happens if Redis moves to another server?

The IP may change.

Orchestration systems provide stable service-discovery mechanisms so applications can find other applications without hardcoding changing container IP addresses.

---

# 9. Problem 7 — Managing Hundreds or Thousands of Containers

Imagine:

```text
Company
   |
   +-- 50 applications
          |
          +-- 10 containers each
```

That is:

```text
500 containers
```

Now imagine:

```text
5000 containers
```

Manually running:

```bash
docker run ...
docker stop ...
docker start ...
docker rm ...
```

is not a practical management strategy.

You need a system that continuously maintains the desired state.

---

# 10. What Is Container Orchestration?

### Interview Definition

> Container orchestration is the automated management of containerized workloads, including deployment, scheduling, scaling, networking, service discovery, health management and recovery.

In simple words:

> **Container orchestration is like a manager for containers.**

Docker can run containers.

An orchestrator manages **many containers across a cluster**.

---

# 11. Real-World Analogy

Imagine a restaurant.

### Containers = Chefs

Each chef performs a task.

```text
Chef 1 -> Pizza
Chef 2 -> Burger
Chef 3 -> Pasta
```

### Servers/Nodes = Kitchens

```text
Kitchen 1
Kitchen 2
Kitchen 3
```

### Kubernetes = Restaurant Manager

The manager decides:

- Which kitchen should receive work?
- How many chefs are needed?
- What happens if a chef stops working?
- How should orders be distributed?
- How should more chefs be added during rush hour?

That is similar to what Kubernetes does for containers.

---

# 12. Docker vs Container Orchestration

Think of the relationship like this:

```text
Docker
  |
  +-- Build image
  +-- Run container
  +-- Stop container
  +-- Manage container
```

Kubernetes adds cluster-level management:

```text
Kubernetes
   |
   +-- Schedule containers
   +-- Maintain desired replicas
   +-- Restart failed workloads
   +-- Scale applications
   +-- Service discovery
   +-- Load balancing
   +-- Rolling updates
   +-- Manage workloads across nodes
```

Important:

> Kubernetes does not replace the concept of containers. Kubernetes orchestrates containerized workloads.

Modern Kubernetes clusters commonly use containerd or another CRI-compatible runtime rather than the Docker Engine itself.

---

# 13. What Does "Desired State" Mean?

This is one of the most important Kubernetes ideas.

Suppose you say:

```text
I want 3 copies of my application.
```

That is the **desired state**.

Kubernetes continuously observes the actual state.

### Desired

```text
3 Pods
```

### Actual

```text
2 Pods
```

Kubernetes attempts to make:

```text
Actual State = Desired State
```

So it creates another Pod.

```text
Before:

Pod 1
Pod 2

After:

Pod 1
Pod 2
Pod 3
```

This is a fundamental idea behind Kubernetes controllers.

---

# 14. Declarative Management

Kubernetes commonly uses a declarative approach.

Instead of saying:

```text
Create container.
Then create another.
Then start another.
```

you describe what you want:

```yaml
replicas: 3
```

Kubernetes works out the actions required to reach that state.

### Imperative thinking

```text
DO THIS
DO THIS
DO THIS
```

### Declarative thinking

```text
I WANT THIS FINAL STATE
```

Kubernetes continuously works toward the desired state.

---

# 15. First Practical Exercise — Docker

Run an NGINX container:

```bash
docker run -d --name web1 -p 8080:80 nginx
```

Check it:

```bash
docker ps
```

Open:

```text
http://localhost:8080
```

Stop it:

```bash
docker stop web1
```

Check:

```bash
docker ps
```

The application is no longer running.

Docker does not automatically provide the full cluster-level reconciliation system that Kubernetes provides.

---

# 16. Practical Exercise — Simulating Replicas With Docker

Run three containers:

```bash
docker run -d --name web1 -p 8081:80 nginx
docker run -d --name web2 -p 8082:80 nginx
docker run -d --name web3 -p 8083:80 nginx
```

Check:

```bash
docker ps
```

You now have:

```text
web1 -> :8081
web2 -> :8082
web3 -> :8083
```

Now remove one:

```bash
docker rm -f web2
```

Check:

```bash
docker ps
```

You now have only:

```text
web1
web3
```

Nothing automatically recreated `web2`.

This demonstrates why a reconciliation/orchestration system is useful.

---

# 17. Mini Project 1 — Manual Docker Application

## Goal

Run a simple application with multiple containers.

Architecture:

```text
              User
                |
                v
          Frontend Container
                |
                v
          Backend Container
                |
                v
          Redis Container
```

Use:

- NGINX for frontend.
- Python/Flask for backend.
- Redis for a simple cache.

## Student Tasks

1. Create a Docker network.
2. Run Redis.
3. Run a backend container.
4. Run an NGINX frontend container.
5. Make the frontend communicate with the backend.
6. Make the backend communicate with Redis.
7. Stop the backend container.
8. Observe what happens.
9. Explain what Kubernetes would automate.

Example network:

```bash
docker network create app-network
```

Run Redis:

```bash
docker run -d \
  --name redis \
  --network app-network \
  redis
```

The exact frontend/backend implementation can be added as a later lab.

---

# 18. Mini Project 2 — Docker Failure Simulation

Run:

```text
3 backend containers
```

Then randomly stop one:

```bash
docker stop backend2
```

Ask students:

1. Who detects the failure?
2. Who creates a replacement?
3. How would traffic be redirected?
4. How would the desired number of replicas be maintained?

Expected discussion:

```text
Docker:
Runs containers.

Kubernetes:
Can continuously reconcile the desired application state.
```

---

# 19. When Do We Need Orchestration?

Orchestration becomes increasingly valuable when you have:

- Multiple services.
- Multiple replicas.
- Multiple machines.
- High availability requirements.
- Frequent deployments.
- Automatic scaling requirements.
- Service discovery requirements.
- Automated recovery requirements.

For a tiny application:

```text
One server
One container
Low traffic
```

Kubernetes may be unnecessary complexity.

---

# 20. Kubernetes Is Not Always the Answer

Do not teach students:

> "Always use Kubernetes."

Instead teach:

> "Choose Kubernetes when its operational benefits justify its complexity and cost."

For example:

### Simple website

```text
NGINX
   |
One VM
```

Kubernetes may be excessive.

### Large microservices platform

```text
Frontend
Backend
Payments
Orders
Users
Messaging
Workers
Multiple replicas
Multiple nodes
Autoscaling
```

Kubernetes becomes much more attractive.

---

# 21. Interview Questions and Answers

## Q1. Why do we need container orchestration?

**Answer:**

Container orchestration automates deployment, scheduling, scaling, networking, service discovery, health management and recovery of containers across a cluster.

---

## Q2. Is Docker a container orchestrator?

**Answer:**

Docker is primarily a container platform and runtime ecosystem. Docker Compose can manage multi-container applications on a host, while Docker Swarm provides orchestration. Kubernetes is a separate, full-featured container orchestration platform.

---

## Q3. What problem does Kubernetes solve?

**Answer:**

Kubernetes manages containerized workloads across a cluster and helps maintain the desired state through scheduling, reconciliation, scaling, service discovery, rolling updates and recovery.

---

## Q4. What is self-healing?

**Answer:**

Self-healing means Kubernetes can detect certain failed or unhealthy workloads and take corrective actions, such as recreating Pods to restore the desired state.

---

## Q5. What is scaling?

**Answer:**

Scaling means changing the number of application instances according to workload requirements.

Example:

```text
3 replicas -> 10 replicas
```

---

## Q6. What is the desired state?

**Answer:**

The desired state is the state the user declares for the application, such as the number of replicas, image version and configuration. Kubernetes continuously works to make the actual state match the desired state.

---

## Q7. What is declarative management?

**Answer:**

Declarative management means specifying the desired end state rather than manually specifying every operation needed to reach that state.

---

## Q8. Why not simply use Docker?

**Answer:**

Docker is excellent for building and running containers, but large distributed applications also require cluster-level capabilities such as scheduling, service discovery, scaling, reconciliation and workload management.

---

## Q9. What happens when a container crashes in Kubernetes?

**Answer:**

The exact behavior depends on the workload and Pod configuration. Kubernetes and the relevant controllers/runtime can restart containers or recreate Pods to move the system back toward the desired state.

---

## Q10. Is Kubernetes only for microservices?

**Answer:**

No. Kubernetes can run many types of workloads, including web applications, batch jobs, APIs, workers and stateful applications. Microservices are simply a common use case.

---

# 22. Key Takeaways

Remember these five ideas:

```text
Docker
  |
  | Packages and runs containers
  v

Kubernetes
  |
  +-- Schedules workloads
  +-- Maintains desired state
  +-- Scales applications
  +-- Helps recover failed workloads
  +-- Provides service discovery/networking primitives
  +-- Supports controlled deployments
```

The most important mental model is:

> **Docker gives you containers. Kubernetes manages containerized workloads at cluster scale.**

---

# Module 1 Lab Checklist

Students should be able to demonstrate:

- [ ] Build/run a Docker container.
- [ ] Run multiple containers.
- [ ] Create a Docker network.
- [ ] Explain why manual replica management becomes difficult.
- [ ] Simulate a container failure.
- [ ] Explain self-healing.
- [ ] Explain scaling.
- [ ] Explain desired state.
- [ ] Explain declarative management.
- [ ] Explain why Kubernetes is useful.

---

# Homework

## Theory

Answer:

1. What is container orchestration?
2. Why is Docker alone insufficient for large distributed systems?
3. What is desired state?
4. What is self-healing?
5. What is declarative configuration?
6. When should you avoid Kubernetes?

## Practical

Create:

```text
frontend
backend
redis
```

using Docker.

Then:

1. Put them on one Docker network.
2. Start all services.
3. Stop the backend.
4. Explain what breaks.
5. Write a short explanation of how Kubernetes could solve the operational problem.

---

# Next Module

## Module 2 — Introduction to Kubernetes and Architecture

We will learn:

- What Kubernetes actually is.
- Kubernetes cluster.
- Control plane.
- Worker nodes.
- API Server.
- etcd.
- Scheduler.
- Controller Manager.
- Kubelet.
- Kube Proxy.
- Container runtime.
- How a request travels through a Kubernetes cluster.
- Creating the first local Kubernetes cluster.
