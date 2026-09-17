# Kubernetes DaemonSet Guide

## What Is a DaemonSet?

A Kubernetes `DaemonSet` ensures that a copy of a Pod runs on every node, or on every node that matches a scheduling rule.

When a new node joins the cluster, the DaemonSet creates the Pod on that node automatically. When a node is removed, its DaemonSet Pod is removed with it.

A DaemonSet is useful when a node-level service must be available across the cluster rather than when an application needs a fixed number of replicas.

Common examples include:

- Log collectors that read logs from each node.
- Monitoring agents that collect node metrics.
- Container runtime or security agents.
- Network plugins and other node networking components.
- Storage or hardware-management agents.
- Node-level proxies or service mesh components.

## Example DaemonSet

The `ds.yaml` file in this directory creates one Nginx Pod on each eligible node:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  labels:
    app: nginx
spec:
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
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply and inspect it:

```powershell
kubectl apply -f ds.yaml
kubectl get daemonsets
kubectl get pods -o wide
kubectl describe daemonset nginx-daemonset
```

The `-o wide` option displays the node on which each Pod is running. The number of Pods normally matches the number of eligible nodes.

## How a DaemonSet Is Used

A DaemonSet controller continuously compares the desired state with the actual cluster state:

1. It finds the nodes eligible for the DaemonSet.
2. It creates one Pod on each eligible node.
3. It recreates a Pod if that Pod is deleted or fails.
4. It creates a Pod on a newly added eligible node.
5. It removes Pods from nodes that are no longer eligible.

A DaemonSet does not use `spec.replicas`. The number of Pods is determined by the eligible nodes and scheduling rules.

### Running on Selected Nodes

Use `nodeSelector`, node affinity, or tolerations when the agent should run only on particular nodes.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: node-agent
          image: example/node-agent:1.0
```

This example limits the Pod to Linux nodes. Labels must already exist on the target nodes.

A DaemonSet can also tolerate a node taint when a system agent must run on tainted nodes:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

Use tolerations carefully. A toleration allows scheduling on a tainted node; it does not require the Pod to run there.

### Updating a DaemonSet

By default, a DaemonSet uses a rolling update strategy. Kubernetes replaces old Pods gradually when the Pod template changes.

```powershell
kubectl set image daemonset/nginx-daemonset nginx=nginx:1.27
kubectl rollout status daemonset/nginx-daemonset
kubectl rollout history daemonset/nginx-daemonset
kubectl rollout undo daemonset/nginx-daemonset
```

For production workloads, prefer a specific image tag instead of `latest` so that updates are predictable.

## DaemonSet and Deployment Differences

Both resources manage Pods and replace failed Pods, but they solve different placement problems:

| Feature           | DaemonSet                                                | Deployment                                                 |
| ----------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| Main goal         | One Pod per eligible node                                | A chosen number of interchangeable Pod replicas            |
| Replica count     | Determined by eligible nodes                             | Set with `spec.replicas`                                   |
| Placement         | Targets nodes across the cluster                         | Scheduler places replicas wherever resources are available |
| Typical workload  | Node agent or infrastructure service                     | Web server, API, worker, or other application              |
| New node behavior | Automatically adds a Pod                                 | Does not add a Pod just because a node was added           |
| Scaling           | Add or remove eligible nodes, or change scheduling rules | Change `spec.replicas`                                     |
| Updates           | Rolling replacement of node-level Pods                   | Rolling updates and rollbacks for application replicas     |

Use a `Deployment` when you need, for example, five API Pods for capacity and availability. Use a `DaemonSet` when every eligible node needs its own copy of a node-level service.

## DaemonSet, Deployment, ReplicaSet, and StatefulSet

These controllers have different ownership and workload models:

| Resource      | Pod model                                                   | Identity and storage                                                                    | Best suited for                                                        |
| ------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `DaemonSet`   | One Pod per eligible node                                   | Pods are tied to node coverage, not stable application identity                         | Logging, monitoring, security, networking, and node agents             |
| `Deployment`  | A desired number of interchangeable Pods                    | Pods are replaceable and normally have no stable identity                               | Stateless web applications, APIs, and workers                          |
| `ReplicaSet`  | A desired number of interchangeable Pods                    | No stable identity; maintains a matching Pod count                                      | Low-level replica management; usually managed by a Deployment          |
| `StatefulSet` | A desired number of ordered or individually identified Pods | Stable Pod names, stable network identities, and stable persistent volume relationships | Databases, brokers, clustered systems, and other stateful applications |

### ReplicaSet

A `ReplicaSet` maintains a specified number of matching Pods. For example, if its replica count is three, it tries to keep three matching Pods running.

In normal application deployments, you create a `Deployment` instead of a standalone ReplicaSet. The Deployment creates and manages ReplicaSets during updates, which enables rollout history and rollback.

### StatefulSet

A `StatefulSet` is designed for applications where each Pod needs a stable identity or persistent storage. StatefulSet Pods have predictable names such as `database-0` and `database-1`.

StatefulSets can provide:

- Stable Pod names and network identities.
- Stable storage relationships through persistent volume claims.
- Ordered creation, updates, and termination when configured for those behaviors.

A StatefulSet does not automatically make an application highly available or replicate its data correctly. The application still needs an appropriate clustering, replication, and recovery design.

## Choosing the Right Controller

Ask what relationship the Pods should have with nodes and with each other:

- **Every eligible node needs one copy:** use a `DaemonSet`.
- **The Pods are interchangeable and you need a target count:** use a `Deployment`.
- **You only need a low-level controller to maintain a count:** use a `ReplicaSet`, although a Deployment is usually preferred.
- **Each Pod needs a stable identity or persistent storage:** use a `StatefulSet`.

A Service is separate from all four controllers. A Service provides stable network access to selected Pods; it does not replace the controller that creates and maintains those Pods.

## Useful Commands

```powershell
kubectl get daemonsets
kubectl get daemonset nginx-daemonset -o yaml
kubectl get pods -l app=nginx -o wide
kubectl describe daemonset nginx-daemonset
kubectl rollout status daemonset/nginx-daemonset
kubectl delete -f ds.yaml
```

To check which nodes are available:

```powershell
kubectl get nodes --show-labels
```

## Important Notes

- A DaemonSet normally schedules one Pod per eligible node, not one Pod per cluster replica count.
- Taints, tolerations, node selectors, node affinity, resource requests, and node capacity affect eligibility and scheduling.
- A DaemonSet Pod can still be temporarily unavailable during node maintenance or an update.
- Use readiness and liveness probes for agents that need health checks.
- Set resource requests and limits so node agents do not consume all resources needed by application workloads.
- Use a Service only when the DaemonSet Pods need stable network access; node-level agents often do not need one.
