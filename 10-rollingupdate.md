# Kubernetes Rolling Updates & Rollbacks

## What is a Rolling Update?

**Rolling Update** = Gradually replace old pods with new ones (zero downtime)

### How It Works

```
Pod 1 (old) → Pod 1 (new)
Pod 2 (old) → Pod 2 (new)
Pod 3 (old) → Pod 3 (new)

One at a time, users never notice
```

### Without Rolling Update (Bad)

```
Kill all 3 old pods
    ↓
Wait 2 minutes
    ↓
Start 3 new pods
    ↓
During those 2 minutes: NO PODS = NO SERVICE ❌
Users see: 503 Service Unavailable
```

### With Rolling Update (Good)

```
Kill Pod 1, start new Pod 1
    (Pod 2, Pod 3 still serving traffic) ✅
    ↓
Kill Pod 2, start new Pod 2
    (Pod 1, Pod 3 still serving traffic) ✅
    ↓
Kill Pod 3, start new Pod 3
    (Pod 1, Pod 2 still serving traffic) ✅
    ↓
All updated, zero downtime
```

---

## Update Deployment

### Change Image

```bash
# Update image to new version
kubectl set image deployment/backend backend=myapp:v2.0
```

Or edit YAML:

```yaml
spec:
  containers:
  - name: backend
    image: myapp:v2.0  # Changed from v1.0
```

Then apply:
```bash
kubectl apply -f deployment.yaml
```

---

### Watch Rolling Update

```bash
# See pods being replaced
kubectl get pods -w

# Or check rollout status
kubectl rollout status deployment/backend
```

Output:
```
Waiting for deployment "backend" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "backend" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "backend" rollout to finish: 3 out of 3 new replicas have been updated...
Waiting for deployment "backend" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "backend" rollout to finish: 0 old replicas are pending termination
deployment "backend" successfully rolled out
```

---

## Control Rolling Update Speed

### Default (One at a time)

```yaml
spec:
  replicas: 3
  # Default: maxSurge=1, maxUnavailable=1
```

Replaces 1 pod at a time.

---

### Faster (Two at a time)

```yaml
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 2          # Allow 2 extra pods during update
      maxUnavailable: 0    # Keep all pods available
```

Replaces 2 pods at a time (faster, but uses more resources).

---

### Slower (Safe)

```yaml
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1          # Only 1 extra pod
      maxUnavailable: 1    # Allow 1 pod to be down
```

Replaces 1 pod at a time (safer, uses less resources).

---

## Rollback

### Why Rollback?

```
Deploy new version
    ↓
It has a bug
    ↓
Users see errors
    ↓
Rollback to previous version (1 command)
    ↓
Users happy again
```

---

### Rollback Command

```bash
# Undo last deployment
kubectl rollout undo deployment/backend

# Back to v1.0 immediately
# Rolling update reverses: new pods replaced with old
```

---

### Check Rollout History

```bash
# See all versions
kubectl rollout history deployment/backend

Output:
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=deployment.yaml
2         kubectl apply --filename=deployment.yaml
3         kubectl apply --filename=deployment.yaml

# Revision 3 is current (latest)
```

---

### Rollback to Specific Version

```bash
# Rollback to revision 1
kubectl rollout undo deployment/backend --to-revision=1
```

---

## Real Example

### Step 1: Deploy v1.0

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myapp:v1.0
```

Deploy:
```bash
kubectl apply -f deployment.yaml
# All 3 pods running v1.0
```

---

### Step 2: Update to v2.0

```bash
kubectl set image deployment/backend backend=myapp:v2.0

# Or edit YAML
# Change image: myapp:v1.0 → myapp:v2.0
# kubectl apply -f deployment.yaml
```

Watch:
```bash
kubectl rollout status deployment/backend
# Shows: 1/3 → 2/3 → 3/3 new pods running
```

---

### Step 3: Bug in v2.0!

```bash
# Users complaining
# Check pod logs
kubectl logs backend-pod-xxx

# Confirms: v2.0 has bug
```

---

### Step 4: Rollback to v1.0

```bash
kubectl rollout undo deployment/backend

# Rolling update reverses
# v2.0 pods replaced with v1.0 pods
# Users happy again
```

---

## Readiness Probes + Rolling Updates

**Critical:** Use readiness probes with rolling updates

```yaml
spec:
  strategy:
    rollingUpdate:
      maxUnavailable: 0    # Never remove pod without replacement
  template:
    spec:
      containers:
      - name: backend
        image: myapp:v2.0
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**How it works:**

```
New pod starts
    ↓
Readiness probe: Is it ready?
    → NO (still loading)
    ↓
Service waits (traffic still on old pod)
    ↓
Readiness probe: Is it ready?
    → YES
    ↓
Service adds new pod (traffic can go here)
    ↓
Old pod terminating
    ↓
Users see zero errors
```

---

## Best Practices

### 1. Use Readiness Probes

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
```

Ensures new pods are ready before old ones are killed.

---

### 2. Set Graceful Termination

```yaml
terminationGracePeriodSeconds: 30
```

Gives pods 30s to finish requests before killing.

---

### 3. Zero Downtime Settings

```yaml
strategy:
  rollingUpdate:
    maxSurge: 1          # One extra pod during update
    maxUnavailable: 0    # Never have 0 pods
```

---

### 4. Check Logs Before Rolling Back

```bash
# Why did it fail?
kubectl logs backend-pod-xxx

# Then rollback
kubectl rollout undo deployment/backend
```

---

## Troubleshooting

### Update Stuck

```bash
# Check status
kubectl rollout status deployment/backend

# See what's wrong
kubectl describe deployment backend

# Check pod events
kubectl describe pod backend-pod-xxx

# Usually: Readiness probe failing, pod not ready
```

---

### Pod Won't Start

```bash
# Check logs
kubectl logs backend-pod-xxx

# Check if readiness probe is too strict
kubectl describe pod backend-pod-xxx

# Increase initialDelaySeconds if needed
```

---

## Quick Reference

```bash
# Update image
kubectl set image deployment/backend backend=myapp:v2.0

# Watch rollout
kubectl rollout status deployment/backend

# Check history
kubectl rollout history deployment/backend

# Rollback last update
kubectl rollout undo deployment/backend

# Rollback to specific version
kubectl rollout undo deployment/backend --to-revision=1

# Pause rollout (if needed)
kubectl rollout pause deployment/backend

# Resume rollout
kubectl rollout resume deployment/backend
```

---

## Summary

**Rolling Update = Update without downtime**

- Old pods gradually replaced with new pods
- Service keeps serving traffic
- Uses readiness probes to check pod health
- Can rollback immediately if bug

**Always:**
- ✅ Use readiness probes
- ✅ Use maxUnavailable: 0
- ✅ Test new version before deployment
- ✅ Know how to rollback

**Result:** Zero downtime deployments
