# Kubernetes Init Containers & Sidecar Containers

## What is an Init Container?

An **Init Container** is a special container that runs **before** the main application container starts.

Its job is to perform setup tasks. Once it finishes successfully, Kubernetes starts the main container.

If the init container fails, the main container **does not start**.

---

## When are Init Containers Used?

Common setup tasks include:

- Waiting for a database to become available
- Downloading configuration files
- Creating directories
- Running database migrations

Think of it as a **preparation step** before your application starts.

---

## How Init Containers Work

```text
Pod
├── Init Container (runs first)
│      ↓
└── Main Container (starts after init succeeds)
```

The init container runs **only once** during Pod startup.

---

## Basic Init Container YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:latest
      command:
        - sh
        - -c
        - |
          echo "Waiting for database..."
          sleep 5

  containers:
    - name: my-app
      image: nginx:latest
```

---

## YAML Breakdown

| Field | Meaning |
|---|---|
| `initContainers` | List of containers that run before the main application starts |
| `command` | Command the init container executes |
| `sleep 5` | Simulates waiting before starting the application |
| `containers` | The main application containers |

---

## Key Points

- Runs **before** the main container
- Runs **only once**
- Must finish successfully
- Good for setup and initialization tasks

---

# Kubernetes Sidecar Containers

## What is a Sidecar Container?

A **Sidecar Container** is a helper container that runs **alongside** the main application container.

Both containers run **at the same time** inside the same Pod.

The sidecar adds extra functionality without changing the main application.

---

## When are Sidecars Used?

Common examples include:

- Collecting and forwarding logs
- Monitoring the application
- Acting as a network proxy
- Synchronizing files

The sidecar supports the main application while it is running.

---

## How Sidecars Work

```text
Pod
├── Main Container
└── Sidecar Container
```

Both containers:

- Start together
- Run together
- Stop together

---

## Basic Sidecar YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
    - name: my-app
      image: nginx:latest
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html

    - name: logger
      image: busybox:latest
      command:
        - sh
        - -c
        - |
          while true; do
            echo "Monitoring app..."
            sleep 5
          done
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html

  volumes:
    - name: shared-data
      emptyDir: {}
```

Both containers share the `shared-data` volume. The main container can write files, and the sidecar can read or process those files.

---

## YAML Breakdown

| Field | Meaning |
|---|---|
| `containers` | List of containers in the Pod |
| `volumeMounts` | Mounts the shared volume inside each container |
| `volumes` | Shared storage used by all containers in the Pod |
| `emptyDir` | Temporary shared storage that exists while the Pod is running |

---

## Init Container vs Sidecar Container

| Init Container | Sidecar Container |
|---|---|
| Runs before the application starts | Runs alongside the application |
| Runs only once | Runs continuously |
| Used for setup tasks | Used for helper tasks |
| Must finish before the app starts | Runs as long as the Pod is running |

---

## Common kubectl Commands

```bash
# Create the Pod
kubectl apply -f pod.yaml

# List Pods
kubectl get pods

# View Pod details
kubectl describe pod app-with-init

# View logs from the init container
kubectl logs app-with-init -c wait-for-db

# View logs from the main container
kubectl logs app-with-sidecar -c my-app

# View logs from the sidecar container
kubectl logs app-with-sidecar -c logger

# Delete the Pod
kubectl delete pod app-with-init
```

---

## Key Takeaways

- **Init Containers** prepare the environment before the application starts.
- **Sidecar Containers** help the application while it is running.
- Both are part of the same Pod.
- Containers in the same Pod share the **network** and can share **storage** using volumes.
- Use **Init Containers** for startup tasks and **Sidecar Containers** for continuous helper tasks.