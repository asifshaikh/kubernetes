# Kubernetes Deployment kubectl Commands

This reference uses the Deployment name `nginx-deployment` and the manifest `deployment.yaml`.

## Create and Apply

### Create a Deployment from YAML

```powershell
kubectl create -f deployment.yaml
```

Creates the resources described in the file. It fails if the resource already exists.

### Apply a Deployment Manifest

```powershell
kubectl apply -f deployment.yaml
```

Creates the Deployment if it does not exist, or updates it if it already exists. This is the preferred declarative workflow.

### Apply from a Directory

```powershell
kubectl apply -f .
```

Applies supported manifest files in the current directory.

### Validate a Manifest Without Changing the Cluster

```powershell
kubectl apply --dry-run=client -f deployment.yaml
```

Checks whether the manifest can be processed locally.

### Preview the Server-Side Result

```powershell
kubectl apply --dry-run=server -f deployment.yaml
```

Sends the request to the API server for validation without saving changes.

### Explain Deployment Fields

```powershell
kubectl explain deployment
kubectl explain deployment.spec
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.template.spec.containers
```

Displays Kubernetes documentation for Deployment fields.

## View Deployments

### List Deployments

```powershell
kubectl get deployments
```

Shows Deployments in the current namespace.

### List Deployments in All Namespaces

```powershell
kubectl get deployments --all-namespaces
```

Shows Deployments across every namespace.

### Get One Deployment

```powershell
kubectl get deployment nginx-deployment
```

Shows the current, desired, available, and ready replica counts.

### Watch a Deployment

```powershell
kubectl get deployment nginx-deployment --watch
```

Continuously displays changes until stopped with `Ctrl+C`.

### Show Labels

```powershell
kubectl get deployments --show-labels
```

Displays labels attached to each Deployment.

### Filter by Label

```powershell
kubectl get deployments -l app=nginx
```

Shows only Deployments matching the label selector.

### Show Deployment as YAML

```powershell
kubectl get deployment nginx-deployment -o yaml
```

Displays the complete object returned by the API server, including status and metadata.

### Show Deployment as JSON

```powershell
kubectl get deployment nginx-deployment -o json
```

Displays the object in JSON format, useful for scripts and tools.

### Show a Wide Summary

```powershell
kubectl get deployment nginx-deployment -o wide
```

Displays additional columns when supported by the resource.

## Inspect and Troubleshoot

### Describe a Deployment

```powershell
kubectl describe deployment nginx-deployment
```

Shows configuration, ReplicaSets, conditions, events, and rollout information.

### List ReplicaSets

```powershell
kubectl get replicasets
```

Shows the ReplicaSets managed by Deployments.

### List Pods Owned by the Deployment

```powershell
kubectl get pods -l app=nginx
```

Lists Pods selected by the Deployment's Pod labels.

### Show Pods with Node Details

```powershell
kubectl get pods -l app=nginx -o wide
```

Shows Pod IP addresses and the nodes where Pods are running.

### Describe a Pod

```powershell
kubectl describe pod <pod-name>
```

Shows Pod state, scheduling details, container state, probes, and events.

### View Container Logs

```powershell
kubectl logs <pod-name>
```

Displays logs from the Pod's container.

### Follow Container Logs

```powershell
kubectl logs -f <pod-name>
```

Streams new log output until stopped with `Ctrl+C`.

### View Logs from the Previous Container

```powershell
kubectl logs <pod-name> --previous
```

Shows logs from the previous container instance after a restart.

### View Recent Events

```powershell
kubectl get events --sort-by=.lastTimestamp
```

Displays recent cluster events in time order.

## Scale Deployments

### Scale to a Specific Number of Pods

```powershell
kubectl scale deployment/nginx-deployment --replicas=5
```

Changes the Deployment's desired replica count to five.

### Scale Using a Resource and Name

```powershell
kubectl scale deployment nginx-deployment --replicas=3
```

Equivalent syntax using the resource type and resource name separately.

### Confirm Scaling

```powershell
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
```

Checks the desired and available replicas and lists the Pods.

### Scale with a Minimum Condition

```powershell
kubectl scale deployment nginx-deployment --replicas=5 --current-replicas=3
```

Scales only if the current replica count is three. This helps avoid overwriting a change made by another operator.

> Important: If the manifest still contains `replicas: 3`, a later `kubectl apply -f deployment.yaml` can change the Deployment back to three Pods. Update the YAML when the new count should be permanent.

## Update the Application

### Update the Container Image

```powershell
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
```

Changes the `nginx` container image and starts a rollout.

### Update an Image from YAML

```powershell
kubectl apply -f deployment.yaml
```

Use this after changing the image in the manifest. This keeps the configuration in source control.

### Change an Environment Variable

```powershell
kubectl set env deployment/nginx-deployment APP_ENV=production
```

Adds or changes an environment variable and starts a rollout.

### Remove an Environment Variable

```powershell
kubectl set env deployment/nginx-deployment APP_ENV-
```

Removes the environment variable from the Pod template.

### Add or Change a Label

```powershell
kubectl label deployment nginx-deployment team=platform --overwrite
```

Adds or updates a label on the Deployment object.

### Annotate a Deployment

```powershell
kubectl annotate deployment nginx-deployment description="Nginx web deployment" --overwrite
```

Adds or updates metadata that can document the resource or support automation.

### Edit a Deployment Interactively

```powershell
kubectl edit deployment nginx-deployment
```

Opens the live Deployment in an editor and applies the changes when saved. Prefer version-controlled YAML for repeatable changes.

## Rollouts

### Check Rollout Status

```powershell
kubectl rollout status deployment/nginx-deployment
```

Waits for the current rollout to complete and reports its status.

### Watch Rollout Status

```powershell
kubectl rollout status deployment/nginx-deployment --watch
```

Waits and watches until the rollout finishes.

### View Rollout History

```powershell
kubectl rollout history deployment/nginx-deployment
```

Lists revisions recorded for the Deployment.

### View a Specific Revision

```powershell
kubectl rollout history deployment/nginx-deployment --revision=2
```

Displays the change details recorded for revision 2.

### Pause a Rollout

```powershell
kubectl rollout pause deployment/nginx-deployment
```

Pauses new rollouts so multiple changes can be grouped together.

### Resume a Rollout

```powershell
kubectl rollout resume deployment/nginx-deployment
```

Resumes a paused rollout.

### Roll Back to the Previous Revision

```powershell
kubectl rollout undo deployment/nginx-deployment
```

Restores the previous Deployment revision.

### Roll Back to a Specific Revision

```powershell
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

Restores the selected revision.

### Confirm a Rollback

```powershell
kubectl rollout status deployment/nginx-deployment
kubectl get deployment nginx-deployment -o wide
```

Checks that the rollback completed and shows the current image and replica information.

## Restart and Replace Pods

### Restart a Deployment

```powershell
kubectl rollout restart deployment/nginx-deployment
```

Restarts all Pods by updating the Pod template timestamp. This performs a rolling restart.

### Delete One Pod

```powershell
kubectl delete pod <pod-name>
```

Deletes one Pod. The Deployment creates a replacement to maintain the desired replica count.

### Delete Pods by Label

```powershell
kubectl delete pods -l app=nginx
```

Deletes all matching Pods. The Deployment recreates them.

## Delete Resources

### Delete the Deployment from YAML

```powershell
kubectl delete -f deployment.yaml
```

Deletes the resources defined in the manifest.

### Delete by Resource Name

```powershell
kubectl delete deployment nginx-deployment
```

Deletes the Deployment, its ReplicaSet, and normally its managed Pods.

### Delete and Wait for Completion

```powershell
kubectl delete deployment nginx-deployment --wait=true
```

Waits for dependent resources to be deleted before returning.

## Namespace and Context Options

### Run a Command in a Specific Namespace

```powershell
kubectl get deployments -n production
kubectl apply -f deployment.yaml -n production
kubectl scale deployment/nginx-deployment --replicas=5 -n production
```

Targets the `production` namespace instead of the current namespace.

### Use a Specific Kubernetes Context

```powershell
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <context-name>
```

Displays or changes the cluster and user context used by `kubectl`.

### Specify a Context for One Command

```powershell
kubectl get deployments --context=<context-name>
```

Uses a context without changing the current default context.

## Useful Command Options

### Show Client and Server Versions

```powershell
kubectl version
```

Displays Kubernetes client and server versions.

### Check the Current Cluster

```powershell
kubectl cluster-info
```

Shows control-plane and service endpoint information.

### Get Help

```powershell
kubectl help
kubectl get --help
kubectl rollout --help
kubectl scale --help
```

Shows command syntax, options, and examples.

## Recommended Deployment Workflow

```powershell
# Validate the manifest
kubectl apply --dry-run=client -f deployment.yaml

# Apply the desired configuration
kubectl apply -f deployment.yaml

# Watch the rollout
kubectl rollout status deployment/nginx-deployment

# Inspect the Deployment and Pods
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx -o wide

# Scale when needed
kubectl scale deployment/nginx-deployment --replicas=5

# Verify the new replica count
kubectl get deployment nginx-deployment
```

## Quick Command Summary

| Goal                       | Command                                                          |
| -------------------------- | ---------------------------------------------------------------- |
| Create or update from YAML | `kubectl apply -f deployment.yaml`                               |
| Inspect a Deployment       | `kubectl describe deployment nginx-deployment`                   |
| List Pods                  | `kubectl get pods -l app=nginx`                                  |
| Scale Pods                 | `kubectl scale deployment/nginx-deployment --replicas=5`         |
| Update image               | `kubectl set image deployment/nginx-deployment nginx=nginx:1.25` |
| Check rollout              | `kubectl rollout status deployment/nginx-deployment`             |
| View history               | `kubectl rollout history deployment/nginx-deployment`            |
| Roll back                  | `kubectl rollout undo deployment/nginx-deployment`               |
| Restart Pods               | `kubectl rollout restart deployment/nginx-deployment`            |
| Delete Deployment          | `kubectl delete deployment nginx-deployment`                     |
