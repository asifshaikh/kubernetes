# Module 3 — Kubernetes vs Docker Swarm vs AWS ECS

## Learning Objectives

By the end of this module you should be able to:
- Explain Kubernetes, Docker Swarm and Amazon ECS.
- Compare their architecture, complexity, portability and ecosystem.
- Explain ECS vs EKS.
- Decide when each platform is appropriate.
- Deploy a simple application conceptually on all three platforms.
- Answer common orchestration interview questions.

---

# 1. Why Compare Orchestrators?

Docker is excellent for building and running containers. As applications grow, we need scheduling, scaling, service discovery, load balancing, rolling updates and recovery.

Three important technologies are:

```text
Kubernetes
Docker Swarm
Amazon ECS
```

They solve related problems, but their APIs, architectures and operational models are different.

---

# 2. Kubernetes

Kubernetes is an open-source platform for automating deployment, scaling and management of containerized workloads.

Simplified architecture:

```text
                  Kubernetes Cluster
                         |
             +-----------+-----------+
             |                       |
        Control Plane           Worker Nodes
             |                       |
       +-----+------+          +-----+-----+
       | API Server |          |   Kubelet |
       |    etcd    |          |  Runtime  |
       | Scheduler  |          |    Pods   |
       | Controllers|          +-----------+
       +------------+
```

Kubernetes can run on-premises, in cloud environments or through managed services such as Amazon EKS.

---

# 3. Docker Swarm

Docker Swarm is Docker's native clustering and orchestration technology.

```text
                Swarm Cluster
                     |
              +------+------+
              |             |
           Manager       Workers
              |             |
          Scheduling      Services
          State           Containers
```

It provides concepts such as services, replicas, overlay networking, service discovery, load balancing and rolling updates.

Its major advantage is simplicity, especially for teams already familiar with Docker.

---

# 4. Amazon ECS

Amazon Elastic Container Service (ECS) is AWS's container orchestration service.

```text
                     AWS
                      |
                     ECS
                      |
              +-------+-------+
              |               |
            Service        Task
                              |
                         Containers
```

ECS can run workloads using:

```text
EC2
```

or:

```text
AWS Fargate
```

Fargate is a serverless compute option where you do not manage the underlying servers yourself.

---

# 5. ECS vs EKS

This is one of the most important distinctions.

```text
ECS
 |
 +-- AWS-native container orchestration

EKS
 |
 +-- Managed Kubernetes on AWS
```

ECS does not use the Kubernetes API.

EKS provides Kubernetes APIs and Kubernetes resources while AWS manages important control-plane infrastructure.

---

# 6. Kubernetes vs Docker Swarm vs ECS

| Feature | Kubernetes | Docker Swarm | Amazon ECS |
|---|---|---|---|
| Type | Open-source orchestration | Docker-native orchestration | AWS orchestration service |
| Learning curve | Higher | Low | Low/Medium |
| Ecosystem | Very large | Smaller | Strong AWS ecosystem |
| Portability | High | High | Primarily AWS |
| Extensibility | Very high | Lower | AWS-focused |
| Self-hosting | Yes | Yes | No control plane to self-host as ECS |
| Serverless container option | Available through platforms such as Fargate on EKS | No native equivalent | Fargate |
| Kubernetes API | Yes | No | No |
| Best fit | Complex/cloud-native platforms | Simple Docker-centric environments | AWS-centric container workloads |

These are general comparisons, not absolute rankings.

---

# 7. Kubernetes Strengths

Kubernetes is especially strong when an organization needs:

- A large cloud-native ecosystem.
- Multi-cloud or on-premises portability.
- Advanced workload management.
- Extensibility through controllers/operators.
- Kubernetes-native tooling.
- Existing Kubernetes expertise.

Common ecosystem tools include:

```text
Helm
Prometheus
Grafana
Argo CD
Kustomize
Operators
CSI
CNI
```

Kubernetes can also manage many workload types, including Deployments, StatefulSets, Jobs and DaemonSets.

---

# 8. Kubernetes Weaknesses

Kubernetes introduces substantial concepts:

```text
Pods
Deployments
Services
Ingress
Networking
Storage
RBAC
ConfigMaps
Secrets
Scheduling
Controllers
```

A self-hosted cluster also requires significant operational expertise.

Therefore:

> Kubernetes should be selected because its capabilities are useful, not simply because it is popular.

---

# 9. Docker Swarm Strengths and Weaknesses

### Strengths

- Simple architecture.
- Familiar Docker commands/concepts.
- Easy to start for Docker users.
- Provides replicas, service discovery and overlay networking.

### Weaknesses

- Smaller ecosystem.
- Less extensibility than Kubernetes.
- Less commonly selected for new large cloud-native platforms.
- Fewer integrations and advanced capabilities than Kubernetes.

The correct interview statement is:

> Swarm emphasizes simplicity, while Kubernetes emphasizes a broad ecosystem, extensibility and feature depth.

---

# 10. ECS Strengths and Weaknesses

ECS is attractive for AWS-centric organizations because it integrates naturally with AWS services.

A typical environment can contain:

```text
AWS
 |
 +-- ECR       -> Container images
 +-- ECS       -> Container orchestration
 +-- ALB       -> Load balancing
 +-- IAM       -> Permissions
 +-- CloudWatch-> Monitoring/logging
 +-- RDS       -> Managed database
 +-- VPC       -> Networking
 +-- Secrets Manager
```

The main trade-off is that ECS is primarily an AWS platform, so it is less portable than Kubernetes.

---

# 11. Kubernetes vs ECS Mental Model

Kubernetes commonly uses:

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pods
   |
Containers
```

ECS commonly uses:

```text
Service
   |
   v
Tasks
   |
Containers
```

A rough conceptual mapping is useful:

| Kubernetes | ECS |
|---|---|
| Cluster | ECS Cluster |
| Deployment | ECS Service + deployment configuration |
| Pod | Task (conceptual comparison, not exact equivalence) |
| Container image | Container image |
| Service | Service, but with different semantics |
| ConfigMap/Secret | ECS task configuration/secrets mechanisms |

Do not claim that these objects are exact equivalents.

---

# 12. Real-World Scenario: Small Internal App

Suppose a company has:

```text
1 frontend
1 backend
Low traffic
Small team
Few deployments
```

Possible choices include:

```text
VM + Docker
Docker Compose
ECS
```

Kubernetes may be unnecessary.

Lesson:

> More infrastructure does not automatically mean a better architecture.

---

# 13. Real-World Scenario: Large Microservices Platform

Suppose a company has:

```text
100+ services
Multiple teams
Hundreds of replicas
Frequent deployments
Autoscaling
Complex networking
```

Kubernetes may be a strong fit because its ecosystem and extensibility can justify its operational complexity.

On AWS, EKS is a natural managed-Kubernetes option.

---

# 14. Real-World Scenario: AWS-Only Startup

Suppose a small team already uses:

```text
ECR
ALB
IAM
CloudWatch
RDS
VPC
```

and simply wants to run a few containerized applications.

ECS can be attractive because it provides an AWS-native container orchestration experience without requiring the team to learn the full Kubernetes ecosystem.

---

# 15. Real-World Scenario: Existing Kubernetes Organization

Suppose an organization already has:

```text
Kubernetes engineers
Helm charts
Kubernetes manifests
Kubernetes CI/CD
Kubernetes monitoring
```

and is moving workloads to AWS.

EKS can reduce the need to replace their Kubernetes operational model.

---

# 16. Practical Lab — Kubernetes

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
kubectl get deployment
kubectl get pods
```

Scale:

```bash
kubectl scale deployment web --replicas=5
```

Observe:

```bash
kubectl get pods
```

---

# 17. Practical Lab — Docker Swarm

Initialize:

```bash
docker swarm init
```

Create a service:

```bash
docker service create   --name web   --publish published=8080,target=80   --replicas 3   nginx
```

Inspect:

```bash
docker service ls
docker service ps web
```

Scale:

```bash
docker service scale web=5
```

The objective is to compare orchestration concepts rather than become a Swarm specialist.

---

# 18. Practical Lab — ECS Architecture

Understand this AWS workflow:

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
ECS Task Definition
   |
   v
ECS Service
   |
   +-- Task 1
   +-- Task 2
   +-- Task 3
```

For the later AWS labs, students will create these resources in a controlled AWS account.

---

# 19. Mini Project — Same Application on Three Platforms

Use a simple NGINX application.

Target:

```text
3 replicas/tasks
HTTP traffic
```

### Kubernetes

Create:

```text
Deployment
Service
```

Verify:

```bash
kubectl get pods
kubectl get services
```

### Docker Swarm

Create:

```text
Swarm Service
```

Verify:

```bash
docker service ls
docker service ps web
```

### ECS

Create:

```text
ECR repository
Task Definition
ECS Service
```

Later place the service behind an Application Load Balancer.

---

# 20. Mini Project Architecture on AWS

A basic ECS architecture:

```text
                  Internet
                     |
                     v
             Application Load
                 Balancer
                     |
                     v
                ECS Service
                     |
          +----------+----------+
          |          |          |
        Task       Task       Task
          |          |          |
          +----------+----------+
                     |
                  Backend
```

Suggested AWS components:

```text
Docker
ECR
ECS
ALB
CloudWatch
```

---

# 21. Interview Questions

## Q1. What is Kubernetes?

Kubernetes is an open-source platform for automating deployment, scaling and management of containerized workloads.

## Q2. What is Docker Swarm?

Docker Swarm is Docker's native clustering and container orchestration technology.

## Q3. What is Amazon ECS?

Amazon Elastic Container Service is AWS's managed service for running and managing containerized workloads.

## Q4. ECS vs EKS?

ECS is AWS's native container orchestration service. EKS is AWS's managed Kubernetes service.

## Q5. Which is easier, ECS or Kubernetes?

There is no universal answer, but ECS can be simpler for AWS-centric teams because it provides an AWS-native model. Kubernetes has more concepts and a larger ecosystem.

## Q6. Which is more portable?

Kubernetes is generally more portable because Kubernetes APIs and implementations are available across multiple cloud and on-premises environments.

## Q7. Why choose EKS over ECS?

Reasons can include existing Kubernetes expertise, Kubernetes ecosystem tooling, portability and Kubernetes-specific capabilities.

## Q8. Why choose ECS over EKS?

Reasons can include AWS-only infrastructure, simpler operations and not needing Kubernetes-specific features.

## Q9. What is Fargate?

Fargate is an AWS serverless compute option for running containers without managing the underlying servers directly.

## Q10. What is ECR?

Amazon Elastic Container Registry is AWS's container image registry.

## Q11. Is EKS the same as ECS?

No. EKS runs managed Kubernetes; ECS is AWS's native container orchestration service.

## Q12. Is a Kubernetes Pod exactly the same as an ECS Task?

No. They are useful conceptual comparisons, but their semantics and implementation differ.

## Q13. Is Kubernetes always better than ECS?

No. The appropriate choice depends on requirements, skills, ecosystem, portability, operational complexity and cost.

---

# 22. Interview Scenario

### Scenario A

A startup has five developers, three containerized applications and AWS-only infrastructure.

**Possible answer:**

Consider ECS first if Kubernetes-specific capabilities are not required. It can provide a simpler AWS-native container platform.

### Scenario B

An enterprise has hundreds of services, multiple teams, Kubernetes expertise and multi-cloud requirements.

**Possible answer:**

Kubernetes is likely a stronger fit because its portability, ecosystem and extensibility can justify the additional operational complexity. On AWS, EKS is an option.

---

# 23. Common Beginner Mistakes

### Mistake 1

```text
EKS = ECS
```

Incorrect.

```text
ECS -> AWS container orchestration

EKS -> Managed Kubernetes
```

### Mistake 2

```text
Kubernetes is an AWS service.
```

Incorrect.

Kubernetes is open source. EKS is AWS's managed Kubernetes service.

### Mistake 3

```text
ECS is Kubernetes with different commands.
```

Incorrect. They have different APIs and abstractions.

### Mistake 4

Choosing Kubernetes only because it is popular.

Instead evaluate:

```text
Requirements
   ↓
Team skills
   ↓
Operational complexity
   ↓
Portability
   ↓
Cost
   ↓
Technology choice
```

---

# 24. Module Summary

```text
Docker
   |
   | Runs containers
   v
Orchestration
   |
   +------------------+------------------+
   |                  |                  |
   v                  v                  v
Kubernetes        Docker Swarm        ECS
   |                  |                  |
Flexible           Simple             AWS-native
Extensible         Docker-based       AWS ecosystem
Portable
```

On AWS:

```text
AWS
 |
 +-- ECS -> AWS-native orchestration
 |
 +-- EKS -> Managed Kubernetes
```

Key lesson:

> There is no universally best orchestrator. Choose according to requirements, team skills, ecosystem, portability, operational complexity and cost.

---

# 25. Homework

## Theory

Answer:

1. What is Kubernetes?
2. What is Docker Swarm?
3. What is ECS?
4. What is EKS?
5. ECS vs EKS?
6. Kubernetes vs ECS?
7. Kubernetes vs Docker Swarm?
8. What is Fargate?
9. What is ECR?
10. Why choose ECS instead of EKS?
11. Why choose EKS instead of ECS?
12. Why is Kubernetes portable?

## Practical

### Challenge 1

Deploy NGINX with 3 replicas using Kubernetes.

### Challenge 2

Deploy NGINX with 3 replicas using Docker Swarm.

### Challenge 3

Draw an ECS architecture for 3 NGINX tasks behind an Application Load Balancer.

### Challenge 4

Write a one-page recommendation for this situation:

> "Our startup is AWS-only, has five developers, three containerized applications and limited DevOps expertise. Should we use ECS or EKS?"

Support your recommendation with at least five reasons.

---

# Next Module

## Module 4 — When to Use Kubernetes

Topics:

- When Kubernetes is a good choice.
- When Kubernetes is unnecessary.
- Kubernetes adoption decision framework.
- Microservices vs monoliths.
- High availability.
- Scaling requirements.
- Multi-team environments.
- On-premises vs cloud.
- Operational complexity.
- Cost and engineering effort.
- Real-world architecture scenarios.
- Interview scenarios.
- Mini project: choosing the right platform for fictional companies.
