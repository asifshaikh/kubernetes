# Module 7 — Kubernetes API and kubectl Basics

## Learning Objectives

By the end of this module, students should be able to:

- Explain the Kubernetes API and API Server.
- Explain how `kubectl` communicates with Kubernetes.
- Understand kubeconfig and contexts.
- Work with namespaces.
- Create, inspect, update and delete Kubernetes resources.
- Understand Kubernetes YAML manifests.
- Use common `kubectl` commands.
- Understand imperative vs declarative management.
- Explore API groups and versions.
- Troubleshoot basic Kubernetes resources.
- Use `kubectl` with AWS EKS.

---

## 1. What Is the Kubernetes API?

The **Kubernetes API** is the interface through which users, controllers, operators and applications communicate with a Kubernetes cluster.

Think of it as the **front door of Kubernetes**.

```text
User / Tool
     |
     v
Kubernetes API Server
     |
     +---- Controllers
     |
     +---- Scheduler
     |
     +---- Cluster State
```

Clients normally do not directly modify Kubernetes' internal state. They communicate through the API Server.

Common API clients include:

- `kubectl`
- Controllers
- Operators
- CI/CD systems
- Monitoring and automation tools

---

## 2. What Is the API Server?

The **kube-apiserver** is the central API endpoint of the Kubernetes control plane.

It:

- Receives API requests.
- Authenticates clients.
- Authorizes requested actions.
- Performs admission processing.
- Validates Kubernetes objects.
- Provides access to Kubernetes resources.
- Coordinates persistence of cluster state.

Conceptually:

```text
kubectl
   |
   | HTTPS
   v
API Server
   |
   +--> Authentication
   |
   +--> Authorization
   |
   +--> Admission / Validation
   |
   +--> Kubernetes control-plane state
```

### Interview point

`kubectl` normally communicates with the **API Server**, not directly with kubelet.

---

## 3. Kubernetes API Uses HTTP/HTTPS

The Kubernetes API is exposed through HTTP/HTTPS endpoints.

Examples include:

```text
/api/v1
/apis/apps/v1
/apis/batch/v1
```

A simplified Pod endpoint looks like:

```text
/api/v1/namespaces/default/pods
```

A Deployment endpoint looks like:

```text
/apis/apps/v1/namespaces/default/deployments
```

You normally let `kubectl` construct these requests rather than manually building URLs.

---

## 4. What Is kubectl?

`kubectl` is the **command-line client for Kubernetes**.

It allows you to:

- Create resources.
- Read resources.
- Update resources.
- Delete resources.
- Inspect resources.
- View logs.
- Execute commands inside containers.
- Manage contexts.
- Work with namespaces.

Basic syntax:

```bash
kubectl <command> <resource> <name> [options]
```

Examples:

```bash
kubectl get pods
kubectl describe pod nginx
kubectl delete pod nginx
```

---

## 5. How kubectl Works

When you run:

```bash
kubectl get pods
```

the conceptual flow is:

```text
Terminal
   |
   v
kubectl
   |
   | HTTPS API request
   v
API Server
   |
   +--> Authenticate
   |
   +--> Authorize
   |
   +--> Validate / Admit
   |
   v
Cluster state
   |
   v
API Server
   |
   v
kubectl
   |
   v
Terminal output
```

The key mental model:

> **kubectl is a client; the API Server is the gateway to Kubernetes.**

---

# 6. kubeconfig

How does `kubectl` know:

- Which cluster to contact?
- Which credentials to use?
- Which context to use?

It uses a **kubeconfig** file.

A common default location is:

```bash
~/.kube/config
```

Inspect it:

```bash
kubectl config view
```

Check the current context:

```bash
kubectl config current-context
```

List contexts:

```bash
kubectl config get-contexts
```

---

# 7. Kubernetes Contexts

A **context** tells kubectl which combination of:

- Cluster
- User/credentials
- Namespace

to use.

Imagine you have:

```text
dev
staging
production
```

contexts:

```text
dev-context
staging-context
prod-context
```

Switch context:

```bash
kubectl config use-context dev-context
```

Then:

```bash
kubectl get pods
```

uses that context.

### Production safety rule

Before destructive operations:

```bash
kubectl config current-context
```

Always verify where you are connected.

---

# 8. Kubernetes Namespaces

A **Namespace** provides logical scope for namespaced Kubernetes resources.

Example:

```text
Cluster
 |
 +-- dev
 |    +-- frontend
 |    +-- backend
 |
 +-- staging
 |    +-- frontend
 |    +-- backend
 |
 +-- production
      +-- frontend
      +-- backend
```

List namespaces:

```bash
kubectl get namespaces
```

or:

```bash
kubectl get ns
```

Create one:

```bash
kubectl create namespace dev
```

Get Pods in it:

```bash
kubectl get pods -n dev
```

### Important

Namespaces are useful for organization and resource scoping, but a namespace alone is **not a complete security boundary**.

Security can additionally involve:

- RBAC
- NetworkPolicies
- Pod Security controls
- ResourceQuotas
- LimitRanges

---

# 9. kubectl get

`kubectl get` provides a concise view of resources.

```bash
kubectl get pods
kubectl get nodes
kubectl get deployments
kubectl get services
```

Useful variations:

```bash
kubectl get pods -o wide
kubectl get pod <name> -o yaml
kubectl get pod <name> -o json
kubectl get pods -A
kubectl get pods -w
```

`-A` means all namespaces.

`-w` watches for changes.

---

# 10. kubectl describe

`describe` gives detailed information.

```bash
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
kubectl describe node <node-name>
```

Pay particular attention to:

```text
Events
```

For example, if a Pod is Pending:

```bash
kubectl describe pod <pod-name>
```

may reveal:

```text
FailedScheduling
```

This is one of the most important Kubernetes troubleshooting commands.

---

# 11. kubectl logs

View container logs:

```bash
kubectl logs <pod-name>
```

For a specific container:

```bash
kubectl logs <pod-name> -c <container-name>
```

Follow logs:

```bash
kubectl logs -f <pod-name>
```

View logs from a previous container instance:

```bash
kubectl logs <pod-name> --previous
```

Useful for finding:

- Application crashes.
- Configuration errors.
- Startup failures.
- Dependency failures.

---

# 12. kubectl exec

Execute a command inside a running container:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

Then:

```bash
hostname
env
ls
```

Exit:

```bash
exit
```

For a multi-container Pod:

```bash
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
```

### Important

Not every container image contains `/bin/sh`. Minimal or distroless images may not provide a shell.

---

# 13. kubectl port-forward

Port forwarding lets you temporarily access a Pod or Service from your local machine.

Example:

```bash
kubectl port-forward pod/nginx 8080:80
```

Then open:

```text
http://localhost:8080
```

You can also forward a Service:

```bash
kubectl port-forward service/nginx 8080:80
```

This is useful for:

- Development.
- Debugging.
- Accessing internal applications.
- Testing dashboards.

It is generally a debugging/development technique rather than a production exposure mechanism.

---

# 14. kubectl create

`kubectl create` is commonly used imperatively.

Example:

```bash
kubectl create deployment nginx --image=nginx
```

With replicas:

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

Create a namespace:

```bash
kubectl create namespace dev
```

---

# 15. kubectl apply

`kubectl apply` is commonly used for **declarative configuration**.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

Save as:

```text
nginx.yaml
```

Apply:

```bash
kubectl apply -f nginx.yaml
```

Change:

```yaml
replicas: 5
```

and apply again:

```bash
kubectl apply -f nginx.yaml
```

Kubernetes works toward the desired state.

---

# 16. Imperative vs Declarative

## Imperative

You tell Kubernetes what action to perform:

```bash
kubectl create deployment nginx --image=nginx
```

Think:

> "Do this."

## Declarative

You describe the desired state:

```yaml
replicas: 3
image: nginx
```

Then:

```bash
kubectl apply -f nginx.yaml
```

Think:

> "Make the cluster look like this."

Declarative configuration is especially useful for:

- GitOps.
- CI/CD.
- Version control.
- Repeatable deployments.
- Team collaboration.

---

# 17. kubectl delete

Delete a Pod:

```bash
kubectl delete pod nginx
```

Delete a Deployment:

```bash
kubectl delete deployment nginx
```

Delete resources defined by YAML:

```bash
kubectl delete -f nginx.yaml
```

Delete a namespace:

```bash
kubectl delete namespace dev
```

### Warning

Deleting a namespace can delete namespaced resources inside it.

Always verify your context before destructive commands.

---

# 18. kubectl edit

Edit a live resource:

```bash
kubectl edit deployment nginx
```

This opens the resource in your configured editor.

It is useful for quick troubleshooting, but production workflows generally prefer updating version-controlled manifests and applying them through a controlled deployment process.

---

# 19. kubectl explain

`kubectl explain` provides schema documentation.

Try:

```bash
kubectl explain pod
kubectl explain pod.spec
kubectl explain deployment.spec
kubectl explain deployment.spec.template.spec.containers
```

This is an excellent learning and troubleshooting tool.

---

# 20. API Resources

List supported Kubernetes resource types:

```bash
kubectl api-resources
```

You may see:

```text
NAME          SHORTNAMES   APIVERSION
pods          po           v1
services      svc          v1
deployments   deploy       apps/v1
configmaps    cm           v1
secrets                    v1
```

This tells you:

- Resource name.
- Short name.
- API version.
- Whether it is namespaced, among other information.

---

# 21. API Groups and Versions

Examples:

```yaml
apiVersion: v1
```

and:

```yaml
apiVersion: apps/v1
```

Common examples:

```text
Pod          -> v1
Service      -> v1
Deployment   -> apps/v1
Job          -> batch/v1
Ingress      -> networking.k8s.io/v1
```

Core resources commonly use:

```text
v1
```

Grouped resources use forms such as:

```text
apps/v1
batch/v1
networking.k8s.io/v1
```

List supported API versions:

```bash
kubectl api-versions
```

---

# 22. Labels and Selectors

Labels are key-value metadata used to identify and organize objects.

Example:

```yaml
metadata:
  labels:
    app: frontend
    environment: production
```

Select Pods:

```bash
kubectl get pods -l app=frontend
```

Multiple labels:

```bash
kubectl get pods -l app=frontend,environment=production
```

Services and Deployments heavily depend on labels and selectors.

---

# 23. Labels vs Annotations

### Labels

Used primarily to identify/select objects:

```yaml
labels:
  app: frontend
```

### Annotations

Used primarily to store additional metadata for tools and controllers:

```yaml
annotations:
  example.com/owner: platform-team
```

Simple rule:

```text
Labels       -> identify/select
Annotations  -> store metadata
```

---

# 24. kubectl auth can-i

Check whether your current identity is allowed to perform an action.

```bash
kubectl auth can-i get pods
```

Check deletion:

```bash
kubectl auth can-i delete deployments
```

Check a namespace:

```bash
kubectl auth can-i get pods -n production
```

This is useful when troubleshooting RBAC permissions.

---

# 25. YAML Structure

A basic Kubernetes manifest generally contains:

```yaml
apiVersion: ...
kind: ...
metadata:
  ...
spec:
  ...
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:alpine
```

Meaning:

```text
apiVersion
    |
    +--> API schema/version

kind
    |
    +--> Resource type

metadata
    |
    +--> Name, labels, annotations

spec
    |
    +--> Desired configuration
```

When retrieved from the API, objects also have a `status` section representing observed state. Kubernetes normally manages `status`; users generally define `spec`.

---

# 26. Practical Lab 1 — First kubectl Investigation

Run:

```bash
kubectl version
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
kubectl api-resources
kubectl api-versions
```

Write down what each command tells you.

---

# 27. Practical Lab 2 — Create a Deployment

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
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f web.yaml
```

Inspect:

```bash
kubectl get deployment
kubectl get replicasets
kubectl get pods
kubectl get pods -o wide
```

---

# 28. Practical Lab 3 — Inspect Resources

Deployment:

```bash
kubectl describe deployment web
```

ReplicaSet:

```bash
kubectl get rs
kubectl describe rs <replicaset-name>
```

Pod:

```bash
kubectl describe pod <pod-name>
```

YAML:

```bash
kubectl get deployment web -o yaml
```

JSON:

```bash
kubectl get deployment web -o json
```

---

# 29. Practical Lab 4 — Logs and Exec

Find a Pod:

```bash
kubectl get pods
```

View logs:

```bash
kubectl logs <pod-name>
```

Enter the container:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

Inside:

```bash
hostname
cat /etc/os-release
```

Exit:

```bash
exit
```

---

# 30. Practical Lab 5 — Port Forward

Run:

```bash
kubectl port-forward pod/<pod-name> 8080:80
```

Open:

```text
http://localhost:8080
```

Stop with:

```text
Ctrl+C
```

---

# 31. Practical Lab 6 — Change Desired State

Start with:

```yaml
replicas: 3
```

Apply:

```bash
kubectl apply -f web.yaml
```

Change to:

```yaml
replicas: 5
```

Apply:

```bash
kubectl apply -f web.yaml
```

Watch:

```bash
kubectl get pods -w
```

Observe Kubernetes create additional Pods.

The flow is:

```text
YAML
 |
 | desired replicas = 5
 v
API Server
 |
 v
Deployment Controller
 |
 v
ReplicaSet
 |
 v
5 Pods
```

---

# 32. Practical Lab 7 — Use kubectl explain

Run:

```bash
kubectl explain deployment
kubectl explain deployment.spec
kubectl explain deployment.spec.template
kubectl explain deployment.spec.template.spec.containers
```

Find:

- Where replicas are defined.
- Where labels are defined.
- Where the image is defined.
- Where container ports are defined.

---

# 33. Practical Lab 8 — Explore the Kubernetes API

Run:

```bash
kubectl proxy
```

In another terminal:

```bash
curl http://127.0.0.1:8001/api
```

and:

```bash
curl http://127.0.0.1:8001/apis
```

This demonstrates that Kubernetes exposes an HTTP API and `kubectl` is one client for that API.

Stop the proxy with:

```text
Ctrl+C
```

---

# 34. Practical Lab 9 — Check Permissions

Run:

```bash
kubectl auth can-i get pods
kubectl auth can-i delete deployments
kubectl auth can-i get pods -n kube-system
```

This introduces the RBAC concepts used later in Kubernetes security.

---

# 35. Troubleshooting Workflow

Use a structured process.

### Step 1 — Check context

```bash
kubectl config current-context
```

### Step 2 — Check cluster

```bash
kubectl cluster-info
```

### Step 3 — Check nodes

```bash
kubectl get nodes
```

### Step 4 — Check resource status

```bash
kubectl get pods
```

### Step 5 — Get details

```bash
kubectl describe pod <pod-name>
```

### Step 6 — Check logs

```bash
kubectl logs <pod-name>
```

### Step 7 — Check events

```bash
kubectl get events --sort-by=.lastTimestamp
```

### Step 8 — Check permissions

```bash
kubectl auth can-i <verb> <resource>
```

This workflow handles a large percentage of beginner Kubernetes troubleshooting.

---

# 36. AWS EKS and kubectl

With Amazon EKS, AWS CLI can be used to configure kubectl access.

A common workflow is:

```text
AWS CLI
   |
   v
EKS cluster information
   |
   v
kubeconfig
   |
   v
kubectl
   |
   v
EKS API Server
```

A commonly used command is:

```bash
aws eks update-kubeconfig --region <region> --name <cluster-name>
```

Then:

```bash
kubectl config current-context
kubectl get nodes
```

Authentication and authorization depend on your AWS/EKS configuration and IAM permissions.

---

# 37. Mini Project — Kubernetes Operations Toolkit

## Scenario

You are a junior DevOps engineer responsible for a web application:

```text
Deployment
    |
    +-- 3 nginx Pods
```

Your job is to investigate and manage it entirely using `kubectl`.

### Task 1 — Deploy

Create a Deployment with 3 replicas.

### Task 2 — Inspect

```bash
kubectl get deployment
kubectl get rs
kubectl get pods -o wide
```

### Task 3 — Investigate

```bash
kubectl describe deployment <name>
kubectl describe pod <name>
```

### Task 4 — Logs

```bash
kubectl logs <pod-name>
```

### Task 5 — Shell

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

### Task 6 — Local access

Use `kubectl port-forward` to access nginx locally.

### Task 7 — Scale

Change:

```text
3 replicas -> 5 replicas
```

using YAML.

### Task 8 — Context

```bash
kubectl config current-context
```

### Task 9 — API discovery

```bash
kubectl api-resources
kubectl api-versions
```

### Task 10 — Documentation

Use:

```bash
kubectl explain deployment
```

to investigate the Deployment schema.

---

# 38. Interview Questions and Answers

## Q1. What is kubectl?

**Answer:** `kubectl` is the command-line client used to communicate with the Kubernetes API Server and manage Kubernetes resources.

## Q2. Does kubectl directly communicate with kubelet?

**Answer:** Normally no. kubectl communicates with the Kubernetes API Server. The control plane and node components then perform the required work.

## Q3. What is kubeconfig?

**Answer:** kubeconfig contains configuration used by kubectl to identify clusters, credentials and contexts.

## Q4. What is a Kubernetes context?

**Answer:** A context specifies the cluster, user/credentials and namespace that kubectl should use.

## Q5. What is a namespace?

**Answer:** A namespace provides logical scope for namespaced Kubernetes resources and helps organize resources within a cluster.

## Q6. Difference between `kubectl get` and `kubectl describe`?

**Answer:** `get` provides a concise resource view; `describe` provides detailed information including conditions and events.

## Q7. How do you see Pod logs?

```bash
kubectl logs <pod-name>
```

## Q8. How do you enter a running container?

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

assuming the image contains that shell.

## Q9. What does `kubectl apply` do?

**Answer:** It submits declarative configuration to Kubernetes and allows Kubernetes to reconcile the actual state toward the desired state.

## Q10. What is imperative vs declarative management?

**Answer:**

Imperative:

```bash
kubectl create deployment nginx --image=nginx
```

Declarative:

```yaml
replicas: 3
image: nginx
```

followed by:

```bash
kubectl apply -f deployment.yaml
```

## Q11. What is `kubectl explain`?

**Answer:** It displays Kubernetes resource schema/documentation information.

## Q12. How do you list Kubernetes resource types?

```bash
kubectl api-resources
```

## Q13. How do you list supported API versions?

```bash
kubectl api-versions
```

## Q14. How do you check whether you have permission to perform an action?

```bash
kubectl auth can-i <verb> <resource>
```

## Q15. How do you determine which cluster kubectl is using?

```bash
kubectl config current-context
```

---

# 39. Common Beginner Mistakes

### Mistake 1 — Running commands against the wrong cluster

Always check:

```bash
kubectl config current-context
```

### Mistake 2 — Using only `get`

When something is wrong, also use:

```bash
kubectl describe
kubectl logs
kubectl get events
```

### Mistake 3 — Confusing `apply` and `create`

`create` is commonly imperative.

`apply` is commonly declarative.

### Mistake 4 — Assuming every image has a shell

This may fail:

```bash
kubectl exec -it <pod> -- /bin/sh
```

Minimal images may not include a shell.

### Mistake 5 — Editing production directly

`kubectl edit` is useful, but production changes should generally be reflected in version-controlled configuration.

### Mistake 6 — Treating namespaces as complete security boundaries

Namespaces provide scope and organization. Security requires additional controls such as RBAC and NetworkPolicies.

---

# 40. kubectl Cheat Sheet

| Goal | Command |
|---|---|
| Cluster information | `kubectl cluster-info` |
| Current context | `kubectl config current-context` |
| List contexts | `kubectl config get-contexts` |
| Switch context | `kubectl config use-context <context>` |
| List nodes | `kubectl get nodes` |
| List namespaces | `kubectl get ns` |
| List Pods | `kubectl get pods` |
| All namespaces | `kubectl get pods -A` |
| Detailed information | `kubectl describe <resource> <name>` |
| Pod logs | `kubectl logs <pod>` |
| Previous logs | `kubectl logs <pod> --previous` |
| Shell into Pod | `kubectl exec -it <pod> -- /bin/sh` |
| Port forwarding | `kubectl port-forward ...` |
| Apply YAML | `kubectl apply -f file.yaml` |
| Delete YAML resources | `kubectl delete -f file.yaml` |
| API resources | `kubectl api-resources` |
| API versions | `kubectl api-versions` |
| Resource documentation | `kubectl explain <resource>` |
| Check permission | `kubectl auth can-i ...` |
| Watch resources | `kubectl get pods -w` |
| YAML output | `kubectl get <resource> -o yaml` |
| JSON output | `kubectl get <resource> -o json` |

---

# 41. Quick Revision

```text
                  User
                    |
                    v
                 kubectl
                    |
                  HTTPS
                    |
                    v
              API Server
                    |
          +---------+---------+
          |                   |
          v                   v
      Authentication      Authorization
          |
          v
      Admission/
       Validation
          |
          v
    Kubernetes State
          |
          v
      Controllers
          |
          v
      Worker Nodes
```

Remember:

```bash
kubectl get
kubectl describe
kubectl logs
kubectl exec
kubectl apply
kubectl delete
kubectl explain
kubectl config
kubectl api-resources
kubectl auth can-i
```

---

# 42. Homework

1. Explain the Kubernetes API in your own words.
2. Explain the role of kube-apiserver.
3. Explain how `kubectl get pods` works internally.
4. Explain kubeconfig.
5. Explain Kubernetes context.
6. Create a Deployment using YAML.
7. Scale it from 3 to 5 replicas using `kubectl apply`.
8. Use `kubectl describe` to investigate a Pod.
9. Use `kubectl logs` to inspect application output.
10. Use `kubectl exec` to enter a container.
11. Use `kubectl port-forward` to access nginx locally.
12. Use `kubectl explain` to explore a Deployment.
13. Use `kubectl api-resources` and identify at least 10 resource types.
14. Use `kubectl auth can-i` to test your permissions.
15. Explain imperative vs declarative Kubernetes management.
16. Explain why checking the current context is important before production operations.

---

# Module 7 Takeaway

The key idea is:

> **Almost everything you do with Kubernetes goes through the Kubernetes API Server.**

The core workflow is:

```text
Command / YAML
      |
      v
   kubectl
      |
      v
 API Server
      |
      v
Controllers / Scheduler
      |
      v
 Worker Nodes
      |
      v
 Application
```

Once you are comfortable with:

```bash
kubectl get
kubectl describe
kubectl logs
kubectl exec
kubectl apply
kubectl delete
kubectl explain
kubectl config
```

you have the foundation required for most later Kubernetes topics.

---

# Next Module

## Module 8 — Kubernetes Cluster Networking Concepts

Topics:

- Why Kubernetes networking is needed.
- Container networking.
- Pod networking.
- Pod-to-Pod communication.
- Pod IPs.
- Node networking.
- Service networking.
- ClusterIP.
- DNS and service discovery.
- kube-proxy's role.
- CNI.
- Common CNI implementations.
- Kubernetes networking model.
- Practical networking labs.
- Troubleshooting Pod connectivity.
- AWS VPC/EKS networking connection.
- Mini project.
- Interview questions.
- Common mistakes.
