# Kubernetes Storage (Persistent Volumes & Persistent Volume Claims)

## Why Storage is Needed?

By default, data stored inside a container is **temporary**.

If the Pod is deleted or restarted, all the data inside the container is lost.

To store data permanently, Kubernetes provides **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)**.

---

## Temporary Storage with `emptyDir`

An `emptyDir` volume is created when a Pod starts.

All containers in the Pod can share it.

However:

- Data exists only while the Pod is running.
- If the Pod is deleted, all data is deleted.

Example:

```text
Pod Starts
     │
 emptyDir Created
     │
 Application Stores Data
     │
Pod Deleted
     │
Data Lost
```

---

## What is a Persistent Volume (PV)?

A **Persistent Volume (PV)** is a piece of storage managed by Kubernetes.

It can represent:

- A cloud disk
- An NFS share
- A local disk
- Any supported storage system

Unlike `emptyDir`, a PV exists **independently of Pods**.

Think of a PV as a hard drive available in the Kubernetes cluster.

---

## What is a Persistent Volume Claim (PVC)?

A **Persistent Volume Claim (PVC)** is a request for storage made by a Pod.

Instead of directly using a Persistent Volume, a Pod requests storage through a PVC.

Kubernetes automatically connects the PVC to a matching PV.

Think of it like this:

- **PV = Storage**
- **PVC = Request for Storage**
- **Pod = Uses the Requested Storage**

---

## How PV and PVC Work

```text
          Pod
           │
           │
          PVC
           │
           │
          PV
           │
           │
     Physical Storage
```

The Pod never connects directly to the storage.

It connects through the PVC.

---

# Persistent Volume YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv

spec:
  capacity:
    storage: 1Gi

  accessModes:
    - ReadWriteOnce

  hostPath:
    path: /data/app
```

---

## YAML Breakdown

| Field | Meaning |
|---|---|
| `capacity.storage` | Maximum storage size |
| `accessModes` | How Pods can access the volume |
| `hostPath` | Local directory on the Kubernetes node |

---

# Persistent Volume Claim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc

spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi
```

---

## YAML Breakdown

| Field | Meaning |
|---|---|
| `resources.requests.storage` | Storage requested by the application |
| `accessModes` | Required access mode |

---

# Pod Using a PVC

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-storage

spec:
  containers:
    - name: nginx
      image: nginx:latest

      volumeMounts:
        - name: app-storage
          mountPath: /usr/share/nginx/html

  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: app-pvc
```

---

## How This Works

```text
Pod
│
├── Uses PVC (app-pvc)
│
PVC
│
├── Connected to PV (app-pv)
│
PV
│
└── Stores data on disk
```

Any files written to:

```
/usr/share/nginx/html
```

are stored on the Persistent Volume.

Even if the Pod is deleted, the data remains.

---

# Access Modes

A Persistent Volume can be accessed in different ways.

| Access Mode | Description |
|---|---|
| ReadWriteOnce (RWO) | Mounted as read/write by one node |
| ReadOnlyMany (ROX) | Mounted as read-only by multiple nodes |
| ReadWriteMany (RWX) | Mounted as read/write by multiple nodes |

---

# Storage Classes

A **StorageClass** tells Kubernetes **how to create storage automatically**.

Without a StorageClass:

- Administrator creates PV manually.
- Pod requests PVC.
- PVC binds to an existing PV.

With a StorageClass:

- Pod requests PVC.
- Kubernetes automatically creates the required Persistent Volume.

This process is called **Dynamic Provisioning**.

---

## StorageClass Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass

metadata:
  name: standard-storage

provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

---

## PVC Using a StorageClass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: dynamic-pvc

spec:
  storageClassName: standard-storage

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 2Gi
```

When this PVC is created, Kubernetes uses the StorageClass to provide storage automatically (if supported by the cluster).

---

# Static vs Dynamic Provisioning

| Static Provisioning | Dynamic Provisioning |
|---|---|
| Administrator creates the PV manually | Kubernetes creates the PV automatically |
| PVC binds to an existing PV | PVC requests storage through a StorageClass |
| More manual work | Easier and more scalable |

---

# Common kubectl Commands

```bash
# Create a Persistent Volume
kubectl apply -f pv.yaml

# Create a Persistent Volume Claim
kubectl apply -f pvc.yaml

# Create a Pod
kubectl apply -f pod.yaml

# List Persistent Volumes
kubectl get pv

# List Persistent Volume Claims
kubectl get pvc

# Describe a PV
kubectl describe pv app-pv

# Describe a PVC
kubectl describe pvc app-pvc

# View Pods
kubectl get pods

# Delete Pod
kubectl delete pod nginx-storage

# Delete PVC
kubectl delete pvc app-pvc

# Delete PV
kubectl delete pv app-pv
```

---

# Key Differences

| emptyDir | Persistent Volume |
|---|---|
| Temporary storage | Permanent storage |
| Deleted with the Pod | Exists independently of Pods |
| Good for cache or temporary files | Good for databases and application data |
| Created automatically with the Pod | Managed by Kubernetes |

---

# Key Takeaways

- Containers have temporary storage by default.
- `emptyDir` is temporary and deleted with the Pod.
- A **Persistent Volume (PV)** provides permanent storage.
- A **Persistent Volume Claim (PVC)** requests storage for a Pod.
- Pods use PVCs instead of connecting directly to PVs.
- **StorageClasses** enable automatic (dynamic) storage provisioning.
- Use Persistent Volumes for applications that need to keep data after Pods are restarted or deleted.
