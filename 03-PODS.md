# Kubernetes Pods

## What is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that need to run together.

Kubernetes doesn't manage containers directly — it always manages them through Pods. Even a single container runs inside a Pod.

---

## What containers in a Pod share

- **Network**: Same IP address, containers talk via `localhost`
- **Storage**: Can mount the same volumes
- **Lifecycle**: Start and stop together

---

## Single vs Multi-Container Pods

**Single-container (most common):**
Pod → Container: my-app
Used for standalone services — an API, a web server, a worker.

**Multi-container (sidecar pattern):**
Pod → Container: main-app
→ Container: log-shipper (helper)
Used only when containers are tightly coupled and must scale together (e.g., a logging agent alongside your app).

---

## Pod Lifecycle
Pending → Running → Succeeded / Failed

- **Pending**: Scheduled, but containers aren't running yet (pulling image, waiting for resources)
- **Running**: At least one container is running
- **Succeeded**: All containers finished successfully
- **Failed**: A container exited with an error

---

## Basic Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-app-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

| Field | Meaning |
|---|---|
| `metadata.name` | Name of the pod |
| `metadata.labels` | Tags used by Services/Deployments to find this pod |
| `spec.containers` | List of containers in this pod |
| `image` | Container image to run |
| `containerPort` | Port the container listens on |

---

## Pod with Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
    - name: my-app-container
      image: my-app:latest
      env:
        - name: APP_ENV
          value: "production"
```

---

## Pod with Resource Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
    - name: my-app-container
      image: nginx:latest
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

- `requests` = minimum guaranteed resources (used for scheduling)
- `limits` = maximum the container can use
- `cpu: "250m"` = 0.25 of a CPU core

---

## Multi-Container Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
    - name: main-app
      image: my-app:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: log-shipper
      image: log-shipper:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
```

Both containers share the `shared-logs` volume — `main-app` writes logs, `log-shipper` reads and forwards them.

---

## Important: Pods Are Ephemeral

Pods don't self-heal or scale on their own. If a Pod dies, it's gone — nothing recreates it automatically.

This is why in practice, you rarely create bare Pods. You use a **Deployment**, which manages Pods for you — restarting, scaling, and updating them. Bare Pods are mainly used for quick testing.

---

## Common kubectl Commands

```bash
# Create a pod
kubectl apply -f pod.yaml

# List pods
kubectl get pods

# List pods with node/IP info
kubectl get pods -o wide

# Describe a pod (see events, errors)
kubectl describe pod my-app-pod

# View logs
kubectl logs my-app-pod

# View logs of a specific container (multi-container pod)
kubectl logs my-app-pod -c main-app

# Execute a command inside a running pod
kubectl exec -it my-app-pod -- /bin/bash

# Delete a pod
kubectl delete pod my-app-pod
```

---

## Common Pod States

| Status | Meaning |
|---|---|
| `Pending` | Not scheduled yet — resource or image issue |
| `Running` | Working normally |
| `CrashLoopBackOff` | Container keeps crashing — check logs |
| `ImagePullBackOff` | Can't pull the image — check image name/registry auth |
| `Completed` | Container ran and exited successfully |

---

## Key Takeaways

- A Pod is the smallest unit Kubernetes schedules — never a raw container
- Containers in the same Pod share network and storage
- Pods are ephemeral — use Deployments to manage them in practice
- `kubectl describe` and `kubectl logs` are your first debugging steps
