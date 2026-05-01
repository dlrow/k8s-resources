# Kubernetes Services — Discovery, Load Balancing & Networking

> **Source:** [DAY-35 | EVERYTHING ABOUT KUBERNETES SERVICES](https://www.youtube.com/watch?v=xY6Ic7Igzck)  
> **Course:** Complete DevOps Course (Day 35)  
> **Instructor:** Abhishek

---

## Overview

Every Deployment should have a corresponding Service. This session explains **why Services exist**, the three problems they solve, and the three Service types available in Kubernetes.

Three advantages of Kubernetes Services:
1. **Load Balancing** — distribute traffic across multiple Pods
2. **Service Discovery** — track Pods by label, not by IP address
3. **Expose to external world** — make applications reachable outside the cluster

---

## Why Services Exist — The Problem

When you deploy an application as a Deployment, you get multiple Pod replicas. Without a Service, several problems arise:

### Problem 1: Dynamic IP Addresses

Kubernetes Pods are **ephemeral** — they can die and be recreated at any time (auto-healing). Each time a Pod restarts, it gets a **new IP address**.

```
Pod before crash:   172.16.3.4  ✓
Pod after restart:  172.16.3.8  ✗ (IP changed)
```

If you share Pod IP addresses with other teams, they will break every time a Pod restarts — even though your application is technically "up."

### Problem 2: No Load Balancing

With 3 Pod replicas and 3 teams accessing them:

```
Team 1  →  Pod 1  (172.16.3.4)
Team 2  →  Pod 2  (172.16.3.5)
Team 3  →  Pod 3  (172.16.3.6)
```

This is manual, error-prone, and doesn't balance load intelligently.

### Problem 3: No External Access

Pods have internal cluster IPs. External users cannot reach them without SSHing into the cluster — which is completely impractical for real applications.

---

## The Solution: Kubernetes Service

A Service sits in front of your Pods and provides a **stable endpoint** that never changes, regardless of what happens to individual Pods.

```
External User
      │
      ▼
  [ Service ]  ←── stable IP / DNS name
      │
      ├──▶ Pod 1 (172.16.3.4)
      ├──▶ Pod 2 (172.16.3.5)
      └──▶ Pod 3 (172.16.3.6)
```

The Service uses **kube-proxy** under the hood to manage routing rules on each node.

---

## Advantage 1: Load Balancing

The Service distributes incoming requests across all healthy Pods automatically.

Instead of teams accessing individual Pod IPs, everyone accesses the same Service endpoint:

```
payment.default.svc.cluster.local
```

The Service forwards requests round-robin (or via other strategies) to all Pods matching its selector.

---

## Advantage 2: Service Discovery via Labels & Selectors

This is the key insight — **Services track Pods by label, not by IP address**.

### How it works

Every Pod created by a Deployment inherits labels from the Pod template:

```yaml
# In your Deployment spec.template.metadata
labels:
  app: payment
```

All Pod replicas will have the label `app: payment`, even after they restart.

The Service is configured to select Pods with a matching label:

```yaml
# In your Service spec
selector:
  app: payment
```

Now:
- Pod restarts → IP changes → **label stays the same**
- Service watches for label `app: payment` → **always finds the right Pods**
- New replica added → inherits the same label → **Service picks it up automatically**

```
Pod crashes
    │
    ▼
ReplicaSet creates new Pod
    │
    ▼
New Pod gets same label (app: payment)
    │
    ▼
Service automatically routes to new Pod ✓
```

No manual IP tracking. No configuration changes. It just works.

---

## Advantage 3: Exposing to the External World

Services have three types that control **who can access your application**:

### Service Types

| Type | Who Can Access | Use Case |
|------|---------------|----------|
| `ClusterIP` | Only inside the Kubernetes cluster | Internal microservice-to-microservice communication |
| `NodePort` | Anyone with access to a worker node IP | Internal org / VPC access, dev/test environments |
| `LoadBalancer` | Anyone on the Internet (public IP) | Public-facing applications on cloud providers |

---

## Service Type Deep Dive

### Type 1: ClusterIP (Default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP  # default if omitted
```

- Creates a stable internal DNS name: `payment.default.svc.cluster.local`
- Only reachable **within** the Kubernetes cluster network
- Benefits: load balancing + service discovery
- Does **not** expose to the outside world

**When to use:** Backend services, databases, internal APIs that should not be publicly accessible.

---

### Type 2: NodePort

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # 30000–32767 range
```

- Opens a port on **every worker node** in the cluster
- Anyone who can reach a worker node IP can access the application:  
  `http://<worker-node-ip>:30080`
- Works on minikube and bare-metal clusters

**When to use:** Internal organization access, VPC-level access, development environments where you need external access without a cloud load balancer.

---

### Type 3: LoadBalancer

```yaml
spec:
  type: LoadBalancer
```

- **Cloud providers only** (AWS EKS, GCP GKE, Azure AKS)
- Kubernetes Cloud Controller Manager requests a **public IP / Elastic Load Balancer** from the cloud provider
- Returns a public IP address accessible from anywhere on the Internet
- Does **not** work on minikube by default (use NodePort or Ingress for local dev)

**When to use:** Public-facing applications that need to be accessible from the Internet.

> For local clusters (minikube), use NodePort or Ingress instead of LoadBalancer.

---

## Access Levels — Summary Diagram

```
Internet (Public)
        │
        │  LoadBalancer service type
        ▼
[ Cloud Load Balancer ] (public IP)
        │
        │  NodePort service type
        ▼
[ Worker Node IPs ] (VPC / org network only)
        │
        │  ClusterIP service type
        ▼
[ Kubernetes Cluster Network ] (pods / containers only)
        │
        ▼
      [ Pods ]
```

---

## Real-World Examples

| Application | Service Type | Reason |
|-------------|-------------|--------|
| amazon.com | LoadBalancer | Must be accessible from anywhere in the world |
| Internal payment API | ClusterIP | Only other microservices inside the cluster need to call it |
| Dev/staging environment | NodePort | Only engineers with VPC access need to test it |
| Database | ClusterIP | Should never be exposed externally |

---

## Labels & Selectors — YAML Example

**Deployment with labels:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment    # every Pod gets this label
    spec:
      containers:
        - name: payment-app
          image: payment:latest
          ports:
            - containerPort: 8080
```

**Service selecting those Pods:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment
spec:
  selector:
    app: payment        # matches the Pod label above
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Once created, other services in the cluster can reach it at:
```
http://payment.default.svc.cluster.local
```

---

## Useful Commands

```bash
# List all services
kubectl get svc

# Describe a service (see endpoints/IPs it's routing to)
kubectl describe svc payment

# Create or update a service
kubectl apply -f service.yaml

# See pod labels
kubectl get pods --show-labels

# Get pods matching a specific label
kubectl get pods -l app=payment

# Check endpoints (actual pod IPs the service routes to)
kubectl get endpoints payment
```

---

## Key Takeaways

- **Every Deployment should have a Service** — Pods without a Service are hard to reach reliably.
- Services solve three problems: **load balancing**, **service discovery**, and **external exposure**.
- Services use **labels and selectors**, not IP addresses — this is what makes them resilient to Pod restarts.
- `ClusterIP` = internal only, `NodePort` = org/VPC access, `LoadBalancer` = public Internet.
- `LoadBalancer` type requires a cloud provider; use `NodePort` for local/minikube testing.
- kube-proxy is the underlying component that implements Service routing on each node.
- **Ingress** (covered in a future class) is the recommended way to expose HTTP services publicly with path/host-based routing.

---

## Resources

- [Kubernetes Services Docs](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
