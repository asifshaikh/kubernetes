# Kubernetes Namespace and FQDN Guide

## What Is a Namespace?

A Kubernetes `Namespace` is a logical partition inside one Kubernetes cluster. It groups related Kubernetes objects and gives those objects a scope for names, access control, resource limits, and policies.

Think of a cluster as an office building. The building is the cluster, and namespaces are separate departments inside it. The departments share the building's infrastructure, but each department can have its own applications, users, rules, and budget.

The `namespace.yaml` file in this directory creates a namespace named `demo`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

Create it with:

```powershell
kubectl apply -f namespace.yaml
```

Verify it:

```powershell
kubectl get namespaces
kubectl get namespace demo
```

## Why Are Namespaces Needed?

Without namespaces, all namespaced resources would be placed in one shared space. Namespaces help teams separate and manage workloads in the same cluster.

For example, an online shop might use these namespaces:

```text
shop-dev       Development versions and testing data
shop-staging   Release-candidate versions and acceptance testing
shop-prod      Customer-facing production workloads
```

The same resource name can exist in different namespaces. For example, both `shop-dev` and `shop-prod` can contain a Deployment named `web`:

```text
shop-dev/web
shop-prod/web
```

The names do not conflict because each Deployment name only needs to be unique inside its own namespace.

Namespaces are commonly used to:

- Separate development, testing, staging, and production workloads.
- Give different teams ownership of their resources.
- Apply different RBAC permissions to different environments.
- Apply CPU and memory quotas to control resource consumption.
- Apply default limits and network policies to a group of workloads.
- Make commands and monitoring easier to scope.

## Where Are Namespaces Used?

Namespaces apply to most application resources, including:

- Pods
- Deployments
- ReplicaSets
- Services
- ConfigMaps
- Secrets
- Jobs and CronJobs
- Role and RoleBinding resources
- PersistentVolumeClaims

You can specify a namespace in a manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: shop-dev
```

You can also choose a namespace with `kubectl`:

```powershell
kubectl get pods -n shop-dev
kubectl get all -n shop-prod
kubectl create deployment web --image=nginx:1.25 -n shop-dev
```

To make `kubectl` use a namespace by default in the current context:

```powershell
kubectl config set-context --current --namespace=shop-dev
```

Always check the current namespace when debugging. A command that shows no Pods may simply be looking in the wrong namespace.

## Namespace Scope and Cluster Scope

Some Kubernetes resources belong to a namespace, while others belong to the entire cluster.

Namespaced resources include Pods, Deployments, and Services. The following command lists Pods only in `demo`:

```powershell
kubectl get pods -n demo
```

Cluster-scoped resources include Nodes, PersistentVolumes, StorageClasses, and Namespaces themselves. They are not contained inside `demo`:

```powershell
kubectl get nodes
kubectl get persistentvolumes
kubectl get storageclasses
kubectl get namespaces
```

A namespace is not a separate Kubernetes cluster and does not automatically provide complete security isolation. Pods may still communicate across namespaces unless NetworkPolicies restrict that traffic, and users need RBAC rules that limit what they can access.

## Advantages of Namespaces

### 1. Organization

Namespaces keep related resources together. This makes resource listings, monitoring, and troubleshooting easier.

### 2. Name reuse

Different teams can use common names such as `api`, `web`, or `database` without collisions, as long as the objects are in different namespaces.

### 3. Access control

RBAC can grant a developer access to `shop-dev` without granting access to `shop-prod`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers
  namespace: shop-dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role
subjects:
  - kind: User
    name: developer@example.com
```

### 4. Resource control

ResourceQuota and LimitRange can prevent one team or environment from consuming all cluster resources.

For example, a development namespace can be limited to 10 CPU cores and 20 GiB of memory while production receives a larger quota.

### 5. Environment separation

Development and production versions can run in the same cluster with separate configuration, permissions, quotas, and policies.

## Disadvantages and Limitations

### 1. Not complete isolation

Namespaces share the same cluster control plane and worker infrastructure. A namespace alone does not isolate network traffic, nodes, storage, or security boundaries.

Use NetworkPolicies, RBAC, Pod Security settings, and quotas when stronger separation is required.

### 2. More operational complexity

Every command, manifest, dashboard, and alert may need the correct namespace. Forgetting `-n shop-prod` can lead to confusing results or an accidental change in another environment.

### 3. Cluster-scoped resources are shared

Nodes, StorageClasses, CustomResourceDefinitions, and PersistentVolumes are not isolated by namespace. Changes to them can affect several teams.

### 4. Resource limits require configuration

Creating a namespace does not automatically enforce CPU, memory, or object-count limits. Administrators must configure ResourceQuota and LimitRange separately.

### 5. Some workloads need extra configuration

Cross-namespace communication, shared storage, monitoring, and CI/CD deployments must be designed intentionally. Splitting applications across many namespaces can make those relationships harder to understand.

### 6. A namespace is not always the right boundary

For strict security, regulatory, or failure-domain isolation, separate clusters may be more appropriate than multiple namespaces in one cluster.

## What Is an FQDN?

FQDN means **Fully Qualified Domain Name**. It is the complete DNS name of a host or service, including every required name component from the resource up to the DNS root.

For a Kubernetes Service, the usual internal FQDN is:

```text
service-name.namespace.svc.cluster-domain
```

For example, a Service named `api` in namespace `shop-prod` normally has this FQDN:

```text
api.shop-prod.svc.cluster.local
```

The parts mean:

```text
api          Service name
shop-prod    Namespace
svc          Kubernetes Service DNS zone
cluster      Cluster DNS domain
local        DNS root used by the default Kubernetes configuration
```

The cluster domain is usually `cluster.local`, but an administrator can configure a different value.

## Kubernetes Service Names and FQDNs

Kubernetes DNS allows shorter names when the caller and Service are in the same namespace:

```text
api
api.shop-prod
api.shop-prod.svc
api.shop-prod.svc.cluster.local
```

An application in `shop-prod` can usually call:

```text
http://api:8080
```

An application in `shop-dev` should use the namespace-qualified name to reach the production Service:

```text
http://api.shop-prod:8080
```

The complete FQDN is useful when the name must be unambiguous:

```text
http://api.shop-prod.svc.cluster.local:8080
```

Kubernetes DNS records are created for Services, not merely for Pods. A Service is preferred because Pod IP addresses can change when Pods are recreated.

## Real-World Example: Online Shop

Assume an online shop has a frontend, product API, and database in production:

```text
Namespace: shop-prod

frontend Service -> frontend Pods
api Service      -> API Pods
database Service -> Database Pods
```

The frontend configuration can use the API Service FQDN:

```text
http://api.shop-prod.svc.cluster.local:8080
```

The API can use the database Service FQDN:

```text
postgres://database.shop-prod.svc.cluster.local:5432/shop
```

If the team deploys a test copy to `shop-staging`, it can create another `api` and `database` Service without changing the production Services:

```text
api.shop-prod.svc.cluster.local       Production API
api.shop-staging.svc.cluster.local    Staging API
```

The same application image can therefore run in both environments while each environment gets its own namespace, configuration, permissions, and Service DNS names.

## Practical Commands

```powershell
# List namespaces
kubectl get namespaces

# List resources in one namespace
kubectl get all -n demo

# Show the namespace selected by the current context
kubectl config view --minify --output 'jsonpath={..namespace}'

# Show a Service and its endpoints
kubectl get service api -n shop-prod
kubectl get endpoints api -n shop-prod

# Run a temporary DNS lookup from inside the cluster
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup api.shop-prod.svc.cluster.local

# Delete a namespace and its namespaced resources
kubectl delete namespace demo
```

Deleting a namespace deletes the namespaced objects inside it. Treat this command as destructive, especially for production workloads.

## Summary

- A namespace is a logical boundary for organizing and controlling namespaced Kubernetes resources.
- Namespaces support name reuse, RBAC, quotas, policies, and environment separation.
- A namespace is not a separate cluster and does not automatically provide complete security isolation.
- An FQDN is a complete DNS name.
- A Kubernetes Service FQDN normally follows `service.namespace.svc.cluster.local`.
- Use namespace-qualified or fully qualified Service names when communicating across namespaces.
