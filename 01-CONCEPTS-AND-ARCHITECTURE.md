# Kubernetes: Introduction & Architecture

## Why does Kubernetes exist?

Docker containers are great — isolated, portable, reproducible. But at scale, managing them manually is a nightmare:
- Keep containers alive (restart if they crash)
- Scale services when traffic spikes
- Roll out updates without downtime
- Spread load across multiple machines
- Route traffic to the right containers

**Kubernetes solves this automatically.** You tell it *what* you want, it figures out *how*.

---

## Core Concept: Desired State

You declare: "I want 3 replicas of my app running always."

Kubernetes watches and enforces it:
- If 1 pod dies → restart it
- If new code pushed → roll out updates gracefully
- If traffic increases → scale up
- Keeps reality matching your declaration forever

**You declaratively say WHAT, K8s figures out HOW.**

---

## Architecture: Two Node Types

### Master Node (Control Plane)
The brain. Makes decisions. Usually 1-3 nodes for high availability.

**Components:**

**API Server**
- Every request goes through this
- kubectl, scheduler, kubelet all talk to it
- Single entry point for the cluster
- Validates and authenticates all requests

**etcd**
- Distributed key-value database
- Stores ALL cluster state
- "How many pods should run? Which nodes exist?"
- If etcd dies, K8s loses memory

**Scheduler**
- When new pod needs to run, picks the best worker node
- Checks CPU/RAM availability
- Looks at taints, tolerations, affinity rules
- Assigns pod to a node

**Controller Manager**
- Runs control loops in background
- ReplicaSet controller: "Need 3 replicas? If one died, restart it."
- Deployment, Node, Service controllers, etc.
- **Reconciliation loop**: constantly checks desired state vs actual state, fixes drift

---

### Worker Node (Data Plane)
The muscle. Runs your containers. Can be many nodes.

**Components:**

**kubelet**
- Agent on every worker node
- Watches API server for pods assigned to its node
- Tells container runtime to start/stop containers
- Reports node health back to control plane

**kube-proxy**
- Manages network rules on the node
- When traffic hits a Service IP, routes it to correct pod
- Uses iptables/ipvs under the hood

**Container Runtime**
- containerd or CRI-O (Docker deprecated as K8s runtime)
- Actually starts/stops containers
- kubelet talks to it via CRI (Container Runtime Interface)

**Pods**
- Smallest deployable unit in K8s
- One or more containers sharing:
  - Same IP address (talk via localhost)
  - Same storage volumes
- Usually 1 container per pod

---

## Architecture Diagram
┌─────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────── Master Node ─────────────────┐  │
│  │                                               │  │
│  │  ┌──────────────┐  ┌──────────────┐         │  │
│  │  │ API Server   │  │ Scheduler    │         │  │
│  │  └──────────────┘  └──────────────┘         │  │
│  │                                               │  │
│  │  ┌──────────────┐  ┌──────────────┐         │  │
│  │  │ etcd         │  │ Controller   │         │  │
│  │  │ (state db)   │  │ Manager      │         │  │
│  │  └──────────────┘  └──────────────┘         │  │
│  │                                               │  │
│  └───────────────────────────────────────────────┘  │
│                      ↓                               │
│         API calls, scheduling, watching             │
│                      ↓                               │
│  ┌──────────────────────────────────────────────┐   │
│  │ Worker Node 1        Worker Node 2           │   │
│  │                                              │   │
│  │ ┌────────────┐      ┌────────────┐         │   │
│  │ │ kubelet    │      │ kubelet    │         │   │
│  │ └────────────┘      └────────────┘         │   │
│  │                                              │   │
│  │ ┌────────────┐      ┌────────────┐         │   │
│  │ │ kube-proxy │      │ kube-proxy │         │   │
│  │ └────────────┘      └────────────┘         │   │
│  │                                              │   │
│  │ ┌──────────┐        ┌──────────┐           │   │
│  │ │  Pod A   │        │  Pod C   │           │   │
│  │ │          │        │          │           │   │
│  │ └──────────┘        └──────────┘           │   │
│  │                                              │   │
│  │ ┌──────────┐        ┌──────────┐           │   │
│  │ │  Pod B   │        │  Pod D   │           │   │
│  │ │          │        │          │           │   │
│  │ └──────────┘        └──────────┘           │   │
│  │                                              │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘

---

## Request Flow: kubectl apply to running pod

You: kubectl apply -f deployment.yaml
kubectl → API Server (Master)
└── "Create this deployment"
API Server → validates, stores in etcd
└── desired state saved
Controller Manager → watches etcd
└── "New deployment detected"
Scheduler → picks best worker node
└── "Pod should run on Worker-2"
API Server → kubelet on Worker-2
└── "Start this pod"
kubelet → Container Runtime
└── "Start container from image"
Pod is running
└── Container working
Controller Manager watches forever
└── Pod dies? Restart it.
└── Deployment deleted? Clean up.
└── Node dies? Reschedule pods.


---

## Communication Patterns

### Pattern 1: Master decides, Workers execute
Master: "Start pod X on Worker-2"
Worker-2: "Okay, starting pod X"
Master: "Is it still running?"
Worker-2: "Yes, all healthy"

### Pattern 2: kubelet watches API Server
kubelet continuously asks:
"Do I have any new pods?"
"Are my pods healthy?"
"Should I restart any?"

### Pattern 3: etcd is source of truth
Desired state? → Check etcd
Actual state? → kubelet reports
Do they match? → If no, fix it

### Pattern 4: Declarative enforcement
Your YAML = desired state
etcd = stored desired state
Reality = actual state
Controller Manager = constantly enforces: desired == actual

---

## Key Objects You Create

**Pod**
- Wraps one or more containers
- Smallest K8s unit

**Deployment**
- "Keep N replicas of this pod running"
- Handles rolling updates
- What you use 90% of the time

**Service**
- Stable networking endpoint
- DNS name that never changes
- Routes traffic to healthy pods

**ConfigMap**
- Inject config (env vars, files)
- Non-sensitive data

**Secret**
- Same as ConfigMap but for sensitive data
- Passwords, API keys

**PersistentVolume (PV)**
- Physical storage

**PersistentVolumeClaim (PVC)**
- App requests storage

**Ingress**
- External HTTP routing into cluster

**Namespace**
- Logical isolation within cluster

---

## The Mental Model Shift

### Docker (Imperative)
```bash
docker run -d my-app
docker stop container-id
docker start container-id
# You manually manage each container
```

### Kubernetes (Declarative)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3  # declare what you want
  # K8s figures out how to make it happen
```

---

## Master vs Worker: Quick Comparison

| Aspect | Master | Worker |
|---|---|---|
| How many? | 1-3 (HA) | Many (10+) |
| Components | API, etcd, Scheduler, Controller | kubelet, kube-proxy, runtime |
| Runs app pods? | No (usually) | Yes |
| Fails = ? | Cluster can't schedule | Apps migrate to other nodes |

---

## Single vs Multi-Node

**Minikube (single node):**
1 machine = Master + Worker both

**Production:**
3 Master nodes (HA)
10+ Worker nodes

---

## Remember

**Master = brain** (decisions, state)
**Worker = muscle** (runs containers)

Master talks via API Server.
Worker watches API Server.
Controller Manager watches both, fixes drift.

You interact with Master (kubectl).
Master handles everything else.
