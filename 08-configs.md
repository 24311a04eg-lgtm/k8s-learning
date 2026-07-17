# Kubernetes ConfigMap - Complete Guide

## Table of Contents
1. [What is ConfigMap?](#what-is-configmap)
2. [Why ConfigMap Exists](#why-configmap-exists)
3. [ConfigMap vs Secret](#configmap-vs-secret)
4. [ConfigMap YAML](#configmap-yaml)
5. [How to Use ConfigMap](#how-to-use-configmap)
6. [Create ConfigMap](#create-configmap)
7. [Complete Examples](#complete-examples)
8. [Common Commands](#common-commands)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## What is ConfigMap?

**ConfigMap** = Store non-sensitive configuration data in Kubernetes

### Simple Definition
- **What**: Kubernetes object for storing config
- **For**: Database host, port, log level, app settings
- **Why**: Separate config from code/containers
- **How**: Reference in Deployments via environment variables

### What Goes in ConfigMap
```yaml
✅ DATABASE_HOST: postgres
✅ DATABASE_PORT: "5432"
✅ LOG_LEVEL: debug
✅ APP_NAME: MyApp
✅ CACHE_ENABLED: "true"
✅ MAX_CONNECTIONS: "100"
```

### What Does NOT Go in ConfigMap
```yaml
❌ DATABASE_PASSWORD (use Secret)
❌ API_KEYS (use Secret)
❌ OAuth_TOKENS (use Secret)
❌ Private credentials (use Secret)
```

---

## Why ConfigMap Exists

### Problem: Hardcoded Configuration

**Bad (don't do this):**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  containers:
  - name: backend
    image: myapp:latest
    env:
    - name: DATABASE_HOST
      value: "postgres"          # ← Hardcoded
    - name: DATABASE_PORT
      value: "5432"              # ← In YAML
    - name: LOG_LEVEL
      value: "debug"             # ← Not flexible
```

**Problems:**
- Configuration in YAML (hard to change)
- Same config for all environments (bad)
- Hard to rotate/update without redeploying
- Config mixed with deployment logic

---

### Solution: ConfigMap

**Good (do this):**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
  LOG_LEVEL: debug

---
apiVersion: apps/v1
kind: Deployment
spec:
  containers:
  - name: backend
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config  # Reference ConfigMap
```

**Benefits:**
- Config separated from code
- Easy to update without redeploying
- Different ConfigMaps per environment
- Clear what's configurable

---

## ConfigMap vs Secret

### Key Differences

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| **Data Type** | Non-sensitive config | Sensitive data |
| **Visibility** | Anyone can read | Should be restricted |
| **Encoding** | Plain text | Base64 encoded |
| **Example** | DB_HOST=postgres | DB_PASSWORD=secret123 |
| **Use Case** | App settings | Credentials, keys |

### Decision Tree

**Ask yourself:**
```
Is this sensitive data (password, key, token)?
  YES → Use Secret
  NO → Can anyone see this safely?
       YES → Use ConfigMap
       NO → Use Secret
```

**Examples:**
```
Database host: "prod-db.com"
  → Anyone can know this
  → Use ConfigMap ✅

Database password: "super-secret-pass"
  → Only authenticated access should have this
  → Use Secret ✅

Log level: "debug"
  → Anyone can know this
  → Use ConfigMap ✅

API key: "abc-123-xyz"
  → Only our app should have this
  → Use Secret ✅
```

---

## ConfigMap YAML

### Minimum ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
  LOG_LEVEL: debug
```

### Field Breakdown

**`apiVersion: v1`**
- ConfigMap API version

**`kind: ConfigMap`**
- Creating a ConfigMap

**`metadata.name`**
- ConfigMap name
- Reference in Deployments

**`metadata.namespace`**
- Kubernetes namespace
- Default is `default`

**`data`**
- Key-value pairs (plain text)
- Values are NOT encoded

### Multiple Data Types

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple values
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
  
  # Boolean (as string)
  CACHE_ENABLED: "true"
  
  # Number (as string)
  MAX_CONNECTIONS: "100"
  
  # Complex: Config file
  nginx.conf: |
    server {
      listen 80;
      server_name myapp.com;
    }
```

---

## How to Use ConfigMap

### Method 1: All Keys as Environment Variables

**Loads entire ConfigMap as env vars:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config  # Load all keys
```

**Inside container:**
```bash
echo $DATABASE_HOST   # postgres
echo $DATABASE_PORT   # 5432
echo $LOG_LEVEL       # debug
```

---

### Method 2: Specific Keys Only

**Pick which ConfigMap keys to load:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
```

**Inside container:**
```bash
echo $DB_HOST   # postgres
echo $DB_PORT   # 5432
```

---

### Method 3: Mount as Volume (File)

**Mount ConfigMap as file system:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  
  volumes:
  - name: config
    configMap:
      name: app-config
```

**Inside container:**
```bash
cat /etc/config/DATABASE_HOST   # postgres
cat /etc/config/DATABASE_PORT   # 5432
ls /etc/config                  # lists all keys
```

---

## Create ConfigMap

### Method 1: Command Line

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=postgres \
  --from-literal=DATABASE_PORT=5432 \
  --from-literal=LOG_LEVEL=debug
```

### Method 2: From File

```bash
# Create config file
echo "DATABASE_HOST=postgres" > app.config
echo "DATABASE_PORT=5432" >> app.config

# Create ConfigMap from file
kubectl create configmap app-config --from-file=app.config
```

### Method 3: YAML

```bash
# Create YAML
cat > configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
EOF

# Apply
kubectl apply -f configmap.yaml
```

---

## Complete Examples

### Example 1: Single ConfigMap

**ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
  LOG_LEVEL: debug
  APP_NAME: MyApp
```

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  containers:
  - name: backend
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
```

**Apply:**
```bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
```

---

### Example 2: Environment-Specific ConfigMaps

**Dev ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  DATABASE_HOST: localhost
  LOG_LEVEL: debug
  CACHE_ENABLED: "false"
```

**Prod ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  DATABASE_HOST: prod-db.example.com
  LOG_LEVEL: warn
  CACHE_ENABLED: "true"
```

**Deploy to dev:**
```bash
kubectl apply -f configmap-dev.yaml -n dev
kubectl apply -f deployment.yaml -n dev
```

**Deploy to prod:**
```bash
kubectl apply -f configmap-prod.yaml -n prod
kubectl apply -f deployment.yaml -n prod
```

---

## Common Commands

### List ConfigMaps

```bash
kubectl get configmap
kubectl get configmap -n namespace-name
kubectl get configmap -o wide
```

### Describe ConfigMap

```bash
kubectl describe configmap app-config
# Shows: keys, values
```

### View ConfigMap Data

```bash
# As YAML
kubectl get configmap app-config -o yaml

# Specific value
kubectl get configmap app-config -o jsonpath='{.data.DATABASE_HOST}'
```

### Update ConfigMap

```bash
# Edit interactively
kubectl edit configmap app-config

# Recreate
kubectl delete configmap app-config
kubectl create configmap app-config --from-literal=DATABASE_HOST=newhost
```

### Redeploy Pods (Pick Up New ConfigMap)

```bash
# ConfigMaps don't auto-update pods
# Must restart pods manually

kubectl rollout restart deployment/backend
# or
kubectl delete pod backend-pod-xxx  # Force recreate
```

### Delete ConfigMap

```bash
kubectl delete configmap app-config
kubectl delete configmap --all  # Delete all
```

---

## Best Practices

### 1. Use Different ConfigMaps per Environment

```bash
dev-config, staging-config, prod-config
# Not one config for all environments
```

### 2. Use Descriptive Names

```bash
✅ backend-config
✅ database-config
❌ config1
❌ my-config
```

### 3. Document What Each Key Is For

```yaml
data:
  DATABASE_HOST: postgres  # Where database lives
  DATABASE_PORT: "5432"    # PostgreSQL default port
  LOG_LEVEL: debug         # Verbosity (debug, info, warn, error)
```

### 4. Keep Sensitive Data in Secrets, Not ConfigMap

```yaml
❌ ConfigMap:
  DATABASE_PASSWORD: secret123  # WRONG

✅ Secret:
  DATABASE_PASSWORD: <base64 encoded>

✅ ConfigMap:
  DATABASE_HOST: postgres
  DATABASE_PORT: "5432"
```

### 5. Restart Pods After ConfigMap Update

```bash
# ConfigMaps don't auto-reload
kubectl rollout restart deployment/backend
```

---

## Troubleshooting

### Problem: Pod Not Getting ConfigMap Values

**Check:**
```bash
# 1. ConfigMap exists
kubectl get configmap app-config

# 2. ConfigMap has data
kubectl describe configmap app-config

# 3. Pod references correct ConfigMap name
kubectl describe pod backend-pod-xxx

# 4. Pod is in same namespace as ConfigMap
kubectl get configmap -n your-namespace
```

---

### Problem: ConfigMap Updated But Pod Still Has Old Values

**Fix:**
```bash
# ConfigMaps don't auto-update running pods
# Must restart pods

kubectl rollout restart deployment/backend

# Or delete pod to force recreate
kubectl delete pod backend-pod-xxx
```

---

## Summary

**ConfigMap = Non-sensitive configuration storage**

- Store app settings (database host, port, log level)
- Separate from code
- Reference in Deployments
- Easy to update per environment
- Use Secrets for sensitive data

**Use Pattern:**
```
ConfigMap → Deployment → Pod → Container
```

**Remember:**
- ✅ ConfigMap for non-sensitive config
- ✅ Secret for sensitive data
- ✅ Different ConfigMaps per environment
- ✅ Restart pods after ConfigMap updates
