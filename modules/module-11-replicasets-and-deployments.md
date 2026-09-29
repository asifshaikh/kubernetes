# Module 11 — ReplicaSets and Deployments

## Learning Objectives

By the end of this module, students should be able to:

- Explain ReplicaSets and Deployments.
- Understand desired state vs current state.
- Understand reconciliation.
- Create and manage ReplicaSets.
- Explain why Deployments are normally preferred over direct ReplicaSet management.
- Perform rolling updates and rollbacks.
- Understand `RollingUpdate` and `Recreate`.
- Understand `maxSurge` and `maxUnavailable`.
- Inspect rollout history and revisions.
- Scale Deployments.
- Troubleshoot failed rollouts.
- Understand EKS deployment workflows.
- Answer common interview questions.

---

# 1. Why Do We Need ReplicaSets?

Suppose an application has one Pod:

```text
Pod
 |
 +-- Backend container
```

If that Pod crashes or its node fails, the application may become unavailable.

Instead, we can run:

```text
Pod 1
Pod 2
Pod 3
```

If one disappears:

```text
Pod 1
Pod 2
Pod 3  <-- deleted
```

Kubernetes should create a replacement:

```text
Pod 1
Pod 2
Pod 4  <-- replacement
```

A ReplicaSet helps maintain the desired number of matching Pods.

---

# 2. Desired State vs Current State

Suppose:

```yaml
replicas: 3
```

Desired state:

```text
Desired = 3 Pods
```

But the cluster currently has:

```text
Current = 2 Pods
```

The ReplicaSet controller observes:

```text
Desired: 3
Current: 2
```

and works toward:

```text
Current: 3
```

This is the Kubernetes reconciliation model:

```text
Desired State
      |
      v
Controller
      |
      v
Compare
      |
      v
Current State
      |
      v
Corrective action
```

---

# 3. What Is a ReplicaSet?

A ReplicaSet is a Kubernetes controller that attempts to maintain a stable number of Pods matching a selector.

Example:

```yaml
replicas: 3
```

Architecture:

```text
ReplicaSet
    |
    +---- Pod
    +---- Pod
    +---- Pod
```

---

# 4. ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs

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
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Save as:

```text
replicaset.yaml
```

Apply:

```bash
kubectl apply -f replicaset.yaml
```

Check:

```bash
kubectl get replicasets
kubectl get pods
```

---

# 5. ReplicaSet Self-Healing Lab

Apply:

```bash
kubectl apply -f replicaset.yaml
```

List Pods:

```bash
kubectl get pods
```

Delete one:

```bash
kubectl delete pod <pod-name>
```

Watch:

```bash
kubectl get pods -w
```

The process is:

```text
3 Pods
  |
  v
1 Pod deleted
  |
  v
2 Pods
  |
  v
ReplicaSet detects mismatch
  |
  v
Replacement Pod created
  |
  v
3 Pods
```

---

# 6. ReplicaSets Use Labels and Selectors

Pods:

```yaml
labels:
  app: backend
```

ReplicaSet:

```yaml
selector:
  matchLabels:
    app: backend
```

The selector identifies the Pods the ReplicaSet manages.

Important:

> Selectors are how Kubernetes controllers identify matching resources.

---

# 7. Why We Usually Don't Manage ReplicaSets Directly

A ReplicaSet maintains replicas, but a Deployment adds release-management features such as:

- Rolling updates
- Rollbacks
- Revision history
- Declarative application updates
- Deployment strategies
- Easier release management

Typical hierarchy:

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pods
```

Therefore:

```text
ReplicaSet = maintains replicas

Deployment = manages application releases and ReplicaSets
```

---

# 8. Deployment Architecture

A Deployment typically creates and manages a ReplicaSet.

```text
Deployment
    |
    v
ReplicaSet
    |
    +---- Pod
    +---- Pod
    +---- Pod
```

During an update, a Deployment can create a new ReplicaSet:

```text
Deployment
    |
    +------------------+
    |                  |
    v                  v
Old ReplicaSet      New ReplicaSet
    |                  |
    +-- Pod            +-- Pod
    +-- Pod            +-- Pod
    +-- Pod            +-- Pod
```

The Deployment gradually transitions from the old ReplicaSet to the new one.

---

# 9. Create a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app

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
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Inspect:

```bash
kubectl get deployment
kubectl get replicasets
kubectl get pods
```

---

# 10. Understanding the Relationship

Run:

```bash
kubectl get deployment,replicaset,pods
```

You may see:

```text
deployment.apps/web-app
replicaset.apps/web-app-xxxxx
pod/web-app-xxxxx-aaaaa
pod/web-app-xxxxx-bbbbb
pod/web-app-xxxxx-ccccc
```

Relationship:

```text
Deployment
     |
     v
ReplicaSet
     |
     +---- Pod
     +---- Pod
     +---- Pod
```

The ReplicaSet name normally contains a hash based on the Pod template.

---

# 11. What Happens When You Change the Image?

Current:

```yaml
image: nginx:1.27
```

Change to:

```yaml
image: nginx:1.28
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

The Deployment creates a new ReplicaSet representing the new Pod template.

Conceptually:

```text
Deployment
   |
   +------------------------+
   |                        |
   v                        v
Old ReplicaSet          New ReplicaSet
nginx:1.27              nginx:1.28
   |                        |
   v                        v
Old Pods                 New Pods
```

The Deployment gradually shifts the workload to the new ReplicaSet.

---

# 12. Rolling Update

The default Deployment strategy is generally:

```text
RollingUpdate
```

It gradually replaces old Pods with new Pods.

Conceptually:

```text
Old:
Pod A
Pod B
Pod C

During update:

Old:
Pod A
Pod B

New:
Pod D

Then:

Old:
Pod A

New:
Pod D
Pod E

Finally:

New:
Pod D
Pod E
Pod F
```

The exact sequence depends on rollout settings and cluster conditions.

---

# 13. Watch a Rolling Update

Change the image and apply:

```bash
kubectl apply -f deployment.yaml
```

Watch:

```bash
kubectl get pods -w
```

Also:

```bash
kubectl rollout status deployment/web-app
```

Inspect ReplicaSets:

```bash
kubectl get replicasets
```

---

# 14. Deployment Strategies

A Deployment supports:

```yaml
strategy:
  type: RollingUpdate
```

and:

```yaml
strategy:
  type: Recreate
```

---

# 15. RollingUpdate Strategy

Example:

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

For:

```yaml
replicas: 3
```

the rollout can temporarily have one extra Pod because:

```yaml
maxSurge: 1
```

---

# 16. What Is maxSurge?

`maxSurge` controls how many Pods can be created above the desired replica count during a rolling update.

Example:

```yaml
replicas: 3
maxSurge: 1
```

Conceptually:

```text
Desired = 3
Maximum temporary total = 4
```

It can also be a percentage:

```yaml
maxSurge: 25%
```

---

# 17. What Is maxUnavailable?

`maxUnavailable` controls how many Pods can be unavailable during a rolling update.

Example:

```yaml
replicas: 4

maxUnavailable: 1
```

This permits up to one unavailable replica during the rollout, subject to the Deployment controller's behavior.

It can also be:

```yaml
maxUnavailable: 25%
```

---

# 18. maxSurge + maxUnavailable

Example:

```yaml
replicas: 4

strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

Conceptually:

```text
Desired = 4
Extra Pods allowed = 1
Unavailable replicas allowed = 1
```

These settings let you balance availability and temporary resource usage.

---

# 19. Recreate Strategy

Example:

```yaml
strategy:
  type: Recreate
```

With Recreate, old Pods are terminated before new Pods are created.

Conceptually:

```text
Old Pods
Pod A
Pod B
Pod C

      ↓

No old Pods

      ↓

New Pods
Pod D
Pod E
Pod F
```

This can cause downtime.

It can be useful when old and new versions should not run simultaneously.

---

# 20. RollingUpdate vs Recreate

| Feature | RollingUpdate | Recreate |
|---|---|---|
| Old/new overlap | Yes | No |
| Typical downtime | Designed to minimize it | Possible |
| Default Deployment strategy | Yes | No |
| Temporary extra resource usage | Possible | Usually no overlap |
| Typical use | Most stateless services | Incompatible version transitions |

The correct choice depends on application requirements.

---

# 21. Deployment Revision History

Check rollout history:

```bash
kubectl rollout history deployment/web-app
```

Example:

```text
deployment.apps/web-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

A revision represents a Deployment rollout version.

---

# 22. Rollout History and Change Cause

You can add a change-cause annotation:

```bash
kubectl annotate deployment web-app   kubernetes.io/change-cause="Updated nginx from 1.27 to 1.28"
```

Then:

```bash
kubectl rollout history deployment/web-app
```

In modern CI/CD workflows, Git history, image tags/digests, and deployment tooling are often also used for release traceability.

---

# 23. Rollback

Suppose:

```text
Version 1 → Working
Version 2 → Working
Version 3 → Broken
```

Inspect:

```bash
kubectl rollout history deployment/web-app
```

Rollback:

```bash
kubectl rollout undo deployment/web-app
```

Check:

```bash
kubectl rollout status deployment/web-app
```

---

# 24. Rollback to a Specific Revision

List revisions:

```bash
kubectl rollout history deployment/web-app
```

Then:

```bash
kubectl rollout undo deployment/web-app --to-revision=2
```

Verify:

```bash
kubectl rollout status deployment/web-app
```

---

# 25. Deployment Scaling

Scale:

```bash
kubectl scale deployment web-app --replicas=5
```

Check:

```bash
kubectl get pods
```

Scale down:

```bash
kubectl scale deployment web-app --replicas=2
```

Declarative approach:

```yaml
spec:
  replicas: 5
```

Then:

```bash
kubectl apply -f deployment.yaml
```

---

# 26. Useful Rollout Commands

```bash
kubectl rollout status deployment/web-app
kubectl rollout history deployment/web-app
kubectl rollout pause deployment/web-app
kubectl rollout resume deployment/web-app
kubectl rollout restart deployment/web-app
kubectl rollout undo deployment/web-app
```

---

# 27. Pausing a Rollout

Pause:

```bash
kubectl rollout pause deployment/web-app
```

Make changes, for example:

```bash
kubectl set image deployment/web-app nginx=nginx:1.28
```

Then resume:

```bash
kubectl rollout resume deployment/web-app
```

This can be useful when grouping multiple Deployment changes into one rollout.

---

# 28. Updating an Image with kubectl

```bash
kubectl set image deployment/web-app   nginx=nginx:1.28
```

Check:

```bash
kubectl rollout status deployment/web-app
```

For declarative or GitOps workflows, the version-controlled deployment configuration should remain the source of truth.

---

# 29. Rollout Failure Example

Suppose the image is invalid:

```yaml
image: nginx:does-not-exist
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl rollout status deployment/web-app
```

Inspect:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

You may see:

```text
ErrImagePull
ImagePullBackOff
```

---

# 30. Troubleshooting a Failed Deployment

Use:

```bash
kubectl get deployment
kubectl describe deployment web-app
kubectl get replicasets
kubectl get pods
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deployment/web-app
```

Possible causes:

- Invalid image
- Image cannot be pulled
- Insufficient resources
- Incorrect container command
- Missing configuration
- Failed readiness probe
- Failed startup probe
- Application crash
- Scheduling constraints
- Node problems
- Volume problems
- Network dependencies

---

# 31. Readiness and Rolling Updates

Readiness probes are important during rolling updates.

Suppose:

```text
Old Pods:
3 Ready

New Pod:
Running but NotReady
```

The new Pod should not be treated as ready application capacity simply because its container started.

Conceptually:

```text
New Pod
   |
   v
Running
   |
   v
Readiness probe
   |
   +---- Fail → NotReady
   |
   +---- Pass → Ready
```

---

# 32. Deployment with RollingUpdate Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app

spec:
  replicas: 4

  strategy:
    type: RollingUpdate

    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

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
          image: nginx:1.27

          ports:
            - containerPort: 80

          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
```

---

# 33. Deployment Lifecycle

A useful mental model:

```text
kubectl apply
      |
      v
API Server
      |
      v
Deployment object
      |
      v
Deployment Controller
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
kubelet
      |
      v
Container Runtime
      |
      v
Container
```

The Deployment controller manages ReplicaSets, while ReplicaSets maintain the desired Pod count.

---

# 34. Deployment vs ReplicaSet vs Pod

| Resource | Main responsibility |
|---|---|
| Pod | Runs one or more containers |
| ReplicaSet | Maintains desired number of matching Pods |
| Deployment | Manages ReplicaSets and application releases |

Mental model:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Container
```

---

# 35. Real-World Example — E-Commerce Backend

Initial release:

```text
Version 1.0
3 Pods
```

Release version 1.1.

With RollingUpdate:

```text
Old 1.0:
Pod A
Pod B
Pod C

New 1.1:
Pod D
```

Gradually:

```text
Old:
Pod B
Pod C

New:
Pod D
Pod E
```

Eventually:

```text
New:
Pod D
Pod E
Pod F
```

The Deployment manages this transition.

---

# 36. AWS/EKS Connection

A common EKS workflow is:

```text
Code
 ↓
GitHub / Source Control
 ↓
CI/CD
 ↓
Docker Build
 ↓
Amazon ECR
 ↓
EKS Deployment
 ↓
ReplicaSet
 ↓
Pods
 ↓
Service
```

Example release flow:

```text
Build Docker image
      ↓
Push image to ECR
      ↓
Update Deployment image
      ↓
Rolling update
      ↓
Readiness checks
      ↓
New version serves traffic
```

---

# 37. EKS Deployment Mini Project

## Objective

Deploy a containerized application to EKS and practice a rolling release.

Architecture:

```text
Amazon ECR
    |
    v
EKS Deployment
    |
    v
ReplicaSet
    |
    +---- Pod
    +---- Pod
    +---- Pod
```

## Step 1 — Build

```bash
docker build -t my-web-app:v1 .
```

## Step 2 — Tag for ECR

```bash
docker tag my-web-app:v1   <account-id>.dkr.ecr.<region>.amazonaws.com/my-web-app:v1
```

## Step 3 — Push

After authenticating Docker to ECR:

```bash
docker push   <account-id>.dkr.ecr.<region>.amazonaws.com/my-web-app:v1
```

## Step 4 — Deploy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app

spec:
  replicas: 3

  strategy:
    type: RollingUpdate

    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: my-web-app

  template:
    metadata:
      labels:
        app: my-web-app

    spec:
      containers:
        - name: web
          image: <account-id>.dkr.ecr.<region>.amazonaws.com/my-web-app:v1
          ports:
            - containerPort: 8080

          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 5
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

## Step 5 — Release v2

Build and push v2.

Then:

```bash
kubectl set image deployment/my-web-app   web=<account-id>.dkr.ecr.<region>.amazonaws.com/my-web-app:v2
```

Watch:

```bash
kubectl rollout status deployment/my-web-app
kubectl get replicasets
kubectl get pods
```

## Step 6 — Roll Back

```bash
kubectl rollout undo deployment/my-web-app
```

---

# 38. Blue/Green Deployment Concept

Kubernetes Deployments natively provide RollingUpdate and Recreate. A classic blue/green deployment can be modeled using separate workload versions and traffic selection.

Conceptually:

```text
              Service
                 |
        +--------+--------+
        |                 |
      Blue              Green
      v1                 v2
```

Example labels:

```yaml
app: web
version: blue
```

and:

```yaml
app: web
version: green
```

A Service can select one version:

```yaml
selector:
  app: web
  version: green
```

Production blue/green implementations may use multiple Services, Ingress/Gateway, service mesh, or progressive delivery tooling.

---

# 39. Canary Deployment Concept

A canary release sends only a portion of traffic to a new version.

```text
Users
  |
  v
Traffic Layer
  |
  +---- v1 → majority
  |
  +---- v2 → small portion
```

A basic Kubernetes Service selector does not provide precise percentage-based traffic splitting.

Production canary implementations may use:

- Ingress capabilities
- Gateway APIs/controllers
- Service meshes
- Progressive delivery tools

The key idea:

> Canary means exposing a new version to a controlled portion of traffic before full rollout.

---

# 40. Deployment Strategy Comparison

| Strategy | Main idea | Typical use |
|---|---|---|
| RollingUpdate | Gradually replace old Pods | Most stateless applications |
| Recreate | Remove old Pods, then create new ones | Incompatible versions |
| Blue/Green | Maintain separate versions and switch traffic | Controlled release switching |
| Canary | Gradually expose a new version | Progressive releases |

RollingUpdate and Recreate are native Deployment strategies. Blue/green and canary generally require additional traffic-management design or tooling.

---

# 41. Common Beginner Mistakes

## Mistake 1 — Editing a ReplicaSet created by a Deployment

Change the Deployment instead.

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
```

## Mistake 2 — Editing individual Pods

Pods created by a Deployment are disposable.

Change the Deployment template.

## Mistake 3 — Assuming `kubectl apply` means rollout succeeded

Verify:

```bash
kubectl rollout status deployment/<name>
```

## Mistake 4 — Ignoring readiness probes

A container can be Running but not Ready.

## Mistake 5 — Using mutable image tags carelessly

Prefer:

```text
my-app:v1.4.2
```

or immutable image digests.

## Mistake 6 — Forgetting rollback

Know:

```bash
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
```

---

# 42. Troubleshooting Checklist

When a Deployment rollout fails:

```text
Deployment status
      ↓
Rollout status
      ↓
ReplicaSets
      ↓
Pods
      ↓
Pod events
      ↓
Container logs
      ↓
Readiness/startup probes
      ↓
Scheduling/resources
      ↓
Image availability
      ↓
Rollback if required
```

Useful commands:

```bash
kubectl get deployment
kubectl describe deployment <name>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl get replicasets
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events --sort-by=.lastTimestamp
```

---

# 43. Interview Questions and Answers

## Q1. What is a ReplicaSet?

A ReplicaSet is a Kubernetes controller that maintains a desired number of matching Pods.

## Q2. Why use Deployments instead of ReplicaSets directly?

Deployments provide higher-level release management such as rolling updates, rollback, and revision history while managing ReplicaSets.

## Q3. What happens when a ReplicaSet-managed Pod is deleted?

The ReplicaSet detects that the current number is below the desired count and creates a replacement.

## Q4. What is desired state?

The configuration you declare that Kubernetes should maintain.

Example:

```yaml
replicas: 3
```

## Q5. What is reconciliation?

Comparing desired state with observed state and taking actions to move the system toward the desired state.

## Q6. What is the default Deployment strategy?

`RollingUpdate`.

## Q7. What is RollingUpdate?

It gradually replaces old application Pods with new Pods.

## Q8. What is Recreate?

It terminates old Pods before creating new Pods.

## Q9. What is maxSurge?

It controls how many additional Pods can temporarily exist above the desired replica count during a rolling update.

## Q10. What is maxUnavailable?

It controls how many Pods can be unavailable during a rolling update.

## Q11. Can maxSurge be a percentage?

Yes:

```yaml
maxSurge: 25%
```

## Q12. Can maxUnavailable be a percentage?

Yes:

```yaml
maxUnavailable: 25%
```

## Q13. How do you check rollout status?

```bash
kubectl rollout status deployment/<name>
```

## Q14. How do you see rollout history?

```bash
kubectl rollout history deployment/<name>
```

## Q15. How do you rollback?

```bash
kubectl rollout undo deployment/<name>
```

## Q16. How do you rollback to a specific revision?

```bash
kubectl rollout undo deployment/<name> --to-revision=2
```

## Q17. How do you scale a Deployment?

```bash
kubectl scale deployment <name> --replicas=5
```

## Q18. What happens when the Pod template changes?

A new ReplicaSet is generally created for the new Pod template, and the Deployment manages the transition.

## Q19. Why are readiness probes important during rolling updates?

They help ensure new Pods are ready before being considered eligible to serve application traffic.

## Q20. Does `kubectl apply` guarantee a successful rollout?

No. It submits the desired configuration. Verify rollout status separately.

---

# 44. Interview Scenario — Replica Count

Deployment:

```yaml
replicas: 5
```

Current:

```text
4 Pods
```

What should happen?

The ReplicaSet should create another Pod so the workload moves toward five replicas.

---

# 45. Interview Scenario — Image Update

Current:

```text
ReplicaSet A
nginx:1.27
3 Pods
```

Change to:

```text
nginx:1.28
```

Conceptually:

```text
Deployment
   |
   +---- Old ReplicaSet
   |       |
   |       +-- old Pods
   |
   +---- New ReplicaSet
           |
           +-- new Pods
```

The Deployment performs the configured transition.

---

# 46. Interview Scenario — maxSurge

Suppose:

```yaml
replicas: 4
maxSurge: 1
```

Conceptually, the desired-plus-surge count can reach:

```text
4 + 1 = 5
```

Other rollout conditions, including readiness and `maxUnavailable`, affect the exact observed state.

---

# 47. Interview Scenario — maxUnavailable

Suppose:

```yaml
replicas: 4
maxUnavailable: 1
```

This permits up to one unavailable replica during the rolling update, subject to Deployment rollout behavior.

---

# 48. Practical Lab — Full Deployment Lifecycle

## Step 1 — Create

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lifecycle-demo

spec:
  replicas: 3

  selector:
    matchLabels:
      app: lifecycle-demo

  template:
    metadata:
      labels:
        app: lifecycle-demo

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

Apply:

```bash
kubectl apply -f lifecycle-demo.yaml
```

## Step 2 — Verify

```bash
kubectl get deployment
kubectl get replicasets
kubectl get pods
```

## Step 3 — Update

Change:

```yaml
image: nginx:1.28
```

Apply:

```bash
kubectl apply -f lifecycle-demo.yaml
```

## Step 4 — Watch

```bash
kubectl rollout status deployment/lifecycle-demo
kubectl get replicasets
kubectl get pods
```

## Step 5 — History

```bash
kubectl rollout history deployment/lifecycle-demo
```

## Step 6 — Rollback

```bash
kubectl rollout undo deployment/lifecycle-demo
```

## Step 7 — Verify

```bash
kubectl rollout status deployment/lifecycle-demo
kubectl get replicasets
kubectl get pods
```

---

# 49. Practical Lab — maxSurge and maxUnavailable

Use:

```yaml
replicas: 4

strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Update the image and watch:

```bash
kubectl get pods -w
```

Observe:

```text
Desired = 4
Maximum surge = 1
Unavailable = 0
```

---

# 50. Practical Lab — Failed Rollout and Rollback

Start with:

```yaml
image: nginx:1.27
```

Then intentionally use:

```yaml
image: nginx:does-not-exist
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl rollout status deployment/web-app
kubectl get pods
kubectl describe pod <pod-name>
```

You may see:

```text
ErrImagePull
ImagePullBackOff
```

Rollback:

```bash
kubectl rollout undo deployment/web-app
```

Verify:

```bash
kubectl rollout status deployment/web-app
```

---

# 51. Final Mental Model

```text
Deployment
    |
    | manages
    v
ReplicaSet
    |
    | maintains
    v
Pods
    |
    | run
    v
Containers
```

During an update:

```text
Deployment
    |
    +----------------------+
    |                      |
    v                      v
Old ReplicaSet         New ReplicaSet
    |                      |
 Old Pods                New Pods
    |                      |
    +----------+-----------+
               |
        Rolling transition
```

The Deployment is normally the object you interact with.

---

# 52. Module Summary

You should now understand:

- ReplicaSets maintain desired Pod replicas.
- Deployments manage ReplicaSets.
- Kubernetes uses reconciliation to move current state toward desired state.
- Deployments support RollingUpdate and Recreate.
- `maxSurge` controls temporary extra Pods.
- `maxUnavailable` controls unavailable replicas during rollout.
- Deployments maintain rollout history.
- Deployments can be rolled back.
- Readiness probes are important during application rollouts.
- Failed rollouts should be investigated using Deployment, ReplicaSet, Pod, event, and log information.
- The same Deployment model works on Amazon EKS.

The key hierarchy:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Container
```

---

# Quick Revision Cheat Sheet

| Concept | Remember |
|---|---|
| ReplicaSet | Maintains desired number of matching Pods |
| Deployment | Manages ReplicaSets and application releases |
| Desired state | What you declare Kubernetes should maintain |
| Current state | What currently exists |
| Reconciliation | Moves current state toward desired state |
| RollingUpdate | Gradually replaces old Pods |
| Recreate | Removes old Pods before creating new ones |
| maxSurge | Extra Pods allowed during rollout |
| maxUnavailable | Pods allowed to be unavailable during rollout |
| Revision | Version of a Deployment rollout |
| Rollout status | Shows update progress |
| Rollback | Returns to a previous revision |
| Readiness probe | Helps determine traffic eligibility |
| ECR | AWS container image registry |
| EKS | AWS managed Kubernetes service |

---

# Homework

## Beginner

1. Create a ReplicaSet with three nginx Pods.
2. Delete one Pod.
3. Observe the replacement.
4. Scale the ReplicaSet to five.
5. Scale it back to two.
6. Explain desired vs current state.

## Intermediate

1. Create a Deployment with three replicas.
2. Perform an image update.
3. Watch the rolling update.
4. Inspect ReplicaSets.
5. View rollout history.
6. Roll back to the previous revision.
7. Configure `maxSurge` and `maxUnavailable`.

## Advanced

Build an EKS release workflow:

```text
Application
    ↓
Docker
    ↓
Amazon ECR
    ↓
EKS Deployment
    ↓
ReplicaSet
    ↓
Pods
    ↓
Service
```

Requirements:

- Build v1 Docker image.
- Push v1 to ECR.
- Deploy three replicas.
- Add readiness probe.
- Release v2.
- Observe rolling update.
- Inspect ReplicaSets.
- Intentionally deploy an invalid image.
- Detect failed rollout.
- Roll back to the previous version.
- Document the commands and observations.

---

# Module 11 Complete

## Next Module — Module 12: Service Discovery and Load Balancing

Topics:

- Kubernetes Service discovery
- ClusterIP in depth
- Service DNS
- CoreDNS
- EndpointSlices
- Service load balancing
- kube-proxy
- Internal vs external traffic
- NodePort
- LoadBalancer
- Session affinity
- Headless Services
- AWS Load Balancer integration
- Practical service-discovery labs
- Troubleshooting
- Mini project
- Interview questions
