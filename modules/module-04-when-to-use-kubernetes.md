# Module 4 — When to Use Kubernetes

## Learning Objectives

By the end of this module, students should be able to:

- Explain when Kubernetes is a good technology choice.
- Explain when Kubernetes may be unnecessary.
- Understand the operational cost of adopting Kubernetes.
- Evaluate application size, traffic, availability and team requirements.
- Distinguish between a business requirement and a technology preference.
- Recommend Kubernetes, EKS, ECS, Docker/VMs or another platform for realistic scenarios.
- Answer Kubernetes architecture decision interview questions.

---

# 1. The Most Important Question

A beginner often asks:

> "Is Kubernetes the best way to deploy applications?"

The better question is:

> **"Does Kubernetes solve problems that are important enough for this organization to justify its complexity and cost?"**

Kubernetes is powerful, but powerful systems also introduce concepts, infrastructure and operational responsibility.

---

# 2. Start With a Simple Application

Imagine a small company website:

```text
                    Internet
                       |
                       v
                    NGINX
                       |
                       v
                  Web Application
                       |
                       v
                    Database
```

Traffic:

```text
50 users/day
```

Application:

```text
1 frontend
1 backend
1 database
```

Team:

```text
2 developers
0 Kubernetes engineers
```

Do we automatically need Kubernetes?

**No.**

A simpler architecture could be:

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
VM
   |
 Docker
   |
Application
   |
Database
```

The engineering principle is:

> **Use the simplest architecture that satisfies the actual requirements.**

---

# 3. When Kubernetes Starts Becoming Valuable

Kubernetes becomes increasingly attractive when we have several of these requirements:

```text
Multiple services
        +
Multiple replicas
        +
Multiple machines
        +
Frequent deployments
        +
Autoscaling
        +
High availability
        +
Service discovery
        +
Complex scheduling
        +
Large engineering teams
```

It is not necessary to have every requirement.

---

# 4. Requirement 1 — Many Microservices

Imagine:

```text
                  Application
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
   User Service    Order Service   Payment Service
       |               |               |
       +---------------+---------------+
                       |
                    Database
```

Now imagine:

```text
100 services
```

Each service may have:

```text
3 replicas
```

That's approximately:

```text
300 application instances
```

Managing this manually becomes difficult.

Kubernetes provides standardized APIs and controllers for managing these workloads.

---

# 5. Requirement 2 — High Availability

Suppose:

```text
Node 1
  |
  +-- Backend Pod
```

If Node 1 fails:

```text
Node 1
   X
```

A highly available architecture should continue serving users if capacity exists elsewhere.

With multiple replicas:

```text
Node 1       Node 2       Node 3
   |            |            |
 Backend      Backend      Backend
   |            |            |
   +------------+------------+
                |
              Users
```

If one node fails:

```text
Node 1 X       Node 2       Node 3
                |            |
             Backend      Backend
```

The remaining replicas can continue serving traffic.

Kubernetes provides mechanisms that help build this type of architecture.

Important:

> **Kubernetes does not magically make an application highly available.**

The application and cluster must be designed correctly.

---

# 6. Requirement 3 — Frequent Deployments

Suppose a company deploys:

```text
20 times/day
```

Manually updating servers can become risky.

Kubernetes Deployments can support controlled rollout patterns such as:

```text
Version 1
   |
   v
Replace gradually
   |
   v
Version 2
```

This can reduce deployment downtime and provide rollback mechanisms.

Later modules will cover:

- Rolling updates
- Rollbacks
- Blue-green deployments
- Canary-style deployment patterns

---

# 7. Requirement 4 — Autoscaling

Imagine an application receives:

```text
100 requests/minute
```

Normally:

```text
3 replicas
```

During a sale:

```text
10,000 requests/minute
```

You may need:

```text
3 replicas
     |
     v
10 replicas
```

After traffic falls:

```text
10 replicas
     |
     v
3 replicas
```

Kubernetes supports autoscaling mechanisms such as the Horizontal Pod Autoscaler.

Example:

```bash
kubectl autoscale deployment web \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

Important:

> Autoscaling requires appropriate metrics and resource configuration. Kubernetes does not automatically understand business traffic without being configured with suitable metrics.

---

# 8. Requirement 5 — Multiple Machines

One server might be enough for:

```text
Small application
```

But a large application may need:

```text
Node 1
Node 2
Node 3
Node 4
Node 5
```

Kubernetes provides a common control plane and scheduling model across the cluster.

```text
                  Kubernetes
                       |
          +------------+------------+
          |            |            |
        Node 1       Node 2       Node 3
          |            |            |
         Pods         Pods         Pods
```

The Scheduler can choose appropriate nodes for Pods.

---

# 9. Requirement 6 — Different Workload Requirements

Suppose we have:

```text
GPU workload
CPU workload
Memory-heavy workload
Batch workload
Web workload
```

We may want certain workloads on certain nodes.

Kubernetes provides scheduling features including:

```text
Labels
Node selectors
Affinity
Anti-affinity
Taints
Tolerations
Resource requests/limits
```

Example:

```text
GPU Pod
   |
   v
GPU-capable Node
```

This is an important reason Kubernetes can be useful in complex platforms.

---

# 10. Requirement 7 — Service Discovery

Imagine:

```text
Order Service
      |
      v
Payment Service
```

The payment service may have several replicas:

```text
Payment Pod 1
Payment Pod 2
Payment Pod 3
```

Pods are replaceable and their IP addresses can change.

Applications should not hardcode:

```text
10.0.1.23
```

Kubernetes Services provide a stable way for workloads to discover and reach backend Pods.

Conceptually:

```text
Order Service
      |
      v
Kubernetes Service
      |
   +--+--+
   |  |  |
   v  v  v
 Pod Pod Pod
```

We will study Services deeply later.

---

# 11. Requirement 8 — Standard Platform for Many Teams

Imagine a company with:

```text
Team A -> Payments
Team B -> Orders
Team C -> Users
Team D -> Search
Team E -> Recommendations
```

Each team could invent its own deployment process.

That creates operational inconsistency.

Kubernetes can provide a common platform:

```text
                Kubernetes Platform
                        |
        +---------------+---------------+
        |               |               |
      Team A          Team B          Team C
        |               |               |
      Pods            Pods            Pods
```

Teams can use standardized:

```text
Deployment
Service
ConfigMap
Secret
Ingress
Resource limits
Health checks
```

---

# 12. Requirement 9 — Platform Engineering

Kubernetes is increasingly used as a platform underneath internal developer platforms.

Developers might submit:

```yaml
application: payments
replicas: 3
```

while the platform team handles:

```text
Networking
Security
Observability
Deployment
Scaling
Infrastructure
```

This creates an internal platform.

Kubernetes is therefore not just a "container launcher"; it can be the foundation of a broader platform engineering strategy.

---

# 13. When Kubernetes May Be Overkill

Consider this:

```text
Company
 |
 +-- One application
 +-- 100 users
 +-- One server
 +-- Two developers
```

Deploying:

```text
Kubernetes
Control plane
Worker nodes
Ingress
Storage
Monitoring
RBAC
Networking
CI/CD
```

may create more operational work than the application itself requires.

A simpler solution could be:

```text
VM + Docker
```

or:

```text
ECS/Fargate
```

depending on requirements.

---

# 14. Kubernetes Has a Learning and Operational Cost

Kubernetes requires knowledge of:

```text
Pods
Deployments
Services
Networking
Ingress
Storage
RBAC
ConfigMaps
Secrets
Scheduling
Health probes
Resource management
Observability
Security
```

Production Kubernetes also requires operational processes around:

```text
Upgrades
Backups
Monitoring
Logging
Security
Incident response
Capacity
Cost
```

Therefore:

> **Kubernetes has a total cost beyond the Kubernetes software itself.**

---

# 15. Kubernetes Cost

There are at least two categories of cost.

## Infrastructure Cost

Examples:

```text
Compute
Storage
Load balancers
Network traffic
Logging
Monitoring
```

## Engineering Cost

Examples:

```text
Training
Platform engineers
Cluster operations
Security
Troubleshooting
Upgrades
CI/CD integration
```

For a small application, engineering cost can dominate.

---

# 16. Managed Kubernetes Changes the Equation

Self-hosted:

```text
Your Team
   |
   +-- Control plane
   +-- Worker nodes
   +-- Networking
   +-- Upgrades
   +-- Security
```

Managed Kubernetes:

```text
Cloud Provider
      |
      +-- Managed control plane
      |
Your Team
      |
      +-- Workloads
      +-- Application configuration
      +-- Nodes/capacity depending on model
```

On AWS, the managed Kubernetes service is:

```text
Amazon EKS
```

Managed Kubernetes reduces some operational burden but does not eliminate Kubernetes expertise or all infrastructure costs.

---

# 17. Kubernetes on AWS

A common architecture is:

```text
                    AWS
                     |
                    EKS
                     |
        +------------+------------+
        |                         |
   Managed Control          Worker Capacity
       Plane                /           \
                          EC2         Fargate
                            |
                           Pods
```

EKS becomes especially useful when:

- The organization wants Kubernetes.
- AWS is the cloud provider.
- The team wants AWS integrations.
- The company wants AWS-managed Kubernetes control-plane infrastructure.

---

# 18. Kubernetes vs ECS Decision

A useful simplified decision process:

```text
                    Need containers?
                          |
                         Yes
                          |
              +-----------+-----------+
              |                       |
       Need Kubernetes?          No Kubernetes
              |                       |
             Yes                     ECS?
              |                       |
             EKS              AWS-centric? -> Yes
                                     |
                                    ECS
```

"Need Kubernetes?" might mean:

- Existing Kubernetes expertise.
- Kubernetes-specific tooling.
- Multi-cloud strategy.
- Kubernetes API/ecosystem requirements.
- Advanced extensibility requirements.

---

# 19. Kubernetes vs VM + Docker

## VM + Docker

```text
VM
 |
 +-- Docker
      |
      +-- Container
      +-- Container
```

Simple and easy to understand.

## Kubernetes

```text
Cluster
 |
 +-- Node
 |    +-- Pod
 |
 +-- Node
      +-- Pod
```

More capabilities, but more concepts.

The decision should be based on requirements.

---

# 20. Kubernetes Adoption Checklist

Ask these questions.

### Application

- How many services?
- How many replicas?
- Stateful or stateless?
- How often do we deploy?

### Traffic

- Is traffic predictable?
- Are there large traffic spikes?
- Do we need autoscaling?

### Availability

- What uptime is required?
- Can one server failure be tolerated?
- Do workloads need multiple availability zones?

### Team

- Does the team know Kubernetes?
- Is there a platform/DevOps team?
- Can someone operate the platform?

### Infrastructure

- Cloud or on-premises?
- AWS-only or multi-cloud?
- What AWS integrations are required?

### Cost

- What is the infrastructure budget?
- What is the engineering budget?

---

# 21. Decision Example 1 — Personal Blog

Requirements:

```text
1 application
100 visitors/day
No autoscaling
1 developer
Low availability requirements
```

Recommendation:

```text
VM + Docker
```

Kubernetes is probably unnecessary.

---

# 22. Decision Example 2 — E-Commerce Platform

Requirements:

```text
Frontend
Backend
Payments
Orders
Users
Search
Notifications
Redis
Workers

Multiple replicas
Traffic spikes
High availability
Frequent releases
```

Kubernetes may be a strong fit.

On AWS:

```text
EKS
```

could be considered.

---

# 23. Decision Example 3 — AWS Startup

Requirements:

```text
5 services
AWS only
Small DevOps team
Simple deployments
No Kubernetes expertise
```

Potential option:

```text
ECS/Fargate
```

may be simpler than EKS.

The correct answer depends on future requirements too.

---

# 24. Decision Example 4 — Multi-Cloud Enterprise

Requirements:

```text
AWS
Azure
On-premises
Hundreds of services
Existing Kubernetes teams
```

Kubernetes is likely attractive because a common Kubernetes-based platform can reduce differences between environments.

This does not mean multi-cloud Kubernetes is automatically simple—it can be operationally complex.

---

# 25. Mini Project — Choose the Right Platform

For each company, choose:

```text
A. VM + Docker
B. Docker Swarm
C. ECS
D. EKS/Kubernetes
```

and explain why.

### Company A

```text
One static website
50 users/day
One developer
```

### Company B

```text
AWS-only
10 containerized services
Small team
No Kubernetes experience
```

### Company C

```text
300 microservices
Multiple teams
Kubernetes expertise
Multi-cloud
```

### Company D

```text
Existing Docker Swarm cluster
Small internal application
Stable workload
```

### Company E

```text
Machine-learning platform
GPU workloads
Hundreds of workloads
Complex scheduling
Existing Kubernetes platform
```

Suggested starting answers:

```text
A -> VM + Docker
B -> ECS
C -> Kubernetes
D -> Swarm may remain reasonable
E -> Kubernetes
```

The important part is explaining the reasoning, not memorizing the answers.

---

# 26. Mini Project — Kubernetes Value Demonstration

Build the following progression.

## Stage 1

Run:

```text
1 NGINX container
```

## Stage 2

Run:

```text
3 NGINX containers
```

## Stage 3

Put them behind a load-balancing mechanism.

## Stage 4

Simulate one instance failing.

## Stage 5

Automate replacement.

## Stage 6

Scale replicas.

Then ask:

> "At which stage did orchestration become valuable?"

This exercise teaches students to understand the problem before learning the solution.

---

# 27. Interview Questions

## Q1. When should you use Kubernetes?

**Answer:**

Use Kubernetes when its capabilities—such as workload orchestration, scaling, service discovery, high-availability patterns, scheduling and a broad cloud-native ecosystem—provide enough value to justify its operational complexity.

## Q2. When should you avoid Kubernetes?

**Answer:**

Kubernetes may be unnecessary for simple applications with low traffic, few services, limited scaling requirements and teams without a need for Kubernetes-specific capabilities.

## Q3. Is Kubernetes suitable for every application?

**Answer:**

No. Kubernetes is a platform choice, not a universal requirement.

## Q4. Why is Kubernetes considered complex?

**Answer:**

It provides a large set of abstractions for networking, scheduling, storage, security, configuration, workload management and cluster operations.

## Q5. Does Kubernetes guarantee high availability?

**Answer:**

No. Kubernetes provides mechanisms that help build highly available systems, but high availability depends on cluster topology, workload replicas, application design, storage, networking and cloud infrastructure.

## Q6. Does Kubernetes automatically scale everything?

**Answer:**

No. Kubernetes supports autoscaling mechanisms, but they require appropriate configuration, metrics and resource definitions.

## Q7. What is the cost of Kubernetes?

**Answer:**

Kubernetes has infrastructure costs and engineering/operational costs. These include compute, storage, networking, monitoring, security, maintenance and personnel.

## Q8. Why use EKS instead of self-hosted Kubernetes?

**Answer:**

EKS can reduce the operational burden of managing Kubernetes control-plane infrastructure while providing a managed Kubernetes service integrated with AWS.

## Q9. Does managed Kubernetes mean zero operations?

**Answer:**

No. The provider manages important control-plane infrastructure, but teams still need to manage workloads, security, networking, observability, capacity and other parts of the platform depending on the deployment model.

## Q10. Kubernetes vs ECS for a small AWS startup?

**Answer:**

ECS may be preferable when simplicity and AWS-native integration are more important than Kubernetes portability or ecosystem requirements.

---

# 28. Interview Scenario

### Question

Your manager says:

> "Kubernetes is popular. Let's move every application to Kubernetes."

How should you respond?

### Strong Answer

Do not blindly accept the technology.

Ask:

```text
What problem are we solving?

What are the availability requirements?

How many services do we have?

Do we need autoscaling?

How often do we deploy?

Do we need multi-cloud portability?

Does the team have Kubernetes expertise?

What is the infrastructure and engineering cost?
```

Then compare:

```text
VM + Docker
ECS
EKS
Kubernetes
```

based on requirements.

This demonstrates architecture thinking rather than tool memorization.

---

# 29. Common Beginner Mistakes

## Mistake 1

> "Kubernetes is always better."

Incorrect.

Kubernetes is more capable in many areas, but capability comes with complexity.

## Mistake 2

> "If an application has containers, it needs Kubernetes."

Incorrect.

Containers can run on many platforms.

## Mistake 3

> "Managed Kubernetes means no DevOps work."

Incorrect.

Managed control plane != fully managed application platform.

## Mistake 4

> "Autoscaling means Kubernetes automatically understands business traffic."

Not necessarily.

Autoscaling depends on metrics and configuration.

## Mistake 5

> "High availability comes automatically from replicas."

Not necessarily.

You also need to consider:

```text
Node distribution
Availability zones
Pod disruption
Storage
Networking
Application architecture
```

---

# 30. Architecture Decision Framework

Use this framework in interviews and real projects:

```text
             Business Requirements
                     |
                     v
             Application Needs
                     |
                     v
          Availability / Scaling
                     |
                     v
            Team Capabilities
                     |
                     v
          Operational Complexity
                     |
                     v
                 Cost
                     |
                     v
             Platform Choice
```

Possible result:

```text
VM + Docker
Docker Compose
ECS
EKS
Kubernetes
```

---

# 31. Module Summary

Remember:

> **Kubernetes is a tool for solving operational problems at scale.**

Use it when its capabilities justify the complexity.

Strong reasons can include:

```text
Large number of workloads
Multiple replicas
High availability requirements
Frequent deployments
Autoscaling
Complex scheduling
Service discovery
Large engineering teams
Platform standardization
Multi-environment portability
```

Possible reasons not to use it:

```text
Very small application
Low traffic
Simple infrastructure
Tiny team
No Kubernetes-specific requirement
Operational cost is unjustified
```

The most important lesson:

> **Do not start with "Where can I use Kubernetes?" Start with "What problem do I need to solve?"**

---

# 32. Homework

## Theory

Answer:

1. When should an organization use Kubernetes?
2. When might Kubernetes be overkill?
3. What operational costs does Kubernetes introduce?
4. Does Kubernetes guarantee high availability?
5. Does Kubernetes automatically scale applications?
6. Why might ECS be preferable to EKS?
7. Why might EKS be preferable to ECS?
8. What questions should you ask before adopting Kubernetes?
9. What is the difference between infrastructure cost and engineering cost?
10. What does managed Kubernetes actually manage?

## Architecture Challenge

For each scenario, choose a platform and justify it:

### Scenario 1

```text
Personal blog
100 users/day
One developer
```

### Scenario 2

```text
AWS-only startup
20 services
Small DevOps team
No Kubernetes experience
```

### Scenario 3

```text
Enterprise
500 services
Multi-cloud
Existing Kubernetes team
```

### Scenario 4

```text
Internal tool
Two containers
Low traffic
No scaling
```

### Scenario 5

```text
Large ML platform
GPU workloads
Complex scheduling
Hundreds of workloads
```

For each, explain:

```text
Requirements
Risks
Operational complexity
Cost
Recommended platform
```

---

# Next Module

## Module 5 — Kubernetes Control Plane Components: Deep Dive

We will go deeper into:

- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
- Controller loops
- Reconciliation
- Authentication
- Authorization
- Admission
- How Kubernetes processes API requests
- What happens internally during `kubectl apply`
- Control-plane troubleshooting
- Interview questions
- Practical labs
