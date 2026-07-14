# Kubernetes Secrets 

## What is a Secret?

A **Secret** is a Kubernetes object used to store **sensitive information** such as:

* Database passwords
* API keys
* Tokens
* SSH keys
* TLS certificates

**Simple definition:**

> Secret = A secure place to store passwords and other sensitive data.

---

# Why Do We Need Secrets?

### ❌ Bad Practice (Hardcoding Password)

```yaml
env:
- name: DB_PASSWORD
  value: "password123"
```

Problems:

* Password is visible in YAML.
* If pushed to Git, anyone can see it.
* Difficult to change later.

---

### ✅ Good Practice (Using Secret)

Store the password in a Secret and reference it in your Deployment.

```text
Secret
   ↓
Deployment
   ↓
Container
```

This keeps passwords separate from your application code.

---

# Secret vs ConfigMap

| ConfigMap          | Secret         |
| ------------------ | -------------- |
| Non-sensitive data | Sensitive data |
| App settings       | Passwords      |
| Hostnames          | API Keys       |
| Ports              | Tokens         |

### Rule to Remember

* **ConfigMap → Normal configuration**
* **Secret → Sensitive information**

---

# Basic Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret

type: Opaque

data:
  DB_PASSWORD: cGFzc3dvcmQ=
```

### Important Fields

* `apiVersion` → API version
* `kind` → Secret
* `metadata.name` → Secret name
* `type` → Usually `Opaque`
* `data` → Base64 encoded values

---

# Most Common Secret Type

## Opaque

Used for:

* Passwords
* API Keys
* Tokens

Example:

```yaml
type: Opaque
```

This is the default Secret type and the one you'll use most of the time.

---

# Using Secret in a Deployment

## Method 1 (Recommended)

Load all Secret values.

```yaml
envFrom:
- secretRef:
    name: app-secret
```

All keys become environment variables.

Example:

```
DB_PASSWORD
API_KEY
TOKEN
```

---

## Method 2

Load only one key.

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: DB_PASSWORD
```

Use this when you need only one value.

---

## Method 3

Mount Secret as files.

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: app-secret
```

The Secret will appear inside the container as files.

---

# Create Secret

Using kubectl:

```bash
kubectl create secret generic app-secret \
--from-literal=DB_PASSWORD=password123
```

---

# Common Commands

Create Secret

```bash
kubectl create secret generic app-secret \
--from-literal=DB_PASSWORD=password123
```

List Secrets

```bash
kubectl get secrets
```

Describe Secret

```bash
kubectl describe secret app-secret
```

View Secret (Base64)

```bash
kubectl get secret app-secret -o yaml
```

Delete Secret

```bash
kubectl delete secret app-secret
```

---

# Base64 Encoding

Encode

```bash
echo -n "password123" | base64
```

Decode

```bash
echo "cGFzc3dvcmQxMjM=" | base64 -d
```

---

# Best Practices

✅ Store passwords in Secrets.

✅ Keep Secrets out of Git.

✅ Use different Secrets for Dev, Test, and Production.

✅ Give Secret access only to required applications.

---

# Important Note

**Base64 is NOT encryption.**

It only converts text into another format.

Anyone with access can decode it.

For production, use Secret encryption or external tools like Vault or AWS Secrets Manager.

---

# Quick Interview Questions

### What is a Kubernetes Secret?

A Kubernetes object used to store sensitive information like passwords, API keys, and tokens.

### Difference between Secret and ConfigMap?

* ConfigMap stores normal configuration.
* Secret stores sensitive information.

### Which Secret type is used most?

`Opaque`

### Can Secret values be stored as plain text?

No. They must be Base64 encoded.

### Does Base64 mean encryption?

No. Base64 is encoding, not encryption.

---

# Summary

* Secret stores sensitive data.
* Use Secret for passwords, tokens, and API keys.
* Use ConfigMap for normal configuration.
* Most commonly used type is **Opaque**.
* Access Secrets using **env**, **envFrom**, or **Volume Mounts**.
* Never store passwords directly in Deployment YAML.
