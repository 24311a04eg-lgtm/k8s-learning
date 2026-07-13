# Kubernetes Ingress - Complete Guide

## What is Ingress?

A **Service** is a stable network endpoint for pods. An **Ingress** is a smart HTTP router that directs traffic to multiple services based on domain names or URL paths.

### Visual
```
User Request
    ↓
Ingress (smart router)
    ├─→ example.com/api → API Service → Pods
    ├─→ example.com/web → Web Service → Pods
    └─→ api.example.com → API Service → Pods
```

---

## Why Ingress Exists

### Problem
```
Without Ingress:
- Each app needs its own LoadBalancer IP
- Expensive! ❌

With Ingress:
- One LoadBalancer IP for all apps
- Routes based on domain/path ✅
```

---

## Ingress vs Service

| Feature | Service | Ingress |
|---------|---------|---------|
| **Layer** | Layer 4 (TCP) | Layer 7 (HTTP/HTTPS) |
| **For** | Pod-to-pod communication | External HTTP traffic |
| **Routing** | IP:Port only | Domain names & paths |
| **HTTPS** | No | Yes |

---

## Before Using Ingress

### You Need:
1. **Ingress Controller** - software that routes traffic (NGINX, Traefik, etc.)
2. **Services** - to route traffic to
3. **Deployments** - pods to serve requests

### Check if Controller Exists
```bash
kubectl get pods -n ingress-nginx
# or
kubectl get pods -n kube-system | grep ingress
```

### Install NGINX Controller (Minikube)
```bash
minikube addons enable ingress

# Verify
kubectl get pods -n ingress-nginx
```

---

## Two Types of Routing

### 1. Path-Based Routing

Same domain, different paths → different services

```
example.com/api   → api-service
example.com/web   → web-service
example.com/docs  → docs-service
```

**YAML:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8000
      
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      
      - path: /docs
        pathType: Prefix
        backend:
          service:
            name: docs-service
            port:
              number: 3000
```

---

### 2. Host-Based Routing

Different domains → different services

```
api.example.com   → api-service
web.example.com   → web-service
docs.example.com  → docs-service
```

**YAML:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8000
  
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## Ingress YAML Breakdown

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress                    # Ingress name
spec:
  rules:
  - host: example.com                 # Domain name
    http:
      paths:
      - path: /api                    # URL path
        pathType: Prefix              # Prefix or Exact
        backend:
          service:
            name: api-service         # Service name (must exist)
            port:
              number: 8000            # Service port
```

### Field Definitions

**`host`** - Domain name (optional, matches all if missing)

**`path`** - URL path to match

**`pathType`**:
- `Prefix` - matches `/api` and `/api/v1`, `/api/users` etc.
- `Exact` - matches only exactly `/api`

**`backend.service.name`** - Service name to route to

**`backend.service.port.number`** - Service's port (NOT container port)

---

## Complete Example

### Step 1: Create Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
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

### Step 2: Create Service
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

### Step 3: Create Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### Step 4: Apply & Test
```bash
# Apply all
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

# Get Ingress IP
kubectl get ingress
# Example: 192.168.49.2

# Add to /etc/hosts (for testing)
# Windows: C:\Windows\System32\drivers\etc\hosts
# Mac/Linux: /etc/hosts
# Add: 192.168.49.2 example.com

# Test
curl http://example.com
```

---

## Common Commands

```bash
# List Ingresses
kubectl get ingress

# Detailed info
kubectl describe ingress nginx-ingress

# View YAML
kubectl get ingress nginx-ingress -o yaml

# Watch for changes
kubectl get ingress -w

# Delete Ingress
kubectl delete ingress nginx-ingress
```

---

## HTTPS/TLS

### Create Self-Signed Certificate
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Create Kubernetes Secret
kubectl create secret tls my-tls-secret --cert=cert.pem --key=key.pem
```

### Add TLS to Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: my-tls-secret
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### Test HTTPS
```bash
curl -k https://example.com  # -k ignores self-signed warning
```

---

## Troubleshooting

### Ingress shows no EXTERNAL-IP

```bash
# No Ingress Controller
kubectl get pods -n ingress-nginx

# Install it
minikube addons enable ingress
```

### 502 Bad Gateway

```bash
# Service doesn't exist or wrong name
kubectl get svc
kubectl describe svc nginx-service
```

### 404 Not Found

```bash
# Path doesn't match
# Check: kubectl describe ingress my-ingress
# Verify pathType and exact path
```

### Can't reach from browser

```bash
# Add domain to /etc/hosts
# Get Ingress IP: kubectl get ingress
# Add: IP_ADDRESS example.com

# Or use port-forward
kubectl port-forward ingress/nginx-ingress 8080:80
# Then: curl http://localhost:8080
```

---

## Key Takeaways

1. **Ingress** = HTTP/HTTPS router for multiple apps
2. **Must have Ingress Controller** (NGINX, Traefik, etc.)
3. **Routes to Services** (not directly to Pods)
4. **Two types**: path-based (`/api`) and host-based (`api.example.com`)
5. **Saves money** - one LoadBalancer IP for many apps
6. **Service must exist** before Ingress can route to it

---

## Quick Reference

```
Ingress → Service → Pods
Router   Load Balancer  Containers

Domain/Path → Service → Load Balance to Pods
```

**pathType Examples:**
- `Prefix`: `/api` matches `/api`, `/api/v1`, `/api/users`
- `Exact`: `/api` matches only `/api`

**Always remember:**
- `backend.service.port.number` = Service port (not container port)
- `host` = domain name
- `path` = URL path
- Service must exist and match labels in Deployment
