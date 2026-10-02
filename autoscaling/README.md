# Kubernetes Autoscaling Guide

## What Is Autoscaling?

Autoscaling is the automatic adjustment of computing capacity to match an application's workload. In Kubernetes, autoscaling can change:

- The number of application Pods.
- The CPU and memory resources assigned to each Pod.
- The number of Pods based on events, such as messages waiting in a queue.

For example, an online store may receive more requests during a sale than during a normal day. Instead of keeping the same number of application instances all the time, Kubernetes can add Pods when demand rises and remove extra Pods when demand falls.

## What Does Autoscaling Do?

An autoscaler observes metrics or events, compares them with a configured target, and updates a workload's desired capacity. Kubernetes controllers then work to make the actual cluster state match that desired state.

```text
Workload increases -> autoscaler detects demand -> capacity increases
Workload decreases -> autoscaler detects lower demand -> unused capacity decreases
```

Scaling takes time. Kubernetes needs to collect metrics, make a decision, schedule Pods, start containers, and wait for them to become ready. An autoscaler also cannot create cluster capacity that does not exist; the cluster must have enough available resources or a cluster-level autoscaler must add nodes.

## Where Is Autoscaling Used?

Autoscaling is useful for:

- **Web applications and APIs:** Add replicas when request volume or CPU usage increases.
- **Background workers:** Add worker Pods when a queue or stream has more pending work.
- **Batch workloads:** Add capacity while jobs are pending and reduce it when work finishes.
- **Variable or seasonal workloads:** Match capacity to predictable daily, weekly, or seasonal demand.
- **Right-sizing applications:** Adjust CPU and memory requests to better match observed use.

## Why Is Autoscaling Useful?

1. **Handles changing demand:** Responds to traffic spikes without requiring an operator to manually change replicas.
2. **Maintains application responsiveness:** Adds capacity when configured metrics or events indicate more work.
3. **Avoids paying for unnecessary capacity:** Reduces replicas or resources when demand falls.
4. **Improves resource planning:** Recommendations and observed scaling behavior help teams choose appropriate resource settings.
5. **Supports different workload types:** CPU, memory, custom metrics, and event sources can each be used as scaling signals.

Autoscaling is not a replacement for application health checks, monitoring, capacity planning, or load testing. A poorly configured maximum, unavailable nodes, slow startup, or an unsuitable metric can still prevent an application from handling demand.

## Kubernetes Autoscaling Options

| Option | What it changes | Example signal | Common use |
| --- | --- | --- | --- |
| HPA | Number of workload replicas | CPU, memory, custom, or external metrics | Add or remove application Pods |
| VPA | CPU and memory requests for Pods | Observed resource use | Right-size each Pod |
| KEDA | Number of workload replicas, including scale-to-zero for supported workloads | Queue length, stream lag, or other event sources | Scale event-driven applications and workers |

HPA and VPA do different jobs. They can be combined, but plan which controller owns each scaling decision. In particular, HPA CPU utilization is calculated relative to CPU requests, and VPA can change those requests. Using both on CPU without a deliberate design can lead to confusing or unstable behavior.

## Prerequisites

- A Kubernetes cluster with `kubectl` configured.
- For HPA using CPU or memory, a working Metrics Server. Check metrics with:

```powershell
kubectl top nodes
kubectl top pods
```

- For VPA, install the VPA components compatible with your Kubernetes version. VPA is not included in a standard Kubernetes installation. See [Kubernetes Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler).
- For KEDA, install the KEDA operator and CRDs. See the [KEDA documentation](https://keda.sh/docs/).

### Install VPA Before Applying a VPA Manifest

The `VerticalPodAutoscaler` kind is a custom Kubernetes resource. Its API is not present in a new cluster until the VPA CustomResourceDefinition (CRD) and controller components are installed. The VPA installation also requires Metrics Server and permission to create cluster-scoped resources.

Check the cluster's Kubernetes version and current context, and confirm Metrics Server is providing metrics:

```powershell
kubectl version
kubectl config current-context
kubectl top nodes
```

Choose a VPA release compatible with the cluster version using the compatibility table in the [official VPA installation guide](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/installation.md). Then, from Git Bash or WSL, install VPA by running the upstream setup script from the `vertical-pod-autoscaler` directory. For example:

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
git checkout <compatible-vpa-release-branch>
./hack/vpa-up.sh
```

Replace `<compatible-vpa-release-branch>` with the release branch selected from the compatibility table. The script installs cluster-level VPA resources and components. Make sure Git Bash/WSL's `kubectl` is configured for the same cluster and context you intend to use. Follow the upstream guide for supported versions, permissions, upgrades, and teardown.

Verify that the CRD is established and the VPA API is served:

```powershell
kubectl wait --for condition=Established crd/verticalpodautoscalers.autoscaling.k8s.io --timeout=60s
kubectl api-resources | Select-String VerticalPodAutoscaler
```

Only after the API appears should you apply a manifest containing `kind: VerticalPodAutoscaler`.

Check whether the autoscaling APIs are available:

```powershell
kubectl get apiservices | Select-String metrics
kubectl get crd | Select-String 'verticalpodautoscalers|scaledobjects'
```

If `kubectl top` cannot retrieve metrics, check that Metrics Server is installed and healthy before troubleshooting CPU-based HPA.

## Horizontal Pod Autoscaler (HPA)

### What Is HPA?

The Horizontal Pod Autoscaler adjusts the number of replicas in a Deployment, StatefulSet, or another supported scalable workload. "Horizontal" means adding or removing Pods rather than making an individual Pod larger or smaller.

For a CPU utilization target, HPA compares average CPU usage with the CPU **request** configured on the containers. For example, if a container requests `200m` CPU and uses `100m`, it is using 50% of its request. A CPU request must be set for CPU utilization-based scaling to work.

### HPA Example YAML

This example is also available as [`hpa.yaml`](./hpa.yaml). It creates a web Deployment, a Service for accessing its Pods, and an HPA that keeps the Deployment between 2 and 10 replicas with an average CPU target of 50%.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    run: php-apache
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### HPA Field-by-Field Explanation

#### Deployment

- `kind: Deployment` creates and manages the application Pods.
- `selector.matchLabels` identifies the Pods managed by the Deployment. It must match the labels in `template.metadata.labels`.
- `image` selects the sample web application container image.
- `containerPort: 80` documents the port used by the container.
- `requests.cpu: 200m` requests 0.2 CPU and provides the baseline for calculating CPU utilization.
- `limits.cpu: 500m` caps the container at 0.5 CPU.

#### Service

- `kind: Service` creates a stable in-cluster network endpoint for the Pods.
- `selector` must match the Pod labels so the Service can route requests to them.
- `port: 80` is the port clients use on the Service; `targetPort: 80` is the destination port in the container.

#### HorizontalPodAutoscaler

- `apiVersion: autoscaling/v2` selects the current stable HPA API.
- `scaleTargetRef` identifies the Deployment whose replica count HPA controls.
- `minReplicas: 2` prevents scaling below two replicas.
- `maxReplicas: 10` limits scaling to ten replicas.
- `metrics` selects CPU as the scaling signal.
- `averageUtilization: 50` targets average CPU use of 50% of the Pods' CPU requests. This is a target, not a guarantee that utilization will stay exactly at 50%.

### Apply and Test HPA

Run these commands from the `autoscaling` directory:

```powershell
kubectl apply -f hpa.yaml
kubectl get deployments,services,hpa
kubectl get hpa php-apache --watch
```

In a second PowerShell window, generate continuous requests to the Service:

```powershell
kubectl run load-generator --image=busybox:1.36 --restart=Never -it --rm -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

Leave the load generator running for a few minutes. Check the HPA's `TARGETS` and `REPLICAS` columns. The Deployment should scale up if CPU usage remains above the target and metrics are available. Press `Ctrl+C` in the load-generator window to stop the traffic, then watch the replicas move back toward the minimum.

Useful troubleshooting command:

```powershell
kubectl describe hpa php-apache
```

If the current CPU target shows `<unknown>`, check Metrics Server, container CPU requests, and `kubectl top pods`. HPA can only act on metrics it can retrieve.

Delete the resources created by this example:

```powershell
kubectl delete -f hpa.yaml
```

## Vertical Pod Autoscaler (VPA)

### What Is VPA?

The Vertical Pod Autoscaler recommends or updates CPU and memory resources for containers in Pods. "Vertical" means adjusting resources assigned to each Pod rather than increasing or decreasing the number of Pods.

VPA is useful when a workload's individual Pods are consistently requesting too much or too little CPU or memory. It can help improve scheduling and resource utilization, but applying new recommendations may require Pods to be restarted or evicted, depending on the VPA mode and cluster version.

A typical VPA installation has these components:

- **Recommender:** Observes resource use and produces resource recommendations.
- **Updater:** Finds Pods that need changes and may evict them so they can be recreated.
- **Admission controller:** Applies recommended resources as Pods are created.

### VPA Example YAML

This example is also available in [`vpa.yaml`](./vpa.yaml). It creates a separate Deployment so it can be tested independently from the HPA example.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
        - name: workload
          image: registry.k8s.io/hpa-example
          resources:
            requests:
              cpu: 100m
              memory: 50Mi
            limits:
              cpu: 500m
              memory: 200Mi
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-demo
spec:
  selector:
    app: cpu-demo
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cpu-demo
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: workload
        controlledResources: ["cpu", "memory"]
        minAllowed:
          cpu: 50m
          memory: 32Mi
        maxAllowed:
          cpu: "1"
          memory: 1Gi
```

### VPA Field-by-Field Explanation

#### Deployment

- `replicas: 2` starts two application Pods.
- `selector.matchLabels` must match the labels in the Pod template.
- `requests` specify the initial CPU and memory requested by each container.
- `limits` set the maximum CPU and memory available to each container.

#### Service

- `selector` finds the `cpu-demo` Pods.
- `port: 80` provides a stable endpoint for sending test requests to those Pods.

#### VerticalPodAutoscaler

- `apiVersion: autoscaling.k8s.io/v1` selects the VPA API version.
- `targetRef` identifies the Deployment whose Pods VPA observes.
- `updateMode: "Off"` tells VPA to calculate and report recommendations but not apply them. This is a safe way to start learning what VPA recommends.
- `containerPolicies` applies resource policy to the container named `workload`.
- `controlledResources` lists the resources VPA manages: CPU and memory.
- `minAllowed` and `maxAllowed` bound the recommendations, helping prevent requests from becoming unreasonably small or large.

### Apply and Inspect VPA

Confirm the VPA CRD is installed before applying `vpa.yaml`. If it is missing, `kubectl` reports `no matches for kind "VerticalPodAutoscaler"`; install VPA using the steps under [Install VPA Before Applying a VPA Manifest](#install-vpa-before-applying-a-vpa-manifest). In a multi-document YAML apply, earlier documents may already have been created even if applying the VPA document fails, so it is safe to reapply after installing VPA.

```powershell
kubectl apply -f .\vpa.yaml
kubectl get pods -l app=cpu-demo
```

In a second PowerShell window, send continuous requests to generate activity:

```powershell
kubectl run vpa-load-generator --image=busybox:1.36 --restart=Never -it --rm -- /bin/sh -c "while true; do wget -q -O- http://cpu-demo; done"
```

Let the workload run for a few minutes so VPA can observe resource use. Then inspect its recommendations:

```powershell
kubectl describe vpa cpu-demo
```

Look for the recommendation section. It can include `target`, `lowerBound`, and `upperBound` values for CPU and memory. Recommendations may take time to appear because VPA needs resource-use observations.

The example uses `updateMode: "Off"`, so Pods are not changed. Other modes supported by VPA versions can apply recommendations, potentially by recreating Pods. Confirm the behavior for your installed version and test the impact before enabling automatic updates in production. Consider PodDisruptionBudgets and ensure the cluster has enough capacity to reschedule Pods.

Delete the resources created by this example:

```powershell
kubectl delete -f .\vpa.yaml
```

## KEDA (Kubernetes Event-driven Autoscaling)

### What Is KEDA?

KEDA is a Kubernetes operator that scales applications based on event sources and external metrics. Examples include messages waiting in a queue, stream consumer lag, or events from cloud services.

KEDA is useful when CPU and memory do not represent demand well. For example, a queue worker might be using little CPU while messages are building up. A queue-based scaler can add workers based on the queue depth. KEDA commonly creates and uses an HPA to scale a workload, and can scale supported workloads down to zero when there is no pending work.

### Conceptual KEDA Example

This is a template, not an apply-ready manifest. The scaler `type` and `metadata` must be replaced with settings for a real event source supported by the installed KEDA version.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-worker
spec:
  scaleTargetRef:
    name: queue-worker
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: <installed-scaler>
      metadata:
        <scaler-specific-key>: <scaler-specific-value>
```

- `scaleTargetRef.name` is the name of the Deployment or other supported workload to scale.
- `minReplicaCount` and `maxReplicaCount` define the lower and upper scaling bounds.
- `minReplicaCount: 0` enables scaling to zero when supported by the event source and configuration.
- `triggers` specifies the event source and its scaler-specific settings.

Install KEDA and configure a real scaler before applying a `ScaledObject`. Store credentials in Kubernetes Secrets and reference them using the authentication method supported by the scaler. Do not put credentials directly into the manifest.

## Which Autoscaler Should I Use?

- Use **HPA** when adding more copies of an application helps handle more requests or work. CPU-based HPA needs CPU requests and Metrics Server.
- Use **VPA** when you need recommendations or automatic adjustment of CPU and memory requests for individual Pods. Begin with recommendation-only mode.
- Use **KEDA** when a workload should scale based on events or an external system, especially queue-driven workers that may need to scale to zero.

For production workloads, set realistic minimum and maximum values, monitor scaling decisions and application health, and ensure Pods start quickly enough to handle demand. Avoid configuring HPA and VPA to compete over CPU utilization and CPU requests without testing their interaction.

## Useful Commands

```powershell
kubectl get hpa
kubectl describe hpa <hpa-name>
kubectl top pods
kubectl get vpa
kubectl describe vpa <vpa-name>
kubectl get scaledobjects
```

## Further Reading

- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [KEDA documentation](https://keda.sh/docs/)
