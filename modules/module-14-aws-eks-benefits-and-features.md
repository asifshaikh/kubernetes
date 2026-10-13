# Module 14 — AWS EKS: Benefits and Features

## Learning Objectives

By the end of this module, students should be able to:

- Explain Amazon Elastic Kubernetes Service (EKS).
- Describe EKS architecture and AWS/customer responsibilities.
- Understand EKS compute options: Managed Node Groups, self-managed nodes, Fargate, and Auto Mode.
- Explain EKS integration with IAM, VPC, ECR, load balancers, storage, and CloudWatch.
- Understand EKS networking, security, and autoscaling concepts.
- Deploy a sample application to EKS.
- Troubleshoot common EKS problems.
- Complete a mini-project and answer interview questions.

---

# 1. What Is Amazon EKS?

**Amazon Elastic Kubernetes Service (EKS)** is AWS's managed service for running Kubernetes.

Kubernetes provides container orchestration. EKS provides an AWS-managed way to run a Kubernetes control plane and connect it to AWS compute, networking, identity, storage, and monitoring services.

```text
AWS
 |
 +---- Amazon EKS
          |
          +---- Managed Kubernetes control plane
          |
          +---- Workload compute
          |       +---- EC2 nodes
          |       +---- Fargate for supported Pods
          |       +---- Other supported compute options
          |
          +---- AWS integrations
                  +---- VPC
                  +---- IAM
                  +---- ECR
                  +---- Load Balancing
                  +---- EBS / EFS
                  +---- CloudWatch
```

EKS is not a replacement for Kubernetes knowledge. It is a managed way to run Kubernetes on AWS.

---

# 2. Why Use EKS?

Imagine an organization has 20 containerized microservices. It needs to deploy versions, restart failed workloads, scale replicas, route traffic, connect services, manage configuration, monitor health, and apply security controls.

EKS provides a Kubernetes control plane operated by AWS, while the organization uses Kubernetes resources to deploy and manage applications.

Common benefits:

1. **Managed control plane** — AWS operates the EKS control-plane infrastructure.
2. **Kubernetes APIs and tools** — use familiar resources and tools such as `kubectl`.
3. **AWS integration** — connect with AWS identity, networking, registries, load balancers, storage, and monitoring.
4. **Compute choices** — select appropriate node and compute models.
5. **Automation options** — managed node groups and other EKS capabilities reduce parts of infrastructure lifecycle work.
6. **Kubernetes ecosystem** — use compatible community tools and projects.

These benefits do not mean AWS manages every part of your application or cluster configuration.

---

# 3. EKS Architecture

```text
                         AWS Region
                             |
                             v
                      Amazon EKS Cluster
                             |
                +------------+------------+
                |                         |
                v                         v
        Managed Control Plane       Workload Compute
                |                         |
          +-----+------+          +-------+-------+
          |     |      |          |       |       |
          v     v      v          v       v       v
        API   etcd  Controllers   EC2   Fargate   Other
       Server       / Scheduler  Nodes  (supported compute)
                                      |
                                      v
                                  Application Pods
```

**Control plane:** accepts Kubernetes API requests and coordinates desired state.

**Worker compute:** runs application Pods.

**AWS integrations:** provide image storage, networking, external load balancing, persistent storage, identity, and observability.

---

# 4. What Does AWS Manage?

For an EKS managed control plane, AWS operates the underlying control-plane infrastructure, including components such as the Kubernetes API server and etcd.

Your team remains responsible for designing and operating workloads and configuring many aspects of the cluster.

| Area | Typical responsibility |
|---|---|
| EKS managed control plane | AWS operates managed infrastructure |
| Application code | Customer |
| Docker images | Customer builds and publishes them |
| Deployments and Services | Customer |
| Workload resource requests/limits | Customer |
| Kubernetes access and RBAC design | Customer configures |
| VPC, subnet, and security design | Customer configures within AWS capabilities |
| Worker compute | Depends on selected model |
| Application monitoring and alerts | Customer configures |
| Application secrets and permissions | Customer configures securely |

Exact responsibilities depend on selected EKS features and architecture.

**Key lesson:** managed control plane does not mean fully managed application operations.

---

# 5. EKS Compute Options

## A. Managed Node Groups

Managed Node Groups use EC2 instances as worker nodes, with AWS handling supported node-group lifecycle operations.

```text
EKS
 |
 +---- Managed Node Group
          |
          +---- EC2 node
          +---- EC2 node
          +---- EC2 node
                  |
                  +---- Pods
```

You still configure properties such as instance types, scaling settings, subnets, labels, taints, and other supported options.

## B. Self-Managed Nodes

You manage more of the EC2 node lifecycle directly, including node configuration, updates, and replacement.

## C. Fargate

Fargate can run supported Pods without requiring you to manage EC2 worker nodes for those Pods. It has workload and configuration constraints and is not suitable for every workload.

## D. EKS Auto Mode

EKS Auto Mode automates additional infrastructure lifecycle tasks. Its supported capabilities and limitations depend on the current EKS offering and configuration.

Do not assume every compute option has the same feature set, cost model, or operational responsibility.

---

# 6. AWS Services Commonly Used with EKS

- **Amazon VPC** — network, subnets, routes, and connectivity.
- **IAM** — AWS identity and permissions.
- **Amazon ECR** — private container image registry.
- **Elastic Load Balancing** — internal or external traffic entry points.
- **Amazon EBS / EFS** — persistent block or shared file storage, where appropriate.
- **Amazon CloudWatch** — logs, metrics, dashboards, and alarms.
- **Route 53** — DNS for application endpoints when configured.
- **RDS** — managed relational database outside Kubernetes.

These services integrate through configuration, controllers, add-ons, and IAM permissions. Creating an EKS cluster does not automatically configure every integration for every application.

---

# 7. EKS and IAM

IAM is AWS's identity and access management service. Distinguish between:

1. **AWS identity and permissions** — who can perform AWS API operations.
2. **Kubernetes authentication and authorization** — who can access Kubernetes resources and what they can do.
3. **Pod access to AWS services** — how an application obtains narrowly scoped AWS permissions.

```text
Developer / CI pipeline
       |
       v
AWS identity
       |
       v
EKS cluster access configuration
       |
       v
Kubernetes RBAC authorization
       |
       v
Kubernetes API resources
```

For workloads accessing AWS APIs, use an appropriate workload identity mechanism supported by EKS, such as EKS Pod Identity or IAM Roles for Service Accounts (IRSA), according to your cluster configuration. Avoid placing long-lived AWS keys in container images or source code.

---

# 8. EKS and Amazon ECR

Amazon Elastic Container Registry (ECR) stores container images.

```text
Source Code
    |
    v
Docker Build
    |
    v
Container Image
    |
    v
Amazon ECR
    |
    v
EKS Deployment
    |
    v
Pods pull and run the image
```

Example login command:

```bash
aws ecr get-login-password --region <region> \
  | docker login \
      --username AWS \
      --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
```

Build and tag:

```bash
docker build -t web-app .
docker tag web-app:latest \
  <account-id>.dkr.ecr.<region>.amazonaws.com/web-app:latest
```

Push:

```bash
docker push \
  <account-id>.dkr.ecr.<region>.amazonaws.com/web-app:latest
```

Use meaningful version tags for repeatable deployments rather than relying only on `latest`.

---

# 9. EKS Networking Basics

```text
VPC
 |
 +---- Subnets
        |
        +---- Worker nodes / Pod networking
        |
        +---- Load balancer subnets
```

Important concepts:

- **VPC** — logical network in AWS.
- **Subnets** — IP ranges associated with Availability Zones.
- **Security groups** — control network traffic at supported AWS network interfaces.
- **Route tables** — determine network routing.
- **CNI** — provides Pod networking.
- **Kubernetes Service** — stable access to a group of Pods.
- **Ingress / load-balancer controller** — may provide application-level routing.

EKS commonly uses the Amazon VPC CNI, but cluster networking can vary by supported configuration.

A Pod being `Running` does not prove it is reachable from the internet. Routing, Services, security groups, health checks, and application configuration also matter.

---

# 10. Public and Private Subnets

A common design separates entry points from application compute.

```text
                         Internet
                            |
                            v
                  Public Load Balancer
                            |
                 +----------+----------+
                 |                     |
                 v                     v
           Private Subnet A      Private Subnet B
                 |                     |
                 v                     v
             Worker Node           Worker Node
                 |                     |
                 v                     v
               Pods                  Pods
```

This is a conceptual pattern, not a universal requirement. Subnet placement depends on the architecture, networking mode, and access needs.

A common security goal is to expose only required entry points and keep internal workloads private where practical.

---

# 11. EKS and Load Balancing

EKS can integrate with AWS load-balancing services.

- **Application Load Balancer (ALB)** — commonly used for HTTP/HTTPS routing.
- **Network Load Balancer (NLB)** — commonly used for network-layer traffic and supported Service exposure patterns.

```text
Client
  |
  v
AWS Load Balancer
  |
  v
Kubernetes Service / Ingress routing
  |
  v
Application Pods
```

An AWS Load Balancer Controller can manage supported load-balancer resources from Kubernetes configuration. The setup depends on controller, annotations, networking mode, and cluster configuration.

A Kubernetes `Service` of type `LoadBalancer` and an `Ingress` resource are different abstractions. Ingress requires an Ingress controller to implement its rules.

---

# 12. EKS and Persistent Storage

Containers are replaceable. Data that must survive Pod replacement should not live only in the container's writable layer.

Kubernetes storage abstractions include:

- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- StorageClass
- Container Storage Interface (CSI) drivers

AWS storage options can include:

- **EBS** — block storage, often appropriate for a volume attached to a particular node at a time according to driver and volume capabilities.
- **EFS** — shared file storage where its access and performance model fits.
- **S3** — object storage accessed through APIs, not a direct replacement for a normal filesystem PVC.
- **RDS** — managed relational database outside Kubernetes.

```text
Application Pod
      |
      v
PersistentVolumeClaim
      |
      v
StorageClass / CSI driver
      |
      v
AWS storage resource
```

Choose storage based on durability, access mode, performance, backup, and recovery requirements.

---

# 13. EKS Observability with CloudWatch

Observability helps answer:

- Is the application healthy?
- Are Pods restarting?
- Are nodes running out of resources?
- Are requests failing?
- Is latency increasing?
- Are AWS dependencies failing?

CloudWatch can provide logs, metrics, dashboards, and alarms when collection and permissions are configured.

Useful commands:

```bash
kubectl get pods -A
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl top nodes
kubectl top pods
```

Metrics commands require the relevant metrics components to be installed and working.

A useful monitoring plan includes application logs, Pod and node metrics, load-balancer health, database metrics, and actionable alerts.

---

# 14. EKS Autoscaling Concepts

Autoscaling mechanisms solve different problems.

| Mechanism | What it changes |
|---|---|
| Horizontal Pod Autoscaler (HPA) | Number of Pod replicas |
| Vertical Pod Autoscaler (VPA) | Resource recommendations/adjustments, depending on configuration |
| Cluster Autoscaler | Node group capacity in supported setups |
| Karpenter | Provisions and consolidates nodes based on workload needs, when configured |
| EKS Auto Mode | Automates additional infrastructure operations within its supported scope |

Example:

```text
CPU usage increases
       |
       v
HPA may increase Pod replicas
       |
       v
Cluster needs more capacity
       |
       v
A node-scaling mechanism may add compute
```

HPA does not automatically create more nodes. Pod scaling and node scaling are separate concerns.

Autoscaling requires suitable metrics, permissions, resource requests, and compatible configuration.

---

# 15. EKS Security Checklist

Before deploying a real workload:

- Use least-privilege IAM permissions.
- Restrict Kubernetes API access appropriately.
- Configure Kubernetes RBAC.
- Avoid embedding credentials in images or Git repositories.
- Use a suitable workload identity mechanism for AWS API access.
- Scan and update container images.
- Use supported Kubernetes versions and maintained add-ons.
- Configure security groups and network policies where appropriate.
- Use private networking for internal components where practical.
- Set resource requests and limits thoughtfully.
- Configure logs, alerts, backups, and incident response.
- Review public endpoints and load-balancer exposure.

Security is a shared responsibility. Managed control-plane infrastructure does not secure application code or every workload configuration automatically.

---

# 16. Create an EKS Cluster — Training Workflow

There are several ways to create an EKS cluster, including the AWS Console, AWS CLI, `eksctl`, and infrastructure-as-code tools. This module uses `eksctl` as an illustrative learning path.

## Step 1 — Prepare tools

Install and configure:

- AWS CLI
- `kubectl`
- `eksctl`

Use versions compatible with your target EKS/Kubernetes version.

## Step 2 — Configure credentials

```bash
aws configure
```

For a role-based or federated environment, use the approved credential method instead of creating long-lived access keys.

Verify:

```bash
aws sts get-caller-identity
```

## Step 3 — Create a training cluster

Example:

```bash
eksctl create cluster \
  --name training-cluster \
  --region ap-south-1 \
  --nodes 2 \
  --node-type t3.medium
```

This is a training example, not a production architecture. Cluster creation may provision billable resources. Check current AWS and `eksctl` documentation, choose a supported Kubernetes version and suitable capacity, and confirm account, region, networking, and budget before running it.

## Step 4 — Configure kubectl

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name training-cluster
```

## Step 5 — Verify access

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

---

# 17. Deploy a Sample Application

Create `web.yaml`:

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
          image: nginx:stable
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
```

Apply:

```bash
kubectl apply -f web.yaml
kubectl rollout status deployment/web
kubectl get pods -o wide
```

Create a Service:

```bash
kubectl expose deployment web \
  --name=web-service \
  --type=ClusterIP \
  --port=80 \
  --target-port=80
```

Verify:

```bash
kubectl get deployment
kubectl get svc
kubectl get endpointslices
```

Test from inside the cluster:

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  -it --rm \
  -- curl -sS http://web-service
```

If the interactive command behaves differently in your terminal, create a temporary debugging Pod and run the request from its shell.

---

# 18. Expose the Application

For a training exercise, you can request a cloud load balancer with a `LoadBalancer` Service:

```bash
kubectl expose deployment web \
  --name=web-public \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80
```

Check:

```bash
kubectl get svc web-public
```

An external address may take time to appear.

**Cost and security warning:** an external load balancer may create billable AWS resources and publicly expose the application. Use it only when needed, understand the exposure, and delete it when finished.

In production, decide whether the application should be public or internal, then choose the appropriate traffic-entry design.

---

# 19. Practical Lab — Scale and Self-Heal

Scale the Deployment:

```bash
kubectl scale deployment web --replicas=5
kubectl get pods
```

Watch Pods:

```bash
kubectl get pods -w
```

Delete one Pod:

```bash
kubectl delete pod <pod-name>
```

Observe the replacement:

```bash
kubectl get pods -w
```

Questions:

1. Why did a replacement Pod appear?
2. Did the Service name change?
3. Did you need to update the Service when scaling?
4. What happens if all replicas are unhealthy?

---

# 20. Practical Lab — ECR to EKS

Build an application image and publish it to ECR.

```text
Source Code
    |
    v
Docker Build
    |
    v
Amazon ECR
    |
    v
Deployment image field
    |
    v
EKS Pods
```

Update your Deployment image to the ECR image URI:

```yaml
containers:
  - name: app
    image: <account-id>.dkr.ecr.<region>.amazonaws.com/<repository>:<version>
```

Apply:

```bash
kubectl apply -f app-deployment.yaml
kubectl rollout status deployment/app
kubectl describe pod <pod-name>
```

If the image fails to pull, check:

- image URI and tag
- repository and region
- node or workload image-pull permissions
- registry authentication configuration
- network connectivity
- Pod events

---

# 21. Troubleshooting Common EKS Problems

## `kubectl` cannot reach the cluster

```bash
aws sts get-caller-identity
aws eks update-kubeconfig --region <region> --name <cluster-name>
kubectl cluster-info
kubectl config current-context
kubectl config get-contexts
```

Possible causes: wrong AWS identity, wrong region or cluster name, missing cluster access permissions, restricted API endpoint network access, or stale kubeconfig context.

## Nodes are not Ready

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get pods -n kube-system
```

Investigate compute capacity, node IAM configuration, networking, bootstrap, and system components.

## Pods are Pending

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

Possible causes include insufficient resources, scheduling constraints, taints, or storage requirements.

## `ImagePullBackOff`

```bash
kubectl describe pod <pod-name>
```

Verify image URI, tag, registry access, and permissions.

## Service is unreachable

```bash
kubectl get svc
kubectl get endpointslices
kubectl get pods --show-labels
kubectl describe svc <service-name>
```

Then verify DNS, ports, readiness, NetworkPolicies, and networking.

## External load balancer does not appear

```bash
kubectl describe svc <service-name>
kubectl get events --sort-by=.lastTimestamp
```

Review controller/add-on setup, subnet discovery and tags where applicable, permissions, Service configuration, and AWS events.

---

# 22. EKS vs Self-Managed Kubernetes

| Area | EKS | Self-managed Kubernetes |
|---|---|---|
| Control-plane operations | AWS manages EKS control-plane infrastructure | Organization manages it |
| Worker nodes | Multiple compute options | Organization typically manages them |
| AWS integration | Native integration options | Must be configured and maintained |
| Setup | Often faster | More setup and maintenance |
| Control-plane customization | Limited to supported service capabilities | Greater control |
| On-premises | Not the usual EKS model | Suitable |
| Operational responsibility | Reduced for managed components | Higher |
| Kubernetes skills | Still required | Required, including infrastructure operations |

---

# 23. Mini Project — Deploy a Three-Tier Application to EKS

## Architecture

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
                    +------+------+
                    |             |
                    v             v
                Backend Pod   Backend Pod
                           |
                           v
                      Data Layer
                    (e.g. Amazon RDS)
```

## Components

- Docker for packaging.
- ECR for image storage.
- EKS for Kubernetes orchestration.
- Deployments for frontend and backend replicas.
- ClusterIP Service for internal backend communication.
- Appropriate external entry point for the frontend.
- RDS or another suitable data layer.
- CloudWatch or another observability setup.

## Tasks

1. Containerize frontend and backend.
2. Push versioned images to ECR.
3. Deploy both workloads to EKS.
4. Create a Service for each workload.
5. Keep backend internal unless external access is explicitly required.
6. Configure application settings with ConfigMaps and sensitive values through an appropriate Secrets approach.
7. Scale backend and observe endpoints.
8. Delete a Pod and verify self-healing.
9. Check logs and events.
10. Document AWS-managed and team-managed responsibilities.

## Acceptance Criteria

- Frontend Pods become Ready.
- Backend Pods become Ready.
- Frontend reaches backend through its Kubernetes Service DNS name.
- Backend does not depend on fixed Pod IPs.
- Application recovers from deletion of one Pod.
- Student can explain AWS and Kubernetes responsibilities.

---

# 24. Cost and Cleanup

EKS and supporting resources can incur charges. Costs may include the EKS cluster, EC2 instances, load balancers, storage, data transfer, logging, and other services.

Before creating resources:

- confirm the AWS account and region
- estimate the cost
- choose suitable instance sizes
- avoid leaving test environments running
- identify resources created by the lab

After the lab, delete resources you no longer need.

If the cluster was created with `eksctl`, a typical cleanup command is:

```bash
eksctl delete cluster \
  --name training-cluster \
  --region ap-south-1
```

Review the resources and command behavior for your environment before cleanup. Back up any data you need. Resources created separately or retained by configuration may require separate cleanup.

---

# 25. Interview Questions and Answers

## Q1. What is Amazon EKS?

Amazon EKS is AWS's managed service for running Kubernetes.

## Q2. Does EKS replace Kubernetes?

No. EKS provides managed Kubernetes control-plane infrastructure and AWS integrations. Kubernetes concepts and APIs remain central.

## Q3. What does AWS manage in EKS?

AWS manages the EKS control-plane infrastructure. Worker compute and other components depend on the selected options.

## Q4. What is the difference between EKS and ECS?

EKS runs Kubernetes. ECS is AWS's own container orchestration service.

## Q5. What is a Managed Node Group?

An EKS feature that provides managed lifecycle operations for groups of EC2 worker nodes.

## Q6. What is EKS Fargate?

A compute option that can run supported Pods without requiring you to manage EC2 worker nodes for those Pods.

## Q7. What is EKS Auto Mode?

It automates additional compute and cluster infrastructure lifecycle tasks within the supported feature set.

## Q8. What is Amazon ECR?

A managed container image registry that can store images for EKS workloads.

## Q9. Why does EKS need a VPC?

The VPC provides network infrastructure for supported cluster and workload networking designs.

## Q10. How do you connect kubectl to EKS?

```bash
aws eks update-kubeconfig --region <region> --name <cluster-name>
```

The caller must have appropriate AWS credentials and cluster access.

## Q11. How do you expose an application from EKS?

Depending on requirements, use a Service of type `LoadBalancer`, an Ingress controller, or another supported traffic-entry architecture.

## Q12. What is ECR's role in CI/CD?

It stores versioned images that a pipeline can publish and EKS workloads can pull.

## Q13. How does EKS integrate with IAM?

IAM controls AWS API access and participates in cluster access and workload identity patterns. Kubernetes RBAC governs permissions to Kubernetes resources.

## Q14. How do you troubleshoot a Pod that cannot pull an image from ECR?

Inspect Pod events, image URI/tag, registry and region, node or workload permissions, authentication configuration, and network access.

## Q15. Does HPA add worker nodes?

No. HPA changes Pod replica counts. A node-scaling mechanism is needed when more compute capacity is required.

## Q16. Does EKS automatically make an application highly available?

No. Availability depends on replicas, readiness, placement, capacity, networking, dependencies, and configuration.

## Q17. When should a database be outside Kubernetes?

A managed service such as RDS can be a good choice when its operational model, availability, backup, and recovery features meet the application's needs.

## Q18. What should you monitor in EKS?

Application health, Pod restarts, resource use, node health, request latency, errors, load-balancer health, and important dependencies.

## Q19. What is the difference between a Kubernetes Service and an AWS Load Balancer?

A Kubernetes Service is an API abstraction for stable access to workloads. An AWS Load Balancer is cloud infrastructure that can distribute traffic. Integrations connect these layers.

## Q20. What should you consider before creating an EKS cluster?

Region, Kubernetes version, networking, access model, compute choice, workload requirements, security, observability, cost, and cleanup plan.

---

# 26. Interview Scenarios

## Scenario A — Pods Cannot Pull ECR Images

**Question:** The Deployment exists, but Pods show `ImagePullBackOff`. What do you investigate?

**Answer:**

1. Confirm image URI and tag.
2. Check Pod events.
3. Verify image registry access and permissions.
4. Verify the region and repository.
5. Check node/workload configuration and network connectivity.

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

## Scenario B — Application Is Running but Not Publicly Reachable

**Question:** Pods are Ready, but the application cannot be accessed from the internet. What do you check?

**Answer:**

1. Is there an external entry point?
2. Is the Service or Ingress configured correctly?
3. Are healthy endpoints available?
4. Are load-balancer subnets and security controls correct?
5. Is the application listening on the target port?
6. Are controller events reporting errors?

## Scenario C — HPA Scales Pods but They Stay Pending

**Question:** HPA increases replicas, but new Pods remain Pending. Why?

**Answer:**

The cluster may lack compute capacity, or scheduling constraints may prevent placement. Inspect Pod events and verify that the node-scaling mechanism is configured and able to add capacity.

## Scenario D — A Pod Needs S3 Access

**Question:** Should you put AWS access keys in the container image?

**Answer:**

No. Use a supported workload identity mechanism, such as EKS Pod Identity or IRSA, and grant only the permissions the workload needs.

---

# 27. Common Beginner Mistakes

1. Assuming EKS manages every part of the application.
2. Confusing IAM permissions with Kubernetes RBAC.
3. Putting AWS credentials inside images or source code.
4. Exposing every microservice publicly.
5. Assuming a Running Pod is necessarily Ready.
6. Using `latest` for every production deployment.
7. Forgetting resource requests and limits.
8. Assuming HPA automatically scales worker nodes.
9. Ignoring load-balancer and data-transfer costs.
10. Leaving test clusters and cloud resources running.
11. Using public subnets and broad permissions without considering the threat model.
12. Treating EKS as a substitute for understanding Kubernetes.

---

# 28. Quick Revision Cheat Sheet

## Core services

```text
EKS        → Managed Kubernetes service
ECR        → Container image registry
VPC        → AWS network
IAM        → AWS identity and permissions
ALB/NLB    → Load-balancing infrastructure
EBS/EFS    → Persistent storage options
CloudWatch → Logs, metrics, alarms
RDS        → Managed relational database
```

## Common commands

```bash
aws sts get-caller-identity

aws eks update-kubeconfig \
  --region <region> \
  --name <cluster-name>

kubectl cluster-info
kubectl get nodes
kubectl get pods -A
kubectl get svc
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deployment/<deployment-name>
```

## Request flow

```text
Client
  |
  v
AWS Load Balancer / Entry Point
  |
  v
Kubernetes Service / Routing
  |
  v
Application Pod
  |
  v
Database or Other Dependency
```

## Image workflow

```text
Source
  |
  v
Docker Build
  |
  v
ECR
  |
  v
EKS Deployment
  |
  v
Pods
```

---

# 29. Homework

## Beginner

1. Explain EKS in your own words.
2. Draw the control plane and worker compute architecture.
3. Explain the difference between EKS and ECS.
4. Explain the role of ECR.
5. Connect `kubectl` to a training cluster and list nodes.

## Intermediate

1. Deploy an NGINX Deployment with three replicas.
2. Create a ClusterIP Service.
3. Test the Service from another Pod.
4. Scale the Deployment to five replicas.
5. Delete one Pod and observe self-healing.
6. Diagnose a deliberately incorrect Service `targetPort`.

## Advanced

1. Build a Docker image for a simple application.
2. Push a versioned image to ECR.
3. Deploy it to EKS.
4. Expose only the required entry point.
5. Add resource requests and limits.
6. Configure logs and useful alerts.
7. Document IAM, network, and security choices.
8. Estimate costs and clean up lab resources.

---

# 30. Key Takeaways

1. EKS is AWS's managed Kubernetes service.
2. AWS manages the EKS control-plane infrastructure.
3. Workload compute can use different supported models, each with its own responsibilities.
4. ECR stores container images for deployment.
5. VPC networking, security groups, IAM, and Kubernetes RBAC are distinct concepts that must work together.
6. Load balancing requires correct Kubernetes and AWS configuration.
7. Persistent storage must be selected based on workload requirements.
8. CloudWatch and other observability tools need configuration to provide useful visibility.
9. HPA scales Pods; node-scaling mechanisms handle compute capacity.
10. Managed Kubernetes does not automatically make an application secure, scalable, or highly available.
11. Cost management and resource cleanup are part of operating EKS responsibly.
12. Production EKS design requires decisions about access, networking, compute, storage, monitoring, security, and recovery.

---

# 31. Final Mental Model

```text
                 Amazon EKS
                     |
          +----------+----------+
          |                     |
          v                     v
  Managed Control Plane     Workload Compute
                                |
                                v
                         Kubernetes Pods
                                |
             +------------------+------------------+
             |                  |                  |
             v                  v                  v
            ECR             Load Balancer       Storage/DB
         (images)            (traffic)            (data)
             |
             v
          Monitoring
```

Remember:

> EKS provides managed Kubernetes control-plane infrastructure and AWS integration options. Your job is to design, deploy, secure, observe, and operate the workloads responsibly.

---

# Next Module

## Module 15 — EKS Integration with AWS Services

Topics:

- EKS and IAM
- EKS and VPC
- EKS and ECR
- EKS and ALB/NLB
- AWS Load Balancer Controller
- EBS CSI and EFS CSI concepts
- EKS and CloudWatch
- Workload identity
- Secrets management
- Networking and security groups
- End-to-end deployment workflow
- Practical AWS integration project
- Troubleshooting
- Interview questions
