# Kubernetes Deployments

## What is a Deployment?

A **Deployment** is a Kubernetes object that manages Pods and ensures your application stays in the desired state.

Instead of creating Pods directly, you typically create a Deployment. The Deployment automatically creates and manages a ReplicaSet, which in turn manages the Pods.

If a Pod crashes, the Deployment recreates it. If you update your application, the Deployment performs a rolling update with minimal downtime. If you need more capacity, you simply increase the number of replicas.

**In production, you almost always deploy applications using Deployments rather than creating bare Pods.**

---

## Why Do Deployments Exist?

Pods are **ephemeral**, meaning they are temporary.

If you create a Pod directly:

* If the Pod crashes, it stays crashed.
* If the node fails, the Pod is lost.
* Scaling requires manually creating more Pods.
* Updating the application requires deleting and recreating Pods.

A Deployment solves these problems automatically.

It provides:

* Self-healing
* Scaling
* Rolling updates
* Rollbacks
* Declarative management

Instead of manually managing Pods, you declare the desired state and Kubernetes continuously maintains it.

---

## How Deployments Work

A Deployment does **not** manage Pods directly.

The relationship is:

```text
Deployment
      │
      ▼
ReplicaSet
      │
      ▼
Pods
```

* **Deployment** defines how your application should run.
* **ReplicaSet** ensures the correct number of Pods are running.
* **Pods** run your application containers.

---

## Deployment Architecture

```text
                   Deployment
                        │
                        ▼
                 ReplicaSet
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
      Pod-1           Pod-2           Pod-3
        │               │               │
    nginx:1.27      nginx:1.27      nginx:1.27
```

If **Pod-2** crashes:

```text
ReplicaSet detects only 2 Pods running
              │
              ▼
Creates a new Pod automatically
              │
              ▼
Back to 3 healthy Pods
```

---

# Desired State

A Deployment continuously ensures reality matches your configuration.

Suppose your YAML says:

```yaml
replicas: 3
```

Current state:

```text
Pod-1 ✅
Pod-2 ❌
Pod-3 ✅
```

Desired state:

```text
3 Pods Running
```

ReplicaSet notices only two Pods are healthy and immediately creates another Pod.

This process is called **reconciliation**.

---

# Basic Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

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

---

## Understanding the YAML

| Field                      | Purpose                                                 |
| -------------------------- | ------------------------------------------------------- |
| `apiVersion`               | API version of the Deployment object                    |
| `kind`                     | Kubernetes object type                                  |
| `metadata.name`            | Name of the Deployment                                  |
| `spec.replicas`            | Number of Pod replicas to maintain                      |
| `selector.matchLabels`     | Labels used to identify Pods managed by this Deployment |
| `template`                 | Blueprint used to create Pods                           |
| `template.metadata.labels` | Labels applied to created Pods                          |
| `containers`               | Containers to run inside each Pod                       |
| `image`                    | Container image                                         |
| `containerPort`            | Port exposed by the container                           |

---

# Why Must Labels Match?

Notice these two sections:

```yaml
selector:
  matchLabels:
    app: nginx
```

and

```yaml
template:
  metadata:
    labels:
      app: nginx
```

These **must match**.

The selector tells the Deployment:

> "Manage every Pod with the label `app=nginx`."

If they don't match, Kubernetes cannot associate the Pods with the Deployment, and the Deployment will fail validation.

---

# Creating a Deployment

```bash
kubectl apply -f deployment.yaml
```

View Deployments:

```bash
kubectl get deployments
```

Output:

```text
NAME                 READY   UP-TO-DATE   AVAILABLE
nginx-deployment     3/3     3            3
```

---

# Viewing ReplicaSets

Deployments automatically create ReplicaSets.

```bash
kubectl get replicasets
```

Example:

```text
NAME                          DESIRED   CURRENT   READY
nginx-deployment-74c9fd8b7    3         3         3
```

Normally, you interact with the Deployment, not the ReplicaSet.

---

# Viewing Pods

```bash
kubectl get pods
```

Example:

```text
NAME                                READY
nginx-deployment-74c9fd8b7-2w8hd     1/1
nginx-deployment-74c9fd8b7-kr7nx     1/1
nginx-deployment-74c9fd8b7-v9sdt     1/1
```

Notice the Pod names contain the ReplicaSet name.

---

# Scaling a Deployment

Increase replicas:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Result:

```text
Deployment
      │
ReplicaSet
      │
Pod-1
Pod-2
Pod-3
Pod-4
Pod-5
```

Reduce replicas:

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

Kubernetes automatically removes excess Pods.

---

# Updating an Application

Suppose your application currently uses:

```yaml
image: nginx:1.27
```

Update it to:

```yaml
image: nginx:1.28
```

Apply the updated YAML:

```bash
kubectl apply -f deployment.yaml
```

Deployment performs a **Rolling Update**.

Instead of stopping every Pod simultaneously, Kubernetes replaces them one by one.

```text
Old Old Old

↓

New Old Old

↓

New New Old

↓

New New New
```

This minimizes downtime.

---

# Rolling Updates

During a rolling update:

* New Pods are created gradually.
* Old Pods are terminated gradually.
* Traffic continues flowing to healthy Pods.
* Users experience little or no downtime.

Monitor rollout progress:

```bash
kubectl rollout status deployment/nginx-deployment
```

---

# Rollback

If the new version is broken:

```bash
kubectl rollout undo deployment/nginx-deployment
```

Deployment immediately rolls back to the previous working ReplicaSet.

View rollout history:

```bash
kubectl rollout history deployment/nginx-deployment
```

---

# Updating Images

Instead of editing YAML:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

Verify:

```bash
kubectl rollout status deployment/nginx-deployment
```

---

# Deleting a Deployment

```bash
kubectl delete deployment nginx-deployment
```

This deletes:

* Deployment
* ReplicaSet
* Managed Pods

---

# Useful kubectl Commands

```bash
# Create or update a Deployment
kubectl apply -f deployment.yaml

# View Deployments
kubectl get deployments

# View ReplicaSets
kubectl get replicasets

# View Pods
kubectl get pods

# Describe a Deployment
kubectl describe deployment nginx-deployment

# Scale a Deployment
kubectl scale deployment nginx-deployment --replicas=5

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# Roll back to the previous version
kubectl rollout undo deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment

# Delete a Deployment
kubectl delete deployment nginx-deployment
```

---

# Deployment Strategy

By default, Deployments use the **RollingUpdate** strategy.

```text
Version 1

Pod
Pod
Pod

↓

Version 2

New Pod
Old Pod
Old Pod

↓

New Pod
New Pod
Old Pod

↓

New Pod
New Pod
New Pod
```

Another strategy is:

```yaml
strategy:
  type: Recreate
```

With **Recreate**, Kubernetes deletes all existing Pods before creating new ones, causing downtime. This strategy is mainly used for applications that cannot run multiple versions simultaneously.

---

# Best Practices

* Use Deployments instead of bare Pods for long-running applications.
* Always use versioned image tags (e.g., `nginx:1.27`) instead of `latest`.
* Set appropriate resource requests and limits.
* Use readiness and liveness probes in production.
* Label resources consistently.
* Store configuration in ConfigMaps and sensitive data in Secrets.
* Keep replica counts greater than one for high availability.

---

# Common Mistakes

❌ Creating production applications as bare Pods.

❌ Using the `latest` image tag.

❌ Mismatched labels between `selector.matchLabels` and `template.metadata.labels`.

❌ Scaling Pods manually instead of scaling the Deployment.

❌ Deleting Pods to perform updates instead of updating the Deployment.

---

# Deployment vs Pod

| Feature          | Pod | Deployment |
| ---------------- | --- | ---------- |
| Self-healing     | ❌   | ✅          |
| Scaling          | ❌   | ✅          |
| Rolling updates  | ❌   | ✅          |
| Rollbacks        | ❌   | ✅          |
| Production ready | ❌   | ✅          |

---

# Real-World Example

Imagine an e-commerce website.

```text
Users
   │
   ▼
Service
   │
   ▼
Deployment
   │
ReplicaSet
   │
├── Pod (Frontend)
├── Pod (Frontend)
├── Pod (Frontend)
```

If one Pod crashes, the ReplicaSet immediately creates a replacement. During a new release, the Deployment gradually replaces old Pods with new ones without taking the website offline.

---

# Key Takeaways

* A Deployment is the standard way to run long-lived applications in Kubernetes.
* Deployments manage ReplicaSets, and ReplicaSets manage Pods.
* Deployments provide self-healing, scaling, rolling updates, and rollbacks.
* Always update and scale the Deployment—not the Pods it creates.
* For almost every production application, create a Deployment instead of a bare Pod.
