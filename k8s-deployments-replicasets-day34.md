# Kubernetes Deployments & ReplicaSets — Deep Dive

> **Source:** [DAY-34 | KUBERNETES DEPLOYMENT | REPLICASETS](https://www.youtube.com/watch?v=lVKLkyuRWCY)  
> **Course:** Complete DevOps Course (Day 34)  
> **Instructor:** Abhishek

---

## Overview

This session explains the progression from containers → pods → deployments, focusing on **why deployments exist** and how they implement auto-healing via ReplicaSets.

Key concepts covered:
- Difference between Container, Pod, and Deployment
- What is a ReplicaSet and why it exists
- Auto-healing behavior in Kubernetes
- Zero-downtime deployments
- Live demo: creating, scaling, and deleting pods

---

## Container vs. Pod vs. Deployment

This is a classic Kubernetes interview question. Understanding the distinction is fundamental.

### Container

A container is run by providing all configuration on the command line:

```bash
docker run -it -d \
  --name my-app \
  -p 8080:8080 \
  -v /data:/data \
  --network my-net \
  my-image:latest
```

Kubernetes said: *"Instead of CLI flags, let's use a YAML manifest."*

### Pod

A Pod is a **running specification** for one or more containers. It is the smallest deployable unit in Kubernetes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

**Why multiple containers in one Pod?**  
Containers in the same Pod share the same network (communicate via `localhost`) and can share volumes. The most common use case is **sidecar containers** — e.g., a service mesh proxy (Envoy/Istio) running alongside the main application container.

**Pod limitation:** Pods have **no auto-healing or auto-scaling**. If a Pod is deleted or crashes, it does not come back.

### Deployment

A Deployment is a higher-level abstraction that manages Pods. It provides:

- **Auto-healing** — if a Pod dies, it is automatically recreated
- **Auto-scaling** — you declare a replica count; Kubernetes maintains it
- **Zero-downtime updates** — rolling updates without disrupting live traffic

> In production, **never create Pods directly**. Always use a Deployment.

---

## How a Deployment Works Internally

```
Deployment (YAML manifest)
    │
    └─▶ ReplicaSet (kubernetes controller)
              │
              ├─▶ Pod 1
              ├─▶ Pod 2
              └─▶ Pod 3
```

1. You create a `Deployment` resource.
2. The Deployment controller creates a **ReplicaSet**.
3. The ReplicaSet controller creates and maintains the specified number of **Pods**.

You never create a ReplicaSet directly — the Deployment manages it for you.

---

## What is a ReplicaSet?

A ReplicaSet is a **Kubernetes controller** (a Go application running inside the cluster) that enforces a desired replica count.

**Its single responsibility:** ensure the *actual* number of running Pods always matches the *desired* number declared in the Deployment YAML.

| Event | ReplicaSet Response |
|-------|---------------------|
| Pod crashes | Creates a replacement Pod immediately |
| User manually deletes a Pod | Creates a replacement Pod immediately |
| Replica count increased (e.g., 1 → 3) | Creates 2 new Pods |
| Replica count decreased (e.g., 3 → 1) | Terminates 2 Pods |

**Key concept — desired state vs. actual state:**  
A Kubernetes controller is anything that ensures the actual cluster state always matches the desired state declared in the YAML manifest. ReplicaSet is one such controller.

---

## Interview Questions

### Q1: What is the difference between Container, Pod, and Deployment?

| | Container | Pod | Deployment |
|--|-----------|-----|------------|
| **Unit** | Single process/app | One or more containers | Manages ReplicaSets & Pods |
| **Auto-healing** | No | No | Yes (via ReplicaSet) |
| **Auto-scaling** | No | No | Yes |
| **YAML-based** | No (CLI flags) | Yes | Yes |
| **Use in production** | No | No | Yes |

### Q2: What is the difference between Deployment and ReplicaSet?

- **Deployment** is the high-level abstraction — you interact with it.
- **ReplicaSet** is the controller created by the Deployment — it does the actual work of maintaining Pod count.
- You should manage Deployments, not ReplicaSets directly.

---

## Deployment YAML Structure

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3           # desired Pod count
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

Key fields:
- `replicas` — how many Pod instances to maintain
- `selector.matchLabels` — how the Deployment finds its Pods
- `template` — the Pod spec used to create each replica

---

## Useful Commands

```bash
# List everything in the current namespace
kubectl get all

# List all resources across all namespaces
kubectl get all -A

# Create or update a Deployment from a YAML file
kubectl apply -f deployment.yaml

# Watch Pod status in real-time
kubectl get pods -w

# Get detailed Pod info (including IP, node)
kubectl get pods -o wide

# View ReplicaSets
kubectl get rs

# Scale a deployment
kubectl scale deployment nginx-deployment --replicas=5

# Edit a deployment live
kubectl edit deployment nginx-deployment
```

---

## Live Demo Walkthrough

### Step 1 — Pod without Deployment (shows the problem)

```bash
kubectl apply -f pod.yaml
kubectl get pods -o wide   # note the Pod IP

# SSH into cluster and curl the app
minikube ssh
curl 172.17.0.3

# Delete the pod (simulates crash)
kubectl delete pod nginx

# App is now gone — no auto-healing
curl 172.17.0.3   # connection refused
```

### Step 2 — Create a Deployment (auto-healing in action)

```bash
kubectl apply -f deployment.yaml
kubectl get deploy
kubectl get rs        # ReplicaSet created automatically
kubectl get pods      # Pods created automatically
```

### Step 3 — Demonstrate auto-healing

Open two terminals:

```bash
# Terminal 1 — watch pods live
kubectl get pods -w

# Terminal 2 — delete a pod
kubectl delete pod <pod-name>
```

Observation: Before the deleted Pod is fully terminated, a **new Pod starts in parallel**. The replica count is always maintained.

### Step 4 — Scale up

Edit `deployment.yaml`, change `replicas: 1` to `replicas: 3`, then:

```bash
kubectl apply -f deployment.yaml
kubectl get pods   # 3 pods now running
```

---

## Zero-Downtime Deployment Explained

When you increase replicas from 100 → 150:
- ReplicaSet creates 50 new Pods in parallel with the existing 100.
- No existing traffic is disrupted.
- Once new Pods are `Running`, they start serving traffic.

When a Pod is deleted (accidentally or during an update):
- Termination and new Pod creation happen **in parallel**.
- Users experience no interruption.

---

## Key Takeaways

- **Never create Pods directly** in production — use Deployments.
- **ReplicaSet** is the controller that implements auto-healing; it is always created by the Deployment.
- A **Kubernetes controller** = anything that ensures desired state == actual state.
- Deployments give you auto-healing, auto-scaling, and zero-downtime updates with a single YAML file.
- Use `kubectl get all` to list all resources in a namespace at once.

---

## Resources

- [Kubernetes Deployments Docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes ReplicaSet Docs](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Kubernetes Example Manifests (GitHub)](https://github.com/kubernetes/examples)
