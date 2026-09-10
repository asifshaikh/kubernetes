# Kubernetes Service Guide

## What Is a Service?

A Kubernetes `Service` provides a stable network endpoint for a group of Pods. Pods are temporary: they can be recreated, moved to another node, or receive new IP addresses. A Service gives clients a consistent name and virtual IP address even when the Pods behind it change.

A Service uses labels to discover the Pods that should receive traffic. Kubernetes then load-balances connections across the matching, healthy Pods.

For example, a Deployment might run three Nginx Pods:

```text
Client -> nginx-service:80 -> Pod 1
                         -> Pod 2
                         -> Pod 3
```

The client does not need to know the individual Pod IP addresses.

## Where Is a Service Used?

Services are used wherever one part of an application needs to communicate reliably with another part, including:

- Inside a cluster, such as a frontend calling a backend API.
- From outside the cluster, such as a user's browser accessing a web application.
- Between replicated Pods, where traffic must be distributed across multiple instances.
- With Pods whose IP addresses can change during deployments, scaling, or failures.

A common real-world application has these Services:

```text
Internet -> frontend Service -> frontend Pods
frontend Pods -> backend Service -> backend Pods
backend Pods -> database Service -> database Pods
```

The frontend calls a stable DNS name such as `backend-service`, rather than calling a changing Pod IP address.

## Why Is a Service Needed?

A Service solves several networking problems:

1. **Stable addressing:** Clients use one DNS name and virtual IP instead of changing Pod IPs.
2. **Load balancing:** Traffic is distributed among matching Pods.
3. **Service discovery:** Kubernetes DNS lets applications find Services by name.
4. **Decoupling:** Producers and consumers can change independently. A backend can be scaled or replaced without changing the frontend configuration.
5. **Controlled exposure:** The Service type determines whether the application is reachable only inside the cluster or from outside it.

A Pod declaration such as `containerPort: 80` documents the port used by a container, but it does not create a stable endpoint or expose the Pod. The Service provides that network access.

## Example Service

This is the Service in `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-deployment
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
```

Assume the Nginx Deployment creates Pods with this label:

```yaml
metadata:
  labels:
    app: nginx-deployment
```

The Service selects those Pods and forwards traffic to port `80` in them.

A request from outside the cluster can use:

```text
http://<NodeIP>:30007
```

The request path is:

```text
Client -> NodeIP:30007 -> nginx-service:80 -> selected Pod:80
```

`<NodeIP>` is the IP address of any Kubernetes worker node that can receive the request. Kubernetes forwards the request to one of the selected Nginx Pods.

## Types of Services

### `ClusterIP`

`ClusterIP` is the default Service type. It exposes the application only inside the Kubernetes cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

A frontend Pod can call:

```text
http://backend-service:80
```

**Real-world example:** A payment API should be reachable by the internal checkout application but should not be directly accessible from the public internet. `ClusterIP` is appropriate for the payment API.

### `NodePort`

`NodePort` exposes the Service on a port on every worker node. External clients connect to the node IP and the assigned node port. The Service then forwards traffic to the selected Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-deployment
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

**Real-world example:** A development team wants to test an Nginx website running in a local or small cluster. They can open `http://<NodeIP>:30007` without setting up a cloud load balancer.

NodePort is convenient for development and simple environments. In production, a `LoadBalancer` or an Ingress is usually preferred for public HTTP or HTTPS traffic.

### `LoadBalancer`

`LoadBalancer` asks the cloud provider or supported infrastructure to create an external load balancer. It also provides the normal internal Service behavior.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```

**Real-world example:** A company runs its website in a managed cloud Kubernetes cluster. A cloud load balancer receives internet traffic and forwards it to the web Pods.

The exact external IP or hostname depends on the infrastructure provider. On a local cluster, a `LoadBalancer` may remain pending unless a load-balancer implementation is installed.

### `ExternalName`

`ExternalName` maps a Kubernetes Service name to an external DNS name. It does not select Pods and does not create a proxy or load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-service
spec:
  type: ExternalName
  externalName: payments.example.com
```

**Real-world example:** An application can call `payments-service` inside the cluster while the actual payment system runs outside Kubernetes at `payments.example.com`. This can reduce the amount of application configuration that changes when the external system changes.

## Explanation of Each Field

### `apiVersion`

```yaml
apiVersion: v1
```

Specifies the Kubernetes API version used for the resource. Core resources such as Services use `v1`.

### `kind`

```yaml
kind: Service
```

Identifies the resource type. This tells Kubernetes to create a Service.

### `metadata`

```yaml
metadata:
  name: nginx-service
```

Contains identifying information about the Service.

#### `metadata.name`

```yaml
name: nginx-service
```

The name of the Service. It must be unique within its namespace. Other Pods can use this name for service discovery, for example:

```text
http://nginx-service
```

#### `metadata.namespace`

A namespace can optionally be specified:

```yaml
metadata:
  name: nginx-service
  namespace: production
```

If no namespace is written, Kubernetes uses the namespace selected by the command, usually `default`. A Service name is normally resolved within its own namespace.

#### `metadata.labels`

Labels can be added to the Service itself:

```yaml
metadata:
  name: nginx-service
  labels:
    app: nginx
    environment: production
```

These labels organize and identify the Service. They are different from `spec.selector`, which selects the backend Pods.

### `spec`

```yaml
spec:
  selector:
    app: nginx-deployment
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
```

`spec` describes the desired behavior of the Service: which Pods receive traffic, how it is exposed, and which ports are used.

### `spec.selector`

```yaml
selector:
  app: nginx-deployment
```

Selects Pods whose labels match the key-value pair. Only matching Pods become Service endpoints.

The selector must match the labels on the Pods, not merely the labels on a Deployment object. Check the selected Pods with:

```powershell
kubectl get pods --show-labels
kubectl get endpoints nginx-service
```

If the selector does not match any Pod, the Service can exist but traffic has nowhere to go.

### `spec.type`

```yaml
type: NodePort
```

Controls how the Service is exposed. The main choices are `ClusterIP`, `NodePort`, `LoadBalancer`, and `ExternalName`.

If `type` is omitted, the default is `ClusterIP`.

### `spec.ports`

```yaml
ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30007
```

`ports` is a list, so one Service can expose more than one port, such as HTTP and HTTPS.

### `protocol`

```yaml
protocol: TCP
```

Specifies the network protocol. `TCP` is the default and is used by HTTP, HTTPS, and most web applications. Kubernetes also supports `UDP` and `SCTP` where appropriate.

### `port`

```yaml
port: 80
```

The port exposed by the Service. Other Pods connect to this port through the Service's DNS name or ClusterIP.

In this example, an internal client uses:

```text
http://nginx-service:80
```

`port` is the Service-facing port. It does not have to be the same number as the application's container port.

### `targetPort`

```yaml
targetPort: 80
```

The port on the selected Pod that receives the traffic. Here, the Service forwards traffic to port `80` in the Nginx container.

The values can be different:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

In this case, clients use the Service on port `80`, while the application listens on port `8080` inside each Pod.

`targetPort` can also refer to a named container port:

```yaml
ports:
  - port: 80
    targetPort: http
```

### `nodePort`

```yaml
nodePort: 30007
```

The port opened on every worker node when the Service type is `NodePort`. External clients use:

```text
http://<NodeIP>:30007
```

The usual Kubernetes NodePort range is `30000-32767`, unless the cluster has been configured differently. If `nodePort` is omitted for a `NodePort` Service, Kubernetes assigns an available port automatically.

`nodePort` is not the port used by the application inside the Pod. The complete mapping in this example is:

```text
NodeIP:30007 -> Service port 80 -> Pod target port 80
```

## Port Comparison

| Field        | Where it is used  | Value in this example | Meaning                             |
| ------------ | ----------------- | --------------------: | ----------------------------------- |
| `port`       | Service           |                  `80` | Port clients use on `nginx-service` |
| `targetPort` | Selected Pod      |                  `80` | Port where Nginx listens            |
| `nodePort`   | Every worker node |               `30007` | Port external clients use on a node |

A useful analogy is a company reception desk:

- `nodePort` is the building's public entrance number.
- `port` is the department extension at the reception desk.
- `targetPort` is the employee's desk where the request is finally delivered.

## Create and Inspect the Service

Apply the manifest:

```powershell
kubectl apply -f service.yaml
```

View the Service:

```powershell
kubectl get service nginx-service
kubectl describe service nginx-service
```

Verify the backend Pods and endpoints:

```powershell
kubectl get pods -l app=nginx-deployment
kubectl get endpoints nginx-service
```

The endpoints should contain the IP addresses and port `80` of the selected Pods. If no endpoints appear, first check that the Pod labels match the Service selector.

Delete the Service with:

```powershell
kubectl delete -f service.yaml
```

## Summary

A Service is the stable networking layer in front of changing Pods. Its selector finds the backend Pods, `port` defines the port exposed by the Service, `targetPort` identifies the application port inside those Pods, and `nodePort` provides an external node-level port for a `NodePort` Service.
