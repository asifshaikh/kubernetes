### Pods can be created in 2 ways

- Imperative way -> writing `kubectl run nginx-pod --image=nginx:latest`
- Declarative way -> creating a yaml configuration file

#### [Pod YAML Guide](./pod-yaml-guide.md)

#### Useful Pod commands

- `kubectl get pods` -> get pods
- `kubectl get pods -o wide` -> extended version of get pods
- `kubectl describe pod <name-of-pod>` -> describe the pod
- `kubectl edit pod <name-of-pod>` -> edit pod configuration directly
- `kubectl exec -it <name-of-pod> -- sh` -> exec inside the pod
- `kubectl run nginx --image=nginx:latest --port=80 --dry-run=client -o yaml > pod.yaml` ->
  - `kubectl run nginx`: Defines a Pod named nginx.
  - `--image=nginx:latest`: Specifies the container image.
  - `--port=80`: Sets the container port.
  - `--dry-run=client`: Generates YAML without creating the Pod.
  - `-o yaml`: Outputs the configuration as YAML.
  - `> pod.yaml`: Saves the YAML to pod.yaml.
