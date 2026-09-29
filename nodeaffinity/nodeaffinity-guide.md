# Kubernetes Node Affinity, Taints, and Tolerations

## What Is Node Affinity?

Node affinity is a scheduling rule in a Pod's specification that tells Kubernetes which worker nodes are suitable for that Pod. It matches node labels: a node is a Kubernetes machine with labels such as `kubernetes.io/os=linux` or labels that you add, such as `disktype=ssd`.

When a Pod is created, the Kubernetes scheduler checks its scheduling rules along with available resources and other constraints. If no eligible node satisfies a required node-affinity rule, the Pod remains Pending until a suitable node becomes available or the rule changes.

Node affinity is commonly set directly on a Pod or in a workload's Pod template, such as a Deployment, StatefulSet, Job, or DaemonSet. For example, in a Deployment, it belongs under `spec.template.spec.affinity`.

## Why Use Node Affinity?

Node affinity is useful when workloads have specific placement needs, for example:

- Run storage-heavy work on nodes with SSDs.
- Run a workload in a particular region, zone, or hardware pool.
- Keep specialized workloads on nodes with GPUs or other required devices.
- Prefer certain nodes for cost or performance while retaining fallback nodes.

Node affinity selects nodes by labels. It does not reserve resources, guarantee that a node has spare capacity, or permit a Pod to bypass a node taint. Kubernetes still checks resources and other scheduling constraints.

## Label Nodes

Node affinity can only match labels that exist on nodes. Inspect labels with:

```bash
kubectl get nodes --show-labels
kubectl describe node <node-name>
```

Add a label to a node:

```bash
kubectl label node <node-name> disktype=ssd
```

Change an existing label by adding `--overwrite`:

```bash
kubectl label node <node-name> disktype=ssd --overwrite
```

Remove a label by adding a trailing hyphen to its key:

```bash
kubectl label node <node-name> disktype-
```

Use labels you control for workload placement. Kubernetes also supplies standard node labels, including labels for operating system, architecture, region, and zone. Check the labels on your cluster before relying on a particular label or value.

## Two Node-Affinity Modes

Node affinity has two main rule types. Both are set under `spec.affinity.nodeAffinity`.

### Required: `requiredDuringSchedulingIgnoredDuringExecution`

This is a hard requirement. The scheduler must find a node that matches it before the Pod can be placed. If there is no match, the Pod stays Pending.

The `IgnoredDuringExecution` part means that if the node's labels later change so it no longer matches, Kubernetes does not evict the already-running Pod just because of that label change. The rule is enforced when scheduling, not continuously after placement.

Example: only schedule on nodes labeled `disktype=ssd`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-on-ssd
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: redis
      image: redis:7
```

This is the rule used in the neighboring `affinity.yaml` example. Label at least one schedulable node with `disktype=ssd` before creating the Pod, or it will remain Pending.

### Preferred: `preferredDuringSchedulingIgnoredDuringExecution`

This is a preference, not a requirement. The scheduler tries to place the Pod on a matching node, but can use another eligible node if necessary. Each preference has a `weight` from 1 to 100; a larger weight makes that preference more influential relative to other preferences.

Example: prefer SSD nodes, but allow other nodes if needed:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-prefers-ssd
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: redis
      image: redis:7
```

The `IgnoredDuringExecution` behavior is the same as for the required rule: changing a node's labels later does not cause the running Pod to be evicted by this rule.

## Node-Affinity Operators

An expression compares a node label's key and, depending on the operator, its value or existence:

| Operator       | Meaning                                                                         | Example                    |
| -------------- | ------------------------------------------------------------------------------- | -------------------------- |
| `In`           | The key exists and its value is one of the listed values.                       | `disktype In [ssd, nvme]`  |
| `NotIn`        | The key is absent, or its value is not one of the listed values.                | `disktype NotIn [hdd]`     |
| `Exists`       | The key exists; its value is not checked.                                       | `gpu Exists`               |
| `DoesNotExist` | The key must not exist.                                                         | `maintenance DoesNotExist` |
| `Gt`           | The key exists and its integer value is greater than the single listed integer. | `cpu-count Gt 8`           |
| `Lt`           | The key exists and its integer value is less than the single listed integer.    | `cpu-count Lt 16`          |

`Gt` and `Lt` require a single integer value. For `In` and `NotIn`, supply one or more values. `Exists` and `DoesNotExist` do not take values.

Example using more than one expression:

```yaml
matchExpressions:
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  - key: disktype
    operator: In
    values: [ssd, nvme]
  - key: maintenance
    operator: DoesNotExist
```

All expressions in this one `matchExpressions` list must match the same node.

## How Terms and Expressions Combine

The structure of the required rule has important logic:

- Multiple expressions inside one `matchExpressions` list are ANDed: every expression must match.
- Multiple `nodeSelectorTerms` are ORed: a node can match any one term.
- In a preferred rule, each preference is evaluated and contributes according to its weight; preferences are not hard requirements.

For example, this permits either SSD nodes in zone `east-1a` **or** NVMe nodes in zone `east-1b`:

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
    - matchExpressions:
        - key: disktype
          operator: In
          values: [ssd]
        - key: topology.kubernetes.io/zone
          operator: In
          values: [east-1a]
    - matchExpressions:
        - key: disktype
          operator: In
          values: [nvme]
        - key: topology.kubernetes.io/zone
          operator: In
          values: [east-1b]
```

If both required and preferred rules are specified, the node must satisfy the required rule; the scheduler then uses the preferences when choosing among eligible nodes.

## What Are Taints and Tolerations?

Node affinity is a Pod-side way to say, "I want to run on nodes with these labels." Taints and tolerations provide a complementary control:

- A **taint** is applied to a node and marks it as unsuitable for Pods that do not tolerate it.
- A **toleration** is added to a Pod and says it is allowed to run on a node with a matching taint.

A toleration does **not** select or attract a Pod to a node. It only removes the taint as a reason to reject that node. Use node affinity or a node selector as well when a Pod should target a particular labeled node or group of nodes.

## Taint a Node

The general command format is:

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>
```

For example, reserve a node for database workloads:

```bash
kubectl taint nodes worker-01 dedicated=database:NoSchedule
```

View a node's taints with:

```bash
kubectl describe node worker-01
```

Remove this taint by adding a hyphen to its effect:

```bash
kubectl taint nodes worker-01 dedicated=database:NoSchedule-
```

Apply taints carefully: they can prevent ordinary workloads from scheduling on a node.

## Taint Effects

| Effect             | What Kubernetes does                                                                                                                                                                        |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NoSchedule`       | Does not schedule new Pods onto the node unless they tolerate the taint. Existing Pods are not evicted just because this taint is added.                                                    |
| `PreferNoSchedule` | Tries to avoid placing Pods without a matching toleration on the node, but may do so if needed.                                                                                             |
| `NoExecute`        | Does not schedule new Pods without a matching toleration and evicts existing Pods that do not tolerate the taint. A toleration can set `tolerationSeconds` to limit how long a Pod remains. |

## Add a Toleration to a Pod

A toleration belongs in the Pod specification. For a Deployment, put it under `spec.template.spec`:

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

This Pod tolerates the `dedicated=database:NoSchedule` taint. It is allowed onto a matching tainted node, but it is not guaranteed to land there; without an additional selection rule it may run on another eligible node.

To require that placement as well, label the intended node and combine the toleration with required node affinity:

```bash
kubectl label node worker-01 workload=database
```

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: workload
                operator: In
                values: [database]
  tolerations:
    - key: dedicated
      operator: Equal
      value: database
      effect: NoSchedule
  containers:
    - name: database
      image: postgres:16
```

Here, affinity selects nodes labeled `workload=database`, while the toleration permits the Pod to run despite the matching taint. Other scheduling requirements, such as sufficient CPU and memory, still apply.

## Toleration Operators

- `Equal` (the default) matches when the taint key, value, and effect match the toleration.
- `Exists` matches a taint with the specified key and effect regardless of its value. Do not set `value` when using `Exists`.

For example, tolerate a control-plane `NoSchedule` taint with any value:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

An empty `key` with `operator: Exists` matches all taint keys for the specified effect. This is broad access and should only be used when the workload genuinely needs it, such as some cluster-level agents.

## `NoExecute` and `tolerationSeconds`

With a `NoExecute` taint, a matching toleration can optionally limit how long a Pod stays on the node after the taint is applied:

```yaml
tolerations:
  - key: maintenance
    operator: Equal
    value: planned
    effect: NoExecute
    tolerationSeconds: 300
```

This Pod tolerates the matching taint for 300 seconds. If the taint remains after that period, Kubernetes evicts the Pod. If `tolerationSeconds` is omitted, a matching `NoExecute` toleration has no time limit.

## Putting the Concepts Together

| Mechanism     | Attached to | Purpose                                                                       |
| ------------- | ----------- | ----------------------------------------------------------------------------- |
| Node label    | Node        | Describes a node, such as its disk type or workload pool.                     |
| Node affinity | Pod         | Selects or prefers nodes based on their labels.                               |
| Taint         | Node        | Repels Pods that do not have a matching toleration.                           |
| Toleration    | Pod         | Permits a Pod to use a node with a matching taint; does not select that node. |

A useful way to reserve a node group for one kind of work is to label and taint those nodes, then give the intended Pods both matching node affinity and a toleration. Affinity directs the Pods to the group, and the taint discourages unrelated Pods from using it.

## Troubleshooting

Check where a Pod was placed and its scheduling events:

```bash
kubectl get pod redis -o wide
kubectl describe pod redis
```

Check node labels and taints:

```bash
kubectl get nodes --show-labels
kubectl describe node <node-name>
```

Common causes of a Pod remaining Pending include:

- A required label is missing from all nodes, or its value does not match the rule.
- The expressions in a single required term cannot all match the same node.
- A node has a taint the Pod does not tolerate.
- Nodes that match the rules do not have enough available resources or are otherwise ineligible.

Use `kubectl describe pod <pod-name>` and inspect the Events section for the scheduler's reason.
