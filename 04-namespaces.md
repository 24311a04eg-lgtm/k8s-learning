# Kubernetes Namespaces

## What is a Namespace?

A Namespace is a logical partition inside a Kubernetes cluster that groups related resources.

It allows multiple teams, applications, or environments to share the same cluster while remaining logically isolated.

Think of it as folders inside the same operating system.

```
Cluster
├── dev
├── staging
└── production
```

Resources with the same name can exist in different namespaces because Kubernetes treats them separately.

---

## Why Use Namespaces?

Without namespaces, every resource is created in the same space, making large clusters difficult to manage.

Namespaces help by:

- Organizing resources
- Separating environments
- Allowing identical resource names
- Applying security and resource limits independently

Example:

```
dev
└── frontend

prod
└── frontend
```

Both Deployments are valid because they belong to different namespaces.

---

## Default Namespaces

Every cluster contains built-in namespaces.

| Namespace | Purpose |
|-----------|---------|
| default | Default workspace |
| kube-system | Kubernetes system components |
| kube-public | Public cluster resources |
| kube-node-lease | Node heartbeat information |

View them:

```bash
kubectl get ns
```

---

## Creating a Namespace

```bash
kubectl create namespace dev
```

or

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

---

## Deploying into a Namespace

```bash
kubectl apply -f deployment.yaml -n dev
```

or specify it inside the manifest.

```yaml
metadata:
  namespace: dev
```

---

## Viewing Resources

```bash
kubectl get pods -n dev
kubectl get deployments -n dev
kubectl get svc -n dev
```

View resources from every namespace:

```bash
kubectl get pods -A
```

---

## Deleting a Namespace

```bash
kubectl delete namespace dev
```

Deleting a namespace removes **all resources** inside it.

---

## Best Practices

- Use separate namespaces for dev, staging, and production.
- Avoid deploying everything in `default`.
- Use meaningful namespace names.
- Combine namespaces with RBAC and ResourceQuotas for better isolation.

---

## Common Mistakes

- Forgetting to specify the namespace.
- Using only the `default` namespace.
- Assuming namespaces completely isolate networking.
- Accidentally deleting an entire namespace.

---

## Quick Revision

- Namespace = Logical isolation inside one cluster.
- Same resource names can exist across namespaces.
- Default namespace is used if none is specified.
- Deploy with `-n <namespace>`.
- Deleting a namespace deletes everything inside it.
