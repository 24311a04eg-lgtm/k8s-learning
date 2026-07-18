# Kubernetes Health Probes - Complete Guide

## What Are Health Probes?

**Health Probes** = Kubernetes checks if your pod is healthy

Kubernetes periodically asks: "Is your app working?"
- If YES → Keep pod running
- If NO → Restart pod / Remove from traffic

---

## The 3 Types of Probes

### 1. Liveness Probe

**Question:** Is the app alive (not stuck/crashed)?

**If fails:** Pod is restarted

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 10    # Wait 10s before first check
      periodSeconds: 10          # Check every 10s
      failureThreshold: 3        # Restart after 3 failures
```

**When to use:** Detect stuck apps and restart them

---

### 2. Readiness Probe

**Question:** Is the app ready to serve traffic?

**If fails:** Pod is removed from Service (traffic stops)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 1
```

**When to use:** App needs time to start (loading data, connecting to DB)

---

### 3. Startup Probe

**Question:** Has the app finished starting up?

**If fails:** Keep waiting (other probes don't run until this passes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    startupProbe:
      httpGet:
        path: /startup
        port: 3000
      failureThreshold: 30
      periodSeconds: 2    # Check every 2s for 60s total
```

**When to use:** App takes long time to start (load big database, initialize cache)

---

## Probe Order

```
Container starts
    ↓
Startup Probe runs
  (Is app starting?)
    ↓
Startup passes (or timeout)
    ↓
Readiness + Liveness run continuously
  (Is app ready? Is app alive?)
```

---

## Probe Methods

### Method 1: HTTP (Most Common)

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
```

App must have health endpoint that returns 200 OK.

---

### Method 2: TCP

```yaml
readinessProbe:
  tcpSocket:
    port: 3000
```

Kubernetes checks if port is open/listening.

---

### Method 3: Command

```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - test -f /tmp/healthy
```

Runs command inside container. Exit code 0 = healthy.

---

## Real 3-Tier Example

**Deployment with all probes:**

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
        image: myapp:latest
        ports:
        - containerPort: 3000
        
        # App takes time to start (connect to DB)
        startupProbe:
          httpGet:
            path: /startup
            port: 3000
          failureThreshold: 30
          periodSeconds: 2
        
        # Check if ready for traffic
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # Check if alive
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
```

---

## Common Settings

```yaml
initialDelaySeconds: 10    # Wait 10s before first check
                           # (app needs time to start)

periodSeconds: 10          # Check every 10s

timeoutSeconds: 1          # Wait 1s for response

failureThreshold: 3        # Fail after 3 checks fail

successThreshold: 1        # Pass after 1 check succeeds
```

---

## What Happens

### Scenario 1: App Crashes

```
Liveness probe checks every 10s
    ↓
App is dead (no response)
    ↓
Fails 3 times (30s total)
    ↓
Kubernetes restarts pod
    ↓
New pod starts
    ↓
Ready for traffic
```

**Result:** Automatic recovery (users don't notice)

---

### Scenario 2: App Not Ready

```
Pod starts
    ↓
Startup probe: Is app ready to start health checks?
    → NO (still loading data)
    ↓
Wait, retry...
    ↓
Readiness probe: Is app ready for traffic?
    → NO (still connecting to DB)
    ↓
Service removes pod (traffic goes to other pods)
    ↓
Pod finishes loading
    ↓
Readiness probe: Is app ready?
    → YES
    ↓
Service adds pod back (traffic now goes here)
```

**Result:** No broken requests (pod only gets traffic when ready)

---

## Test Probes

```bash
# Deploy with probes
kubectl apply -f deployment.yaml

# Watch pod status
kubectl get pods -w

# Describe pod (shows probe events)
kubectl describe pod backend-pod-xxx

# Check if pod is ready
kubectl get pods
# Ready column: 1/1 (ready), 0/1 (not ready)

# Check logs if probe fails
kubectl logs backend-pod-xxx
```

---

## Best Practices

### 1. Always Use Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
```

Prevents broken requests.

---

### 2. Use Liveness for Stuck Apps

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
```

Restarts dead/stuck pods automatically.

---

### 3. Use Startup for Slow Starts

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 3000
  failureThreshold: 30
```

Give app time to start without liveness killing it.

---

### 4. Set Realistic Timeouts

```yaml
# Don't be too aggressive
initialDelaySeconds: 10    # ✅ Give app time
periodSeconds: 10          # ✅ Don't check too often
failureThreshold: 3        # ✅ Allow some failures

# Bad
initialDelaySeconds: 1     # ❌ Too fast
periodSeconds: 1           # ❌ Too frequent
failureThreshold: 1        # ❌ Too strict
```

---

## Quick Checklist

Before deploying:
- [ ] App has /health endpoint (liveness)
- [ ] App has /ready endpoint (readiness)
- [ ] App has /startup endpoint (if slow to start)
- [ ] Endpoints return 200 OK when healthy
- [ ] Timeouts are realistic
- [ ] failureThreshold makes sense

---

## Summary

**Health Probes = Keep pods healthy**

- **Startup:** App finishing startup?
- **Readiness:** App ready for traffic?
- **Liveness:** App alive and working?

**Use all 3 for production apps.**

```
Startup → Readiness + Liveness continuously
```

**Benefits:**
- Auto-restart dead apps
- Auto-remove unhealthy pods
- Zero downtime deployments
- Reliable microservices
