# Module 13 — Managed vs Self-Hosted Kubernetes

## Learning Objectives

By the end of this module, students should be able to:

- Explain managed Kubernetes and self-hosted Kubernetes.
- Compare control-plane and worker-node responsibilities.
- Understand Amazon EKS architecture.
- Understand EKS managed node groups, Fargate, and Auto Mode.
- Compare operational effort, security, networking, upgrades, and cost.
- Choose an appropriate Kubernetes deployment model for real-world scenarios.
- Deploy and inspect a basic EKS cluster.
- Answer common managed-vs-self-hosted Kubernetes interview questions.

---

# 1. What Problem Are We Solving?

An organization wants to run Kubernetes.

There are two broad choices:

### Self-hosted

You operate the Kubernetes infrastructure yourself:

```text
Control Plane
+
Worker Nodes
+
Networking
+
Storage
+
Upgrades
+
Monitoring
+
Security
```

### Managed Kubernetes

A cloud provider manages important Kubernetes infrastructure, especially the control plane:

```text
Cloud Provider
      |
      v
Managed Control Plane
      |
      v
Your Workloads
```

The main question is:

> How much Kubernetes infrastructure does your team want to operate?

---

# 2. What Is Self-Hosted Kubernetes?

Self-hosted Kubernetes means your organization is responsible for deploying and operating the Kubernetes cluster infrastructure.

A simplified architecture:

```text
                 Your Organization
                        |
        +---------------+---------------+
        |                               |
        v                               v
   Control Plane                    Worker Nodes
        |                               |
        +---- API Server                +---- kubelet
        +---- etcd                      +---- Runtime
        +---- Scheduler                 +---- kube-proxy
        +---- Controllers               +---- Pods
```

You may run it:

- on physical servers
- on virtual machines
- in your own data center
- on cloud VMs
- in isolated environments

Tools such as `kubeadm` can be used to build self-managed clusters.

---

# 3. What Is Managed Kubernetes?

Managed Kubernetes is a service where a provider manages important parts of the Kubernetes infrastructure.

For example:

```text
                  AWS
                   |
                   v
                Amazon EKS
                   |
          +--------+--------+
          |                 |
          v                 v
   Managed Control       Your Compute
      Plane             / Workloads
```

The provider operates the managed control plane, while your team still manages Kubernetes workloads and many configuration, security, networking, and application responsibilities.

Important:

> Managed Kubernetes reduces infrastructure operations; it does not eliminate Kubernetes administration.

---

# 4. Real-World Analogy

Imagine running a restaurant.

## Self-hosted

You maintain:

- building
- kitchen equipment
- electricity
- plumbing
- security
- staff
- food

You have maximum control and maximum responsibility.

## Managed

You use a professionally managed facility.

The facility infrastructure is maintained for you.

You still manage:

- menu
- employees
- customers
- food
- business operations

Managed Kubernetes is similar.

---

# 5. Kubernetes Responsibility Layers

Think about the stack:

```text
Application
    |
Kubernetes Workloads
    |
Kubernetes Cluster
    |
Control Plane
    |
Infrastructure
    |
Physical / Cloud Infrastructure
```

The more layers your team manages, the greater the operational responsibility.

---

# 6. Self-Hosted Kubernetes Architecture

A simplified cluster:

```text
                    Users
                      |
                      v
                API Server
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       etcd      Scheduler    Controllers
                      |
                      v
              Worker Nodes
          +-----------+-----------+
          |           |           |
          v           v           v
        Pod         Pod         Pod
```

Your team is responsible for keeping the control plane healthy.

That can include:

- installation
- certificates
- etcd
- backups
- upgrades
- high availability
- networking
- monitoring
- disaster recovery

---

# 7. Managed Kubernetes Architecture

With a managed service:

```text
                 Cloud Provider
                       |
               Managed Control Plane
                       |
               Kubernetes API
                       |
                Your VPC / Network
                       |
             +---------+---------+
             |                   |
             v                   v
        Worker Nodes          Serverless
             |                   |
             +---------+---------+
                       |
                       v
                      Pods
```

The provider manages important control-plane infrastructure.

Your team still manages the workloads and relevant cluster configuration.

---

# 8. Core Difference

The easiest way to remember:

```text
Self-hosted:
You operate the Kubernetes control plane.

Managed:
The provider operates the managed control plane.
```

This does not mean managed Kubernetes requires no DevOps or Kubernetes knowledge.

---

# 9. Responsibility Comparison

| Responsibility | Self-Hosted | Managed Kubernetes |
|---|---|---|
| Control plane installation | You | Provider |
| Control plane availability | You | Provider |
| etcd operation | You | Provider |
| API server operation | You | Provider |
| Scheduler | You | Provider |
| Controller components | You | Provider |
| Worker nodes | Usually you | Depends on compute model |
| Kubernetes workloads | You | You |
| Application configuration | You | You |
| RBAC | You | You |
| Application security | You | You |
| Network policies | You | You |
| Monitoring workloads | You | You |
| Cloud integration | You configure | Provider + your configuration |

The exact responsibility boundary varies by product and configuration.

---

# 10. Why Use Managed Kubernetes?

Managed Kubernetes can provide:

### Less infrastructure management

You don't have to manually build and maintain the entire control plane.

### High availability options

The provider operates the managed control-plane infrastructure according to the service architecture.

### Cloud integration

Managed Kubernetes can integrate with:

- IAM
- VPC networking
- load balancers
- storage
- container registries
- monitoring

### Easier upgrades

The provider provides supported mechanisms for control-plane upgrades.

### Faster setup

You can create a cluster without manually configuring every control-plane component.

---

# 11. Disadvantages of Managed Kubernetes

Potential disadvantages include:

- cloud/provider dependency
- provider-specific features
- ongoing service and infrastructure costs
- less control over managed components
- provider-specific networking and IAM concepts
- still requires Kubernetes expertise

Remember:

> Managed Kubernetes reduces operational burden; it does not eliminate Kubernetes complexity.

---

# 12. Why Use Self-Hosted Kubernetes?

Self-hosting can be useful when you need:

- maximum control
- specialized infrastructure
- on-premises deployment
- disconnected or air-gapped environments
- custom control-plane configuration
- unusual networking requirements
- a mature platform engineering team

---

# 13. Disadvantages of Self-Hosted Kubernetes

Self-hosted Kubernetes can require substantial operational effort.

Your team may need to handle:

- control-plane installation
- etcd
- certificates
- backups
- upgrades
- high availability
- networking
- security
- monitoring
- disaster recovery
- node replacement
- troubleshooting

A production self-hosted cluster can become a major platform-engineering responsibility.

---

# 14. Control Plane Responsibility

## Self-hosted

```text
Your Team
   |
   +---- API Server
   +---- etcd
   +---- Scheduler
   +---- Controllers
   +---- Certificates
   +---- Backups
   +---- Upgrades
```

## Managed

```text
Cloud Provider
   |
   +---- Managed API Server
   +---- Managed etcd
   +---- Managed Scheduler
   +---- Managed Controllers
```

You interact with the managed control plane through the Kubernetes API.

---

# 15. Worker Node Responsibility

Managed control plane does not automatically mean managed worker nodes.

Possible models include:

```text
Managed Kubernetes
 |
 +---- Managed Node Groups
 |
 +---- Self-Managed Nodes
 |
 +---- Serverless / Fargate
 |
 +---- More automated compute modes
```

Always ask:

> Which parts of the cluster are actually managed?

---

# 16. Amazon EKS

Amazon Elastic Kubernetes Service (EKS) is AWS's managed Kubernetes service.

Simplified architecture:

```text
                     AWS
                      |
                  Amazon EKS
                      |
          +-----------+-----------+
          |                       |
          v                       v
   Managed Control Plane        Compute
                                  |
                    +-------------+-------------+
                    |             |             |
                    v             v             v
              Managed Nodes   Self-Managed   Fargate
                                 Nodes
```

EKS lets organizations run Kubernetes while AWS manages important control-plane infrastructure.

---

# 17. EKS Control Plane

Conceptually, the EKS control plane includes the Kubernetes components needed to operate the cluster:

```text
EKS Control Plane
       |
       +---- Kubernetes API Server
       +---- etcd
       +---- Scheduler
       +---- Controller components
```

You interact with the cluster using:

```bash
kubectl
```

---

# 18. EKS Worker Nodes

EC2 worker nodes execute application Pods.

A simplified node contains:

```text
EC2 Instance
    |
    +---- kubelet
    +---- container runtime
    +---- networking components
    +---- Pods
```

Example:

```text
EC2 Node
   |
   +---- frontend Pod
   +---- backend Pod
   +---- monitoring Pod
```

---

# 19. EKS Managed Node Groups

Managed Node Groups provide AWS-managed lifecycle operations for groups of EC2 worker nodes.

Conceptually:

```text
EKS
 |
 +---- Managed Node Group
          |
          +---- EC2
          +---- EC2
          +---- EC2
```

You still configure important properties such as:

- instance types
- scaling configuration
- subnets
- labels
- taints
- IAM configuration

AWS handles supported node lifecycle operations according to the service configuration.

---

# 20. Self-Managed EKS Nodes

You can also run EC2 worker nodes with more direct management responsibility.

```text
EKS Control Plane
        |
        v
Self-Managed EC2 Nodes
        |
        +---- Pods
```

Your team has more responsibility for:

- instance lifecycle
- AMI management
- upgrades
- scaling
- node configuration
- replacement

This gives more control but more operational work.

---

# 21. EKS with Fargate

AWS Fargate can run supported Kubernetes Pods without requiring you to manage EC2 worker nodes for those Pods.

Conceptually:

```text
EKS Control Plane
       |
       v
     Fargate
       |
       +---- Pod
       +---- Pod
```

Fargate can reduce node-management responsibility.

However, it has architectural constraints and is not a drop-in replacement for every Kubernetes workload.

---

# 22. EKS Auto Mode

EKS Auto Mode is designed to automate more of the underlying compute and cluster infrastructure lifecycle.

Conceptually:

```text
You define workloads
       |
       v
EKS Auto Mode
       |
       +---- Compute provisioning
       +---- Scaling
       +---- Infrastructure lifecycle
```

The important distinction is:

```text
Managed Kubernetes
    =
Managed control plane

More automated Kubernetes
    =
More infrastructure lifecycle handled by the platform
```

Always verify current AWS documentation for the exact features and supported configurations of the EKS mode/version you use.

---

# 23. EKS Architecture Example

A production-style architecture can look like:

```text
                         Internet
                            |
                            v
                    AWS Load Balancer
                            |
                            v
                       EKS Service
                            |
                  +---------+---------+
                  |                   |
                  v                   v
             Frontend Pods       Frontend Pods
                  |
                  v
             Backend Service
                  |
          +-------+-------+
          |       |       |
          v       v       v
       Backend Backend Backend
         Pod      Pod      Pod
                  |
                  v
                  RDS
```

Possible AWS services:

```text
Route 53
   |
Load Balancer
   |
EKS
   |
ECR
   |
RDS
   |
CloudWatch
```

---

# 24. Managed Kubernetes Does Not Manage Your Application

If you use EKS, AWS does not automatically fix:

```text
Application bugs
Bad Dockerfiles
Incorrect Services
Broken Deployments
Bad RBAC
Bad Secrets
Incorrect resource limits
Poor application architecture
```

Your team still owns the application and Kubernetes configuration.

---

# 25. Security Responsibility

Security should be considered in layers:

```text
AWS infrastructure security
        |
        v
EKS control-plane security
        |
        v
VPC/network security
        |
        v
Kubernetes RBAC
        |
        v
Pod security
        |
        v
Application security
```

Your team still needs to configure appropriate:

- IAM permissions
- Kubernetes RBAC
- security groups
- network policies
- Secrets
- image security
- container security
- Pod security
- application authentication

---

# 26. Networking Responsibility

In self-hosted Kubernetes, your team may need to configure:

```text
CNI
Pod networking
Service networking
Load balancing
Ingress
DNS
Network policies
Routing
```

In EKS, AWS provides integrations and supported networking components, but your team still needs to configure VPCs, subnets, security controls, and Kubernetes networking appropriately.

---

# 27. Storage Responsibility

Managed control planes do not automatically design your application storage architecture.

Depending on the workload, AWS services can include:

```text
EBS
EFS
S3
RDS
```

For example:

```text
Application Pod
      |
      v
PersistentVolumeClaim
      |
      v
EBS/EFS
```

For a database:

```text
Application
      |
      v
RDS
```

A managed database can often be preferable to running a production database inside Kubernetes when appropriate.

---

# 28. Upgrades

## Self-hosted

Your team may need to coordinate:

```text
Control plane
   ↓
Cluster components
   ↓
Worker nodes
   ↓
CNI
   ↓
CSI
   ↓
Ingress
   ↓
Applications
```

## Managed

The provider manages supported control-plane infrastructure and provides upgrade mechanisms.

You still need to plan:

- Kubernetes API compatibility
- node upgrades
- application compatibility
- add-on compatibility
- workload disruption
- testing

Managed does not mean:

> Upgrade everything automatically without planning.

---

# 29. Cost Considerations

Do not simplify the decision to:

```text
Managed = expensive
Self-hosted = cheap
```

The real calculation is:

```text
Infrastructure cost
+
Engineering time
+
Operations
+
Monitoring
+
Backups
+
Security
+
Downtime risk
+
Upgrade effort
```

A self-hosted cluster may have lower direct service fees but much higher engineering effort.

A managed cluster may cost more directly but reduce operational overhead.

---

# 30. Total Cost of Ownership

### Self-hosted

```text
Cloud/physical infrastructure
+
Control plane
+
Worker nodes
+
Engineers
+
Monitoring
+
Backups
+
Security
+
Maintenance
```

### Managed

```text
Managed Control Plane
+
Worker Nodes / Compute
+
Engineers
+
Monitoring
+
Security
```

The correct comparison is total cost of ownership, not just the infrastructure invoice.

---

# 31. When Should You Choose Managed Kubernetes?

Managed Kubernetes is often a strong choice when:

- you run primarily in a major cloud
- your team is small
- you want faster delivery
- you don't want to maintain the control plane
- you need cloud integrations
- you need production Kubernetes without building a large platform team

Example:

> A startup has 5 engineers and wants Kubernetes on AWS.

EKS can be a sensible choice because the team can focus more on workloads than control-plane operations.

---

# 32. When Should You Choose Self-Hosted Kubernetes?

Self-hosting may make sense when:

- you require on-premises Kubernetes
- you need air-gapped environments
- you need unusual infrastructure
- you require deep control
- you already have a mature platform team
- regulatory or infrastructure requirements dictate the architecture

Example:

> A company operates Kubernetes in an isolated data center with no direct internet access.

Self-hosted Kubernetes may be appropriate.

---

# 33. Decision Matrix

| Requirement | Managed Kubernetes | Self-Hosted |
|---|---:|---:|
| Fast setup | Excellent | Poor |
| Control-plane control | Lower | High |
| Operational burden | Lower | High |
| Cloud integration | Excellent | Depends |
| On-premises | Depends | Excellent |
| Air-gapped environments | Depends | Strong |
| Learning Kubernetes internals | Good | Excellent |
| Small team | Strong choice | Often difficult |
| Large platform team | Strong choice | Possible |
| Custom infrastructure | Limited | Excellent |

---

# 34. Scenario Exercise — Small Startup

### Scenario

A startup has:

- 6 developers
- AWS as its cloud provider
- 15 microservices
- no dedicated platform team
- plans to scale quickly

### Answer

Managed Kubernetes such as EKS is generally a strong option.

Reason:

```text
Small platform team
       +
AWS environment
       +
Many services
       ↓
Managed Kubernetes
```

The team can spend less effort maintaining the control plane.

---

# 35. Scenario Exercise — On-Premises

### Scenario

A company has:

- its own data center
- no public cloud
- strict network isolation
- a large infrastructure team

### Answer

Self-hosted Kubernetes may be more appropriate because:

- the environment is on-premises
- there may be no managed cloud control plane
- the organization has infrastructure expertise
- custom control is important

---

# 36. Scenario Exercise — Learning

### Scenario

A student wants to understand:

- kube-apiserver
- etcd
- scheduler
- controller manager
- certificates
- cluster bootstrap

### Answer

A self-managed local Kubernetes environment can be useful for learning these internals.

Examples:

```text
kubeadm
kind
```

For production, managed Kubernetes can reduce operational burden.

---

# 37. Practical Lab 1 — Compare Local and Managed Kubernetes

Create a local cluster:

```bash
kind create cluster --name training
```

Check:

```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

Ask:

> Which control-plane responsibilities are you operating in this local environment?

Then compare:

```text
Local cluster:
You operate the cluster infrastructure.

EKS:
AWS operates the managed control plane.
```

---

# 38. Practical Lab 2 — Create an EKS Cluster

One common training approach is `eksctl`.

Configure AWS credentials:

```bash
aws configure
```

Verify:

```bash
aws sts get-caller-identity
```

Example:

```bash
eksctl create cluster   --name training-cluster   --region ap-south-1   --nodes 2   --node-type t3.medium
```

The exact options and defaults can change. Verify current AWS and eksctl documentation before using commands in a production account.

---

# 39. Connect kubectl to EKS

After the cluster exists:

```bash
aws eks update-kubeconfig   --region ap-south-1   --name training-cluster
```

Check:

```bash
kubectl get nodes
```

Then:

```bash
kubectl cluster-info
```

---

# 40. Inspect the EKS Cluster

Run:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Look for components related to:

- CoreDNS
- networking
- kube-proxy
- storage
- AWS integrations

Ask students:

> Which components are running on the worker nodes and which control-plane components are operated by AWS?

---

# 41. Deploy a Sample Application

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
        - name: web
          image: nginx
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f web.yaml
```

Check:

```bash
kubectl get pods
kubectl get deployment
```

Expose it:

```bash
kubectl expose deployment web   --type=LoadBalancer   --port=80
```

Check:

```bash
kubectl get svc web
```

On AWS, the external address can take some time to become available.

---

# 42. EKS Mini Project — Production-Style Learning Architecture

Build:

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
              Backend Service
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Backend   Backend   Backend
          Pods      Pods      Pods
                    |
                    v
                  RDS
```

Possible AWS services:

```text
ECR
EKS
EC2
ALB/NLB
RDS
CloudWatch
IAM
VPC
```

Students should explain what each service contributes.

---

# 43. Mini Project Tasks

## Task 1 — Containerize

Create:

```text
frontend/Dockerfile
backend/Dockerfile
```

## Task 2 — Push Images

Build:

```bash
docker build -t frontend .
docker build -t backend .
```

Push the images to Amazon ECR.

## Task 3 — Deploy to EKS

Create:

```text
frontend-deployment.yaml
frontend-service.yaml
backend-deployment.yaml
backend-service.yaml
```

## Task 4 — Scale

```bash
kubectl scale deployment backend --replicas=5
```

## Task 5 — Self-Healing

```bash
kubectl delete pod <backend-pod>
```

Observe:

```bash
kubectl get pods -w
```

## Task 6 — Responsibility Exercise

Write:

```text
AWS manages:
...

Our team manages:
...
```

This is the most important learning objective of the project.

---

# 44. Common Beginner Misconceptions

## Misconception 1

> EKS means AWS manages everything.

False.

EKS manages important control-plane infrastructure, but your team still manages workloads and many cluster, security, networking, and application concerns.

## Misconception 2

> Managed Kubernetes means no DevOps work.

False.

You still need:

- CI/CD
- observability
- security
- resource management
- Kubernetes configuration
- application deployment
- troubleshooting

## Misconception 3

> Self-hosted Kubernetes is always cheaper.

Not necessarily.

Engineering and operational costs can dominate total cost.

## Misconception 4

> Managed Kubernetes gives no control.

False.

You still control many Kubernetes resources and workload configurations.

## Misconception 5

> EKS and Kubernetes are different technologies.

EKS is AWS's managed service for running Kubernetes.

```text
Kubernetes = platform
EKS = AWS managed Kubernetes service
```

---

# 45. Interview Questions and Answers

## Q1. What is managed Kubernetes?

Managed Kubernetes is a Kubernetes service where the provider manages important Kubernetes infrastructure, especially the control plane.

## Q2. What is self-hosted Kubernetes?

Self-hosted Kubernetes is a cluster where the organization is responsible for operating Kubernetes infrastructure, including the control plane.

## Q3. What is the biggest difference?

The main difference is operational responsibility for the control plane and associated infrastructure.

## Q4. Does managed Kubernetes eliminate Kubernetes administration?

No. It reduces some infrastructure-management responsibilities, but teams still manage workloads, configuration, security, networking, observability, and other concerns.

## Q5. What is Amazon EKS?

Amazon Elastic Kubernetes Service is AWS's managed Kubernetes service.

## Q6. Who manages the EKS control plane?

AWS manages the EKS managed control plane.

## Q7. Who manages Pods in EKS?

Your workloads run on the compute infrastructure selected for the cluster, such as EC2 worker nodes or supported serverless compute options. Your team remains responsible for workload definitions.

## Q8. What are EKS Managed Node Groups?

They provide AWS-managed lifecycle operations for groups of EC2 worker nodes used by EKS.

## Q9. What is EKS Fargate?

Fargate is a serverless compute option that can run supported Kubernetes Pods without requiring you to manage EC2 worker nodes for those Pods.

## Q10. What is EKS Auto Mode?

EKS Auto Mode automates more of the underlying compute and cluster infrastructure lifecycle compared with a traditional EKS setup.

## Q11. Why choose managed Kubernetes?

Common reasons:

- reduced control-plane operations
- faster setup
- cloud integration
- simpler operational model
- smaller platform team

## Q12. Why choose self-hosted Kubernetes?

Common reasons:

- maximum control
- on-premises requirements
- air-gapped environments
- specialized infrastructure
- mature platform team

## Q13. Is Kubernetes knowledge required when using EKS?

Absolutely.

EKS is Kubernetes. You still need to understand:

```text
Pods
Deployments
Services
Networking
RBAC
Storage
ConfigMaps
Secrets
Ingress
Scheduling
Observability
```

## Q14. What is the difference between EKS and ECS?

EKS runs Kubernetes.

ECS is AWS's own container orchestration service.

```text
EKS → Kubernetes
ECS → AWS-native container orchestration
```

## Q15. Does EKS manage worker nodes automatically?

It depends on the compute model.

Possible options include:

```text
Managed Node Groups
Self-Managed Nodes
Fargate
EKS Auto Mode
```

Each has a different responsibility model.

---

# 46. Interview Scenarios

## Scenario 1 — Small Startup

### Question

A startup has 5 developers and no platform engineer. They want Kubernetes on AWS. Would you recommend self-hosted Kubernetes?

### Answer

Usually no.

A managed service such as EKS is often more practical because it reduces control-plane operational work.

---

## Scenario 2 — On-Premises

### Question

A company has an air-gapped data center and cannot use AWS or another managed cloud control plane. What would you choose?

### Answer

A self-hosted Kubernetes architecture may be appropriate because the organization needs to operate Kubernetes inside its own infrastructure.

---

## Scenario 3 — Control Plane Failure

### Question

In self-hosted Kubernetes, who is responsible for recovering a failed control-plane component?

### Answer

The organization operating the cluster.

In managed Kubernetes, the provider manages the provider-operated control plane according to the service architecture and availability model.

---

## Scenario 4 — Worker Node Failure

### Question

Does a managed control plane mean AWS automatically fixes every worker-node problem?

### Answer

No.

Worker-node responsibility depends on the compute model.

```text
Managed Node Groups → AWS handles supported node lifecycle operations
Self-managed nodes → More responsibility for your team
Fargate → No EC2 worker node management for those Pods
```

---

# 47. Troubleshooting Checklist

When an EKS application is unavailable:

```text
1. Is the cluster reachable?
       ↓
2. Are nodes Ready?
       ↓
3. Are Pods Running?
       ↓
4. Are Pods Ready?
       ↓
5. Does the Service have endpoints?
       ↓
6. Is DNS working?
       ↓
7. Is the application listening?
       ↓
8. Is the LoadBalancer healthy?
       ↓
9. Are security groups/network policies correct?
       ↓
10. Are IAM permissions correct?
       ↓
11. Are AWS dependencies healthy?
```

---

# 48. Managed vs Self-Hosted Decision Framework

Ask:

### Do we need on-premises Kubernetes?

```text
Yes → Self-hosted may be necessary
No  → Managed is an option
```

### Do we have a strong platform team?

```text
Yes → Either can work
No  → Managed is often preferable
```

### Do we need maximum control?

```text
Yes → Self-hosted may be preferable
No  → Managed may be simpler
```

### Are we heavily invested in AWS?

```text
Yes → EKS becomes attractive
```

### Is reducing operational burden important?

```text
Yes → Managed Kubernetes
```

---

# 49. Practical Assignment

Create a comparison document for:

```text
Kubernetes
EKS
Self-hosted Kubernetes
ECS
```

Compare:

- architecture
- control plane
- worker nodes
- networking
- scaling
- security
- upgrades
- cost
- operational responsibility
- ideal use cases

Then choose a platform for:

### Scenario A

Small AWS startup.

### Scenario B

Large on-premises enterprise.

### Scenario C

Air-gapped environment.

### Scenario D

Five-person engineering team building 20 microservices.

### Scenario E

Enterprise with a dedicated platform engineering team.

Explain your decision for each.

---

# 50. Quick Revision Cheat Sheet

## Self-Hosted

```text
You
 |
 +---- Control Plane
 +---- etcd
 +---- Scheduler
 +---- Controllers
 +---- Worker Nodes
 +---- Networking
 +---- Upgrades
 +---- Monitoring
 +---- Backups
```

## Managed Kubernetes

```text
Cloud Provider
 |
 +---- Managed Control Plane

You
 |
 +---- Workloads
 +---- Kubernetes configuration
 +---- Security configuration
 +---- Application
 +---- Compute configuration depending on model
```

## EKS

```text
AWS
 |
 +---- EKS Control Plane
 |
 +---- Managed Node Groups
 |
 +---- Self-Managed Nodes
 |
 +---- Fargate
 |
 +---- Auto Mode
```

---

# 51. Key Takeaways

1. Self-hosted Kubernetes gives more control but requires more operational work.
2. Managed Kubernetes reduces control-plane management responsibilities.
3. Managed does not mean fully managed application operations.
4. Amazon EKS is AWS's managed Kubernetes service.
5. EKS separates the managed control plane from your workload compute.
6. Worker-node responsibility depends on the compute model.
7. Managed Node Groups reduce EC2 node lifecycle management.
8. Fargate can run supported Pods without you managing EC2 worker nodes for those Pods.
9. EKS Auto Mode automates more of the underlying infrastructure lifecycle.
10. Security remains a shared responsibility.
11. Networking and storage still require architecture decisions.
12. Managed Kubernetes can be a strong choice for small teams running on AWS.
13. Self-hosted Kubernetes can be appropriate for on-premises, air-gapped, or highly customized environments.
14. Total cost includes engineering and operational effort, not just infrastructure bills.
15. Always evaluate the responsibility boundary before choosing a Kubernetes platform.

---

# 52. Final Mental Model

The easiest way to remember this module:

```text
              KUBERNETES
                   |
       +-----------+-----------+
       |                       |
       v                       v
  Self-Hosted              Managed
       |                       |
       v                       v
   You operate            Provider operates
   control plane          managed control plane
       |                       |
       +-----------+-----------+
                   |
                   v
              Workloads
                   |
                   v
             Pods/Services
```

For AWS:

```text
                  Amazon EKS
                      |
          +-----------+-----------+
          |                       |
          v                       v
   Managed Control Plane      Compute
                                  |
                  +---------------+---------------+
                  |               |               |
                  v               v               v
             Managed Nodes   Self-Managed     Fargate
                                  Nodes
```

The central lesson is:

> Choose Kubernetes infrastructure based on the amount of operational responsibility your team is prepared to own.

---

# Next Module

## Module 14 — AWS EKS: Benefits and Features

Topics:

- Why use EKS?
- EKS architecture
- EKS control plane
- EKS cluster creation
- Managed Node Groups
- EKS networking
- IAM integration
- ECR integration
- AWS Load Balancers
- EBS/EFS storage
- CloudWatch integration
- Autoscaling
- Security features
- EKS vs self-managed Kubernetes
- Practical EKS deployment project
- Interview questions
