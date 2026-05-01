# Kubernetes Pods — Deploy Your First Application

> **Source:** [DAY-33 | KUBERNETES PODS | DEPLOY YOUR FIRST APP](https://www.youtube.com/watch?v=-rDT9m1RKSA)  
> **Course:** Complete DevOps Course (Day 33)  
> **Instructor:** Abhishek

---

## Overview

This session introduces the foundational Kubernetes deployment unit — the **Pod** — and walks through deploying your first application on a local Kubernetes cluster.

Topics covered:
- What is a Pod and why it exists
- Pod vs. Docker container
- Single vs. multi-container Pods
- Installing `kubectl` and `minikube`
- Deploying, accessing, and debugging a Pod
- Key `kubectl` commands

---

## Why Kubernetes Uses Pods (Not Bare Containers)

Kubernetes does not run containers directly. Instead, it wraps containers inside a **Pod**.

### The Docker Way (CLI flags)

```bash
docker run -d \
  --name nginx \
  -p 80:80 \
  -v /data:/data \
  --network my-net \
  nginx:1.14.2
```

### The Kubernetes Way (YAML manifest)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

The Pod YAML is simply a **structured, declarative version** of the same `docker run` command. It lives in a Git repository, is version-controlled, and is human-readable at a glance — this is what Kubernetes means by *declarative* infrastructure.

**Key insight:** An IP address is assigned to the **Pod**, not to the individual container inside it. (kube-proxy manages this.)

---

## What is a Pod?

> A Pod is a **definition of how to run a container** (or a group of containers) in Kubernetes.

- The smallest deployable unit in Kubernetes
- Always defined as a YAML manifest
- Can contain **one or more containers**
- All containers in a Pod share:
  - **Network** — communicate via `localhost`
  - **Storage** — can share volumes

### Single Container Pod (most common)

The typical use case — one application, one container, one Pod.

### Multi-Container Pod (advanced)

Used when containers are tightly coupled and must share network/storage:

| Pattern | Description |
|---------|-------------|
| **Sidecar** | A helper container running alongside the main app (e.g., Envoy proxy in Istio) |
| **Init container** | Runs to completion before the main container starts (e.g., to pre-load config) |

Example: In a service mesh, an Envoy sidecar container intercepts all traffic. Since it shares localhost with the main app, no network configuration is needed.

---

## Why YAML Over CLI?

| Concern | CLI (`docker run`) | Kubernetes YAML |
|---------|-------------------|-----------------|
| **Readability** | Hard to read all flags | Clear key-value structure |
| **Version control** | No | Yes (stored in Git) |
| **Declarative** | No | Yes |
| **Reusable** | No | Yes |
| **Team collaboration** | Difficult | Easy |

In production, every team member can open `pod.yaml` and immediately understand what container is running, on which port, with what volumes and network config.

---

## Kubernetes Tooling

### kubectl

`kubectl` is the **command-line interface for Kubernetes** — the equivalent of the `docker` CLI.

```bash
# Check version
kubectl version

# View cluster nodes
kubectl get nodes

# View pods
kubectl get pods

# View deployments
kubectl get deployments
```

### minikube

`minikube` creates a **single-node Kubernetes cluster** locally for development and learning.

- Creates a VM (on Mac/Windows) and runs Kubernetes inside it
- Uses Docker as the default driver on most systems
- For Mac with Apple Silicon: use `--driver=hyperkit`

> **Note:** minikube is for learning only. Production clusters have multiple master nodes and many worker nodes.

**Local alternatives:**
- `kind` (Kubernetes in Docker) — preferred for advanced users, can simulate multi-node clusters
- `k3s`, `microk8s`

---

## Installation

### Install kubectl

```bash
# Visit: https://kubernetes.io/docs/tasks/tools/
# Choose your OS and architecture, then run the provided commands

# Verify installation
kubectl version --client
```

### Install minikube

```bash
# Visit: https://minikube.sigs.k8s.io/docs/start/
# Choose your OS and architecture, then run the provided commands

# Start a cluster
minikube start

# With explicit driver (Mac with Apple Silicon)
minikube start --memory=4096 --driver=hyperkit
```

After starting, verify the cluster node is ready:

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.x.x
```

---

## Deploying Your First Pod

### Step 1 — Write the Pod YAML

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

Replace `nginx:1.14.2` with any container image from your previous Docker projects.

### Step 2 — Create the Pod

```bash
kubectl apply -f pod.yaml
# pod/nginx created
```

### Step 3 — Verify it's running

```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          10s

# Get full details including IP address
kubectl get pods -o wide
```

### Step 4 — Access the application

Pods have cluster-internal IPs. To access from your machine, SSH into the cluster first:

```bash
# For minikube
minikube ssh

# For a real cluster (SSH to master or worker node)
ssh -i <key.pem> ubuntu@<node-ip>

# Then curl the pod IP
curl 172.17.0.3   # replace with actual pod IP
# → Welcome to nginx!
```

### Step 5 — Delete the Pod

```bash
kubectl delete pod nginx
```

---

## Debugging Pods

Two essential commands for troubleshooting:

### 1. `kubectl describe pod`

Shows **complete details** of a Pod — current state, events, errors, resource limits, volumes, etc.

```bash
kubectl describe pod nginx
```

Use this when:
- Pod is stuck in `Pending` or `CrashLoopBackOff`
- You need to see why a Pod failed to schedule
- You want to inspect labels, node assignment, container spec

### 2. `kubectl logs`

Shows **application logs** from the container inside the Pod.

```bash
kubectl logs nginx

# Follow logs in real-time
kubectl logs nginx -f

# If the pod has multiple containers
kubectl logs nginx -c <container-name>
```

Use this when:
- The app is throwing errors at runtime
- You want to verify the app started correctly
- You need to see HTTP request logs

> **Interview tip:** When asked "how do you debug a Pod?", answer:  
> `kubectl describe pod <name>` for cluster-level info, `kubectl logs <name>` for application-level logs.

---

## Essential kubectl Commands

```bash
# List all resources in the current namespace
kubectl get all

# List all resources in all namespaces
kubectl get all -A

# Create/update from a YAML file
kubectl apply -f pod.yaml

# Create (fails if already exists)
kubectl create -f pod.yaml

# Get pods with full details
kubectl get pods -o wide

# Describe a pod
kubectl describe pod <pod-name>

# View pod logs
kubectl logs <pod-name>

# Stream logs live
kubectl logs <pod-name> -f

# Delete a pod
kubectl delete pod <pod-name>

# Execute a command inside a running pod
kubectl exec -it <pod-name> -- /bin/bash
```

> **Reference:** [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) — bookmark this. Even experienced engineers use it daily.

---

## Pod vs. Deployment

| | Pod | Deployment |
|--|-----|------------|
| **Auto-healing** | No | Yes |
| **Auto-scaling** | No | Yes |
| **Zero-downtime updates** | No | Yes |
| **Production use** | No | Yes |

A Pod is the foundation you must understand, but in production you will **always use a Deployment** — which is simply a wrapper around Pods that adds auto-healing and scaling. This is covered in Day 34.

---

## Key Takeaways

- A **Pod** is a YAML-based specification for running one or more containers.
- Pods are the **lowest-level deployable unit** in Kubernetes — you can't deploy a bare container.
- Kubernetes assigns an **IP to the Pod**, not to the container.
- Multi-container Pods share **network (localhost)** and **storage** — used for sidecars and init containers.
- **`kubectl`** is to Kubernetes what `docker` CLI is to Docker.
- **`minikube`** creates a local single-node cluster for learning.
- Two debug commands: `kubectl describe pod` (cluster info) and `kubectl logs` (app logs).
- **Never memorize YAML syntax** — always reference the official docs or the kubectl cheat sheet.
- Pods alone have no auto-healing — for that, you need a **Deployment** (Day 34).

---

## Resources

- [Kubernetes Pods Documentation](https://kubernetes.io/docs/concepts/workloads/pods/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Install minikube](https://minikube.sigs.k8s.io/docs/start/)
