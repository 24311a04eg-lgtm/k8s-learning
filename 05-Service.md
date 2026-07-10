# Kubernetes Services - Complete Guide

## Table of Contents
1. [What is a Service?](#what-is-a-service)
2. [Why Services Exist](#why-services-exist)
3. [Port Mapping Explained](#port-mapping-explained)
4. [Label & Selector Matching](#label--selector-matching)
5. [Load Balancing](#load-balancing)
6. [Service Types](#service-types)
7. [Service YAML Explained](#service-yaml-explained)
8. [Common Commands](#common-commands)
9. [Examples](#examples)

---

## What is a Service?

A **Service** is a stable network endpoint that routes traffic to your pods.

### Simple Definition
- **What**: A load balancer + DNS name for your pods
- **Why**: Pods die and get new IPs. Services provide a stable way to access them
- **How**: Service finds pods by labels and distributes traffic

### Visual
```
User/App
    ↓
Service (stable IP/DNS)
    ├──→ Pod 1 (10.244.0.1)
    ├──→ Pod 2 (10.244.0.2)
    └──→ Pod 3 (10.244.0.3)
```

---

## Why Services Exist

### Problem: Pods are Temporary
```
Pod 1: 10.244.0.1  ← Gets new IP when it dies
Pod 2: 10.244.0.2  ← Gets new IP when it dies
Pod 3: 10.244.0.3  ← Gets new IP when it dies
```

If you hardcode these IPs, everything breaks when a pod restarts.

### Solution: Service
```
Service (stable DNS name: nginx-service)
    ↓
Always routes to healthy pods
    ↓
Even if pods restart with new IPs
```

**Service = the bridge between temporary pods and permanent access.**

---

## Port Mapping Explained

Three different ports work together:

```
User Request
    ↓
port (Service port)          ← User connects here
    ↓ (translates to)
targetPort (Service forwards) ← Pod receives here
    ↓ (container listens on)
containerPort (Container)     ← App actually listens here
```

### Example
```yaml
Service:
  port: 80              # User accesses Service on 80
  targetPort: 80        # Service forwards to pod port 80

Deployment:
  containerPort: 80     # Container/app listens on 80
```

### Key Rules
- **targetPort = containerPort** (must match, always)
- **port** can be different (optional)

### Scenario with Different Ports
```yaml
Service:
  port: 8080            # User accesses Service on 8080
  targetPort: 3000      # Service forwards to pod port 3000

Deployment:
  containerPort: 3000   # App listens on 3000
```

Flow: User → 8080 → Service → 3000 → App

---

## Label & Selector Matching

### How Service Finds Pods

**Service Discovery = Labels matching**

```
Deployment creates pods with labels
    ↓
Service searches for pods with matching labels
    ↓
If labels match → Service connects to pods ✅
If labels don't match → Service has no pods ❌
```

### Example

**Deployment:**
```yaml
metadata:
  labels:
    app: nginx           # ← Pods get this label

template:
  labels:
    app: nginx           # ← Every pod gets this label
```

**Service:**
```yaml
selector:
  app: nginx             # ← Service looks for this label
```

**Match?**
```
Deployment label: app=nginx
Service selector: app=nginx
Result: MATCH ✅ Service finds all pods
```

### No Match = Service Fails
```
Deployment label: app=nginx
Service selector: app=apache    # Different!
Result: NO MATCH ❌ Service has 0 pods
```

### Verification
```bash
kubectl describe svc nginx-service
# Check "Endpoints:" - should list pod IPs
# If empty → labels don't match
```

---

## Load Balancing

### What is Load Balancing?

Service automatically distributes traffic across all matching pods.

```
3 Pods running
5 Requests come in:

Request 1 → Pod 1
Request 2 → Pod 2
Request 3 → Pod 3
Request 4 → Pod 1 (repeat)
Request 5 → Pod 2 (repeat)
```

### Algorithm: Round-Robin
Traffic goes: Pod 1, Pod 2, Pod 3, Pod 1, Pod 2, Pod 3...

### Why It Matters
```
Without Service (if accessing pod directly):
- All traffic → Pod 1
- Pod 1 gets overloaded

With Service:
- Traffic spreads across 3 pods
- Each pod gets 1/3 of traffic
- No single pod overloaded
```

### Testing Load Balancing
```bash
# Make multiple requests
for i in {1..9}; do curl http://service-ip:port; done

# Each request goes to different pod (round-robin)
# Pod 1 gets 3 requests, Pod 2 gets 3, Pod 3 gets 3
```

---

## Service Types

### 1. ClusterIP (Default, Internal Only)

**Access:** Inside cluster only

**YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

**How to Access:**
- From another pod: `curl http://nginx-service:80`
- From your PC: Use `kubectl port-forward` (for testing)

**Use Case:**
- Pod-to-pod communication
- Database services (internal only)
- Local testing

---

### 2. NodePort (External via Node Port)

**Access:** Outside cluster via node-ip:nodePort

**YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 8000
    targetPort: 80
    nodePort: 30080
```

**Port Breakdown:**
- `port: 8000` - Internal Service port (pod-to-pod)
- `targetPort: 80` - Pod port
- `nodePort: 30080` - External Node port (what you use)

**How to Access:**
```bash
curl http://192.168.49.2:30080
# Replace IP with your Node IP (minikube ip)
```

**Use Case:**
- Local testing (Minikube)
- Exposing apps externally without cloud load balancer
- Not production-grade

**Port Range:** 30000-32767 (required for nodePort)

---

### 3. LoadBalancer (Cloud External IP)

**Access:** Outside cluster via public load balancer IP

**YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

**What Happens:**
- Cloud provider (AWS, Azure, GCP) creates a real load balancer
- Gets a public IP/DNS
- Routes traffic to your pods

**How to Access:**
```bash
kubectl get svc nginx-service
# EXTERNAL-IP shows the public IP
curl http://your-load-balancer-ip
```

**Use Case:**
- Production apps
- Need public internet access
- AWS/Azure/GCP only (not Minikube)

**Note:** In Minikube, EXTERNAL-IP stays `<pending>`

---

## Service YAML Explained

### Minimum Service YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx              # MUST match Deployment labels
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Field Breakdown

**`apiVersion: v1`**
- Service uses core API (`v1`)

**`kind: Service`**
- Creating a Service

**`metadata.name`**
- Service name
- Used in commands: `kubectl get svc nginx-service`
- Used in DNS: `nginx-service:80` (inside cluster)

**`selector`**
- Which pods this Service manages
- Must match Deployment `template.metadata.labels`
- Service searches for pods with these labels

**`ports`**
- `port` - Service's own port (internal, pod-to-pod)
- `targetPort` - Pod port (must match containerPort)
- `nodePort` - External port (only for NodePort type)

**`type`**
- `ClusterIP` - internal only (default)
- `NodePort` - expose via node port
- `LoadBalancer` - cloud external IP

---

## Common Commands

### Get Services
```bash
kubectl get svc
kubectl get service

# With more details
kubectl get svc -o wide
```

### Describe Service
```bash
kubectl describe svc nginx-service

# Look for:
# - Endpoints: (should show pod IPs, if empty → labels don't match)
# - Port mapping
# - Selector info
```

### Check Endpoints (which pods connected)
```bash
kubectl get endpoints
kubectl get endpoints nginx-service
```

### Port Forward (for ClusterIP testing)
```bash
kubectl port-forward svc/nginx-service 8080:80
# In another terminal: curl http://localhost:8080
```

### Delete Service
```bash
kubectl delete svc nginx-service
```

---

## Examples

### Example 1: Simple Nginx Service

**Deployment:**
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Service (ClusterIP):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**Apply & Test:**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Test inside cluster
kubectl run test-pod --image=busybox -it --rm -- curl http://nginx-service

# Test from outside
kubectl port-forward svc/nginx-service 8080:80
# curl http://localhost:8080
```

---

### Example 2: NodePort for External Access

**Same Deployment + NodePort Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

**Test:**
```bash
minikube ip  # Get node IP, e.g., 192.168.49.2

curl http://192.168.49.2:30080
# Should show nginx welcome page
```

---

### Example 3: Multiple Apps with Different Services

**Deployment 1 (Web):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Service 1 (Web):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**Deployment 2 (API):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myapi:latest
        ports:
        - containerPort: 8000
```

**Service 2 (API):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

**Result:**
- Web pods accessible via `web-service:80`
- API pods accessible via `api-service:8000`
- Pods can communicate with each other
- Both services internal (ClusterIP)

---

## Troubleshooting

### Service has no Endpoints
```bash
kubectl describe svc nginx-service

# If Endpoints: <none>, labels don't match
# Fix: Ensure Service selector matches Deployment labels
```

### Can't Access Service from Outside (ClusterIP)
```bash
# ClusterIP is internal only
# Use port-forward for testing or change to NodePort
kubectl port-forward svc/nginx-service 8080:80
```

### Port Already in Use
```bash
# If 8080 is busy, use different port
kubectl port-forward svc/nginx-service 9090:80
```

### Connection Refused
```bash
# Check if Service and Deployments are created
kubectl get svc
kubectl get deployments
kubectl get pods

# Check endpoints
kubectl get endpoints
```

---

## Key Takeaways

1. **Service = stable endpoint** for temporary pods
2. **Selector must match labels** or Service has no pods
3. **Port mapping:** port (service) → targetPort (pod) → containerPort (app)
4. **Load balancing:** automatic round-robin across pods
5. **3 types:** ClusterIP (internal), NodePort (external), LoadBalancer (cloud)
6. **Testing:** Use `kubectl describe` to verify everything connected

---

## Next Steps

Once comfortable with Services, learn:
- **Ingress** - Advanced routing for multiple apps
- **ConfigMaps/Secrets** - Configuration management
- **StatefulSets** - For stateful applications

---

## Quick Reference

| Concept | Definition |
|---------|-----------|
| **Selector** | Labels used to find pods |
| **Port** | Service's own port (internal) |
| **TargetPort** | Pod port (must match containerPort) |
| **NodePort** | External port on node (30000-32767) |
| **Endpoint** | Actual pod IP that Service routes to |
| **Load Balancing** | Auto-distribute traffic across pods |
| **ClusterIP** | Internal access only |
| **NodePort** | External via node-ip:port |
| **LoadBalancer** | Cloud external IP |
