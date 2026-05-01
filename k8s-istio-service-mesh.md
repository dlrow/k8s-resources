# Istio & Service Mesh — Zero to Hero

> **Source:** [Service Mesh explained in 60 minutes | Istio mTLS and Canary Demo](https://www.youtube.com/watch?v=eSNetKBe7Z8)  
> **Instructor:** Abhishek

---

## Overview

Topics covered:
- What is a service mesh and why it exists
- East-West vs. North-South traffic
- How Istio works — the Envoy sidecar model
- Istio architecture: control plane vs. data plane
- How Istio fits inside Kubernetes (CRDs, controllers, admission webhooks)
- Admission controllers (static) and dynamic admission webhooks
- All Istio custom resources: VirtualService, DestinationRule, Gateway, PeerAuthentication, AuthorizationPolicy, ServiceEntry
- Mutual TLS — permissive and strict modes
- Traffic management: Canary, Blue-Green, A/B testing
- Circuit breaking, timeouts, retries
- Observability with Kiali
- Istio Gateway vs. Kubernetes Ingress
- Does Istio have controllers?

---

## 1. Traffic Types in Kubernetes

Before understanding Istio, understand the two directions traffic flows:

```
                        ┌──────────────────────────────────┐
Internet ─────────────▶ │       Kubernetes Cluster         │
(North-South)           │                                  │
                        │  Login ◀──────────▶ Catalog      │
                        │    │         (East-West)  │       │
                        │    ▼                      ▼       │
                        │ Payment ◀──────▶ Notification    │
                        └──────────────────────────────────┘
```

| Traffic Type | Direction | Example | Managed by |
|-------------|-----------|---------|------------|
| **North-South** | External → cluster or cluster → external | User → Login service via Ingress | Ingress / Gateway |
| **East-West** | Service → service within the cluster | Catalog → Payment → Notification | Kubernetes Service (basic) or Istio (advanced) |

**Service mesh primarily manages East-West traffic.** This is the communication that happens silently between your microservices — and it's where things go wrong in production (latency, partial failures, security gaps).

---

## 2. Why Service Mesh? The Problem

Kubernetes Services give you east-west communication, but only at a basic level. As microservice architectures grow, you need answers to:

- How do I know if service A's call to service B is failing? (observability)
- How do I prevent a failing service B from cascading to bring down A? (circuit breaking)
- How do I gradually roll out a new version of service B? (canary deployment)
- How do I secure service-to-service communication? (mTLS)
- How do I add retries and timeouts without modifying application code?

**Without a service mesh**, developers must implement all of this inside every application — in every language, every team, every service. This doesn't scale.

**With a service mesh**, all of this is handled at the infrastructure layer — transparently, without application code changes.

---

## 3. What is Istio?

Istio is a **service mesh** — an infrastructure layer that adds the following capabilities to east-west (and north-south) traffic in a Kubernetes cluster:

| Capability | Description |
|-----------|-------------|
| **Mutual TLS (mTLS)** | Automatic certificate-based encryption and authentication between every service |
| **Traffic management** | Canary, A/B, Blue-Green deployments; weight-based routing |
| **Circuit breaking** | Stop cascading failures by cutting off a failing service |
| **Request timeouts & retries** | Configurable per-route, without touching application code |
| **Observability** | Automatic metrics, distributed tracing, service graph (via Kiali) |
| **Access control** | Authorization policies — who can call whom |
| **Traffic mirroring** | Shadow traffic to a new version for testing without affecting users |

---

## 4. How Istio Works — The Envoy Sidecar

Istio's core mechanism: **inject an Envoy proxy container into every Pod, alongside the application container.**

```
┌─────────────────────────────────────────────────────────┐
│  Pod: Catalog Service                                   │
│  ┌─────────────────┐     ┌────────────────────────┐    │
│  │   App Container  │◀───▶│  Sidecar (Envoy Proxy) │    │
│  │  (catalog:v1)   │     │   istio-proxy          │    │
│  └─────────────────┘     └────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Every request entering or leaving the Pod goes through the Envoy sidecar — not the app container directly.**

### What the sidecar does

- **Intercepts all inbound and outbound traffic** using iptables rules (injected at pod startup)
- **Enforces mTLS** — presents and verifies certificates automatically
- **Applies routing rules** — traffic weights, retries, timeouts (configured by istiod)
- **Reports telemetry** — sends metrics, traces, and access logs to istiod / Prometheus / Jaeger
- **Implements circuit breaking** — stops sending requests to unhealthy upstream services

### The key insight

Developers **do not change their application code**. They continue to make plain HTTP calls between services. The sidecar transparently upgrades those calls to mTLS, applies routing rules, retries, etc.

---

## 5. Istio Architecture — Control Plane vs. Data Plane

```
┌──────────────────────────────────────────────────────────────────┐
│  CONTROL PLANE                                                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                       istiod                            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │    │
│  │  │  Pilot   │  │ Citadel  │  │       Galley         │  │    │
│  │  │(traffic  │  │  (PKI/   │  │  (config validation) │  │    │
│  │  │ config)  │  │  certs)  │  └──────────────────────┘  │    │
│  │  └──────────┘  └──────────┘                            │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                              │
                     xDS protocol
                    (config push)
                              │
┌──────────────────────────────────────────────────────────────────┐
│  DATA PLANE                                                      │
│                                                                  │
│  Pod A: [App] ◀▶ [Envoy]  ←──────→  Pod B: [App] ◀▶ [Envoy]   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### istiod (Control Plane)

`istiod` is a single binary that merged three historically separate components:

| Sub-component | Responsibility |
|--------------|----------------|
| **Pilot** | Reads Istio CRDs (VirtualService, DestinationRule, etc.), converts them into Envoy configuration, and pushes via xDS protocol to every sidecar |
| **Citadel** | Acts as a Certificate Authority (CA). Issues, rotates, and revokes TLS certificates for every service in the mesh. Enables mTLS. |
| **Galley** | Validates Istio configuration (CRDs) before they're applied. Ensures you don't push invalid configs to sidecars. |

### Envoy Sidecar (Data Plane)

- Runs as `istio-proxy` container inside every Pod
- Receives configuration from istiod via **xDS API** (Envoy's dynamic configuration protocol)
- Actually executes: traffic routing, mTLS, circuit breaking, metrics collection
- **Does not call istiod per-request** — configuration is pushed and cached locally

### xDS — How Config Reaches the Sidecar

xDS is a set of APIs Envoy uses to receive dynamic configuration:

| API | Purpose |
|-----|---------|
| **LDS** (Listener Discovery Service) | What ports/protocols to listen on |
| **RDS** (Route Discovery Service) | Routing rules (VirtualService maps here) |
| **CDS** (Cluster Discovery Service) | Upstream service endpoints (DestinationRule maps here) |
| **EDS** (Endpoint Discovery Service) | Actual pod IP:port for each cluster |

When you apply a `VirtualService`, istiod translates it into xDS config and pushes it to all affected Envoy sidecars — no Pod restart required.

---

## 6. How Istio Fits Inside Kubernetes

### Istio extends Kubernetes via CRDs

Istio does NOT modify Kubernetes internals. It fits into Kubernetes using the standard extension mechanisms:

1. **CRDs** — Istio installs its own Custom Resource Definitions. You create Istio resources the same way you create Pods or Services — `kubectl apply -f`.
2. **Controllers** — istiod is a Kubernetes controller. It watches Istio CRDs and acts on them (pushes xDS config to sidecars).
3. **Admission Webhooks** — istiod registers a mutating admission webhook so it can inject the Envoy sidecar into Pods at creation time.

```
kubectl apply -f virtual-service.yaml
        │
        ▼
   Kubernetes API Server
        │
        ├──▶ Stores VirtualService CR in etcd
        │
        ▼
   istiod (controller watches CRDs)
        │
        ▼
   Translates VirtualService → xDS config
        │
        ▼
   Pushes to all Envoy sidecars in the mesh
```

### Does Istio have controllers?

**Yes — istiod is the controller.** It follows the same Kubernetes controller pattern as the ReplicaSet controller or the Deployment controller:

- Watches Istio CRDs (VirtualService, DestinationRule, Gateway, etc.)
- Reconciles desired state (what the CRDs say) → actual state (Envoy sidecar config)
- Continuously re-syncs when CRDs change

---

## 7. Admission Controllers & How Istio Injects Sidecars

This is one of the most technically interesting parts of Istio — how does the sidecar get added to your Pod without you putting it in your YAML?

### Static Admission Controllers (Built-In)

Before a resource is persisted to etcd, Kubernetes runs **admission controllers** that can:
- **Mutate** (modify) the object — e.g., add a default `storageClass` to a PVC
- **Validate** (reject) the object — e.g., reject a Pod that exceeds a ResourceQuota

There are 30+ built-in admission controllers pre-compiled into the API server (DefaultStorageClass, ResourceQuota, NamespaceLifecycle, LimitRanger, etc.). These cannot be added by external projects.

### Request flow through the API server

```
kubectl apply -f pod.yaml
      │
      ▼
  API Server
      │
      ├── 1. Authentication (who are you?)
      ├── 2. Authorization (are you allowed?)
      ├── 3. Admission Controllers (mutate / validate)
      │         │
      │         ├── DefaultStorageClass controller (mutates PVC)
      │         ├── ResourceQuota controller (validates resource limits)
      │         └── MutatingAdmissionWebhook controller ──▶ istiod
      │
      └── 4. Persist to etcd
```

### Dynamic Admission Webhooks (How Istio Does It)

External projects (like Istio) cannot compile code into the API server. Instead, Kubernetes exposes two special admission controllers:

- **`MutatingAdmissionWebhook`** — forwards requests to an external HTTP webhook for mutation
- **`ValidatingAdmissionWebhook`** — forwards requests to an external HTTP webhook for validation

**Istio's sidecar injection flow:**

1. Istio installs a `MutatingWebhookConfiguration` CRD on the cluster:
   ```yaml
   kind: MutatingWebhookConfiguration
   metadata:
     name: istio-sidecar-injector
   webhooks:
     - name: sidecar-injector.istio.io
       rules:
         - operations: ["CREATE"]
           resources: ["pods"]       # fire on every Pod creation
       clientConfig:
         service:
           name: istiod
           namespace: istio-system
           path: /inject             # call this endpoint
   ```

2. When you create a Pod in a namespace labeled `istio-injection: enabled`:
   - API server sees the Pod creation request
   - `MutatingAdmissionWebhook` controller reads the `MutatingWebhookConfiguration`
   - It forwards the Pod spec to `istiod` at the `/inject` endpoint
   - istiod mutates the Pod spec — adds the `istio-proxy` sidecar container + init container
   - Returns the modified Pod spec to API server
   - API server persists the mutated Pod to etcd
   - kubelet starts the Pod — now with 2 containers (app + sidecar)

3. You never see the sidecar in your YAML — it is injected transparently at admission time.

```bash
# Enable sidecar injection for a namespace
kubectl label namespace default istio-injection=enabled

# Verify: each pod should show 2/2 containers
kubectl get pods
# NAME                        READY   STATUS    RESTARTS
# productpage-v1-xxx          2/2     Running   0         ← app + sidecar
```

---

## 8. Istio Custom Resources (CRDs)

These are the resources you create to configure Istio behavior. They are all standard Kubernetes resources (`kubectl apply` / `kubectl get` / `kubectl describe` all work).

### 8.1 VirtualService

Defines **how traffic is routed** to a service — the "routing rules" layer.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews              # applies to traffic going to the "reviews" service
  http:
    - route:
        - destination:
            host: reviews
            subset: v1     # 80% to v1
          weight: 80
        - destination:
            host: reviews
            subset: v3     # 20% to v3 (canary)
          weight: 20
```

**Key VirtualService capabilities:**
- Weight-based traffic splitting (canary deployments)
- Host-based and path-based routing
- Header-based routing (route specific users to a version)
- Fault injection (for chaos testing)
- Request timeouts: `timeout: 5s`
- Retries: `retries: { attempts: 3, perTryTimeout: 2s }`

### 8.2 DestinationRule

Defines **what happens after routing** — load balancing policy, circuit breaking, mTLS mode, and **subsets** (groups of pods by label).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100      # circuit breaker threshold
    outlierDetection:
      consecutiveErrors: 5       # eject pod after 5 consecutive errors
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1              # selects pods with label version=v1
    - name: v3
      labels:
        version: v3
```

**Subsets** are how VirtualService and DestinationRule connect: VirtualService says "route to subset v1", DestinationRule defines what "subset v1" means (which pods).

**Together, VirtualService + DestinationRule = complete traffic management.**

### 8.3 Gateway

Manages **ingress and egress** at the mesh boundary — the entry/exit point for traffic coming into or leaving the service mesh.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway      # use Istio's ingress gateway pod
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "bookinfo.example.com"
```

A Gateway alone doesn't route traffic — it must be paired with a VirtualService that references it:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "bookinfo.example.com"
  gateways:
    - bookinfo-gateway          # bind to the gateway above
  http:
    - route:
        - destination:
            host: productpage
            port:
              number: 9080
```

### 8.4 PeerAuthentication

Controls the **mTLS mode** for a namespace or workload:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT    # PERMISSIVE (default) | STRICT | DISABLE
```

| Mode | Behavior |
|------|---------|
| `PERMISSIVE` | Accepts both mTLS and plain HTTP — used during migration |
| `STRICT` | Only accepts mTLS — rejects plain HTTP connections |
| `DISABLE` | Disables mTLS entirely |

### 8.5 AuthorizationPolicy

Controls **who can call whom** — service-level access control:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/productpage"]
      to:
        - operation:
            methods: ["GET"]
```

This says: only the `productpage` service account can call `reviews`, and only with GET requests.

### 8.6 ServiceEntry

Registers **external services** (outside the mesh) so that Istio can manage traffic to them:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-payment-api
spec:
  hosts:
    - api.payment-provider.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

Without a ServiceEntry, Istio blocks all egress traffic to unknown external hosts by default (when egress is restricted). With one, you can apply VirtualService and DestinationRule rules to external traffic too.

### 8.7 Sidecar (resource)

Scopes which services a sidecar proxy is aware of — reduces memory footprint in large meshes:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: default
spec:
  egress:
    - hosts:
        - "./*"              # only know about services in same namespace
        - "istio-system/*"
```

---

## 9. Key Features In Depth

### 9.1 Mutual TLS (mTLS)

Standard TLS: client verifies server certificate.  
**Mutual TLS**: both client AND server verify each other's certificates.

**How Istio implements it (automatically):**
1. istiod's Citadel component acts as a CA
2. On pod startup, istiod issues a certificate to each sidecar (SPIFFE/X.509 identity tied to the pod's ServiceAccount)
3. When service A calls service B, A's sidecar initiates a TLS handshake and presents its certificate
4. B's sidecar verifies A's certificate against the CA, then presents its own certificate back
5. A's sidecar verifies B's certificate
6. Only if both trust each other does the connection proceed

**Zero application code change required** — your app thinks it's making a plain HTTP call.

```bash
# Test mTLS is working: this should fail from outside the mesh
minikube ssh
curl http://productpage:9080/   # connection reset by peer (STRICT mode)
```

### 9.2 Traffic Management — Canary Deployment

```
Initial state:   100% → reviews-v1
Canary phase 1:   90% → reviews-v1, 10% → reviews-v2
Canary phase 2:   50% → reviews-v1, 50% → reviews-v2
Stable:          100% → reviews-v2  (v1 removed)
```

This is controlled entirely by editing the VirtualService `weight` values — no redeployment, no scaling, no code change.

### 9.3 Circuit Breaking

Prevents a slow/failing service from cascading failures to its callers.

Configured in DestinationRule:

```yaml
trafficPolicy:
  connectionPool:
    http:
      http1MaxPendingRequests: 100
      http2MaxRequests: 1000
  outlierDetection:
    consecutiveGatewayErrors: 5   # trip circuit after 5 errors
    interval: 10s                  # check every 10s
    baseEjectionTime: 30s          # eject pod for 30s
    maxEjectionPercent: 50         # eject at most 50% of pods
```

When the circuit trips, Envoy returns an error immediately to the caller instead of waiting for the slow service — protecting the rest of the system.

### 9.4 Request Timeouts & Retries

```yaml
# In VirtualService
http:
  - timeout: 3s               # fail the request after 3s
    retries:
      attempts: 3             # retry up to 3 times
      perTryTimeout: 1s       # each attempt times out in 1s
      retryOn: 5xx,reset      # retry on server errors or connection reset
    route:
      - destination:
          host: ratings
```

### 9.5 Fault Injection (Chaos Testing)

Inject artificial failures to test resilience, without touching application code:

```yaml
http:
  - fault:
      delay:
        percentage:
          value: 50.0
        fixedDelay: 5s        # delay 50% of requests by 5s
      abort:
        percentage:
          value: 10.0
        httpStatus: 503       # abort 10% of requests with 503
    route:
      - destination:
          host: ratings
```

### 9.6 Observability with Kiali

Kiali is Istio's built-in service graph and observability dashboard:

```bash
# Install Kiali (included in Istio samples)
kubectl apply -f samples/addons/kiali.yaml

# Open dashboard
istioctl dashboard kiali
```

Because every request goes through a sidecar, Kiali knows:
- Which services are calling which
- Success/error rates per route
- P50/P95/P99 latency
- mTLS status between services
- Live traffic flow graph

Additional integrations available:
- **Prometheus** — metrics storage
- **Grafana** — dashboards
- **Jaeger / Zipkin** — distributed tracing

---

## 10. Istio Gateway vs. Kubernetes Ingress

| Aspect | Kubernetes Ingress | Istio Gateway |
|--------|-------------------|---------------|
| **Resource type** | Built-in K8s API (`networking.k8s.io/v1`) | Istio CRD (`networking.istio.io/v1alpha3`) |
| **Controller** | Separate Ingress Controller (nginx, etc.) | Istio IngressGateway (Envoy pod) |
| **Routing rules** | Defined in Ingress resource itself | Defined in VirtualService (bound to Gateway) |
| **mTLS** | Requires manual TLS cert setup | Automatic via Istio |
| **Traffic management** | Limited (annotations, controller-specific) | Full Istio features (canary, retries, circuit breaking) |
| **East-West traffic** | No — only north-south | Yes — same VirtualService rules apply internally |
| **Egress control** | No | Yes — EgressGateway controls outbound traffic |

**When to use Istio Gateway instead of Kubernetes Ingress:**
- You already have Istio installed
- You want the same routing rules (VirtualService) to apply to both internal and external traffic
- You need advanced features (mTLS at the edge, traffic mirroring, fault injection at ingress)

**Istio Gateway + VirtualService** gives you a unified traffic management layer for all traffic — north-south and east-west — using the same configuration model.

---

## 11. Installation

### Install Istio CLI

```bash
# Download and extract istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH
```

### Install Istio on Cluster

```bash
# Demo profile — suitable for learning
istioctl install --set profile=demo -y

# Minimal profile — production baseline
istioctl install --set profile=minimal -y

# Verify installation
kubectl get pods -n istio-system
# istiod, istio-ingressgateway, istio-egressgateway
```

### Enable Sidecar Injection

```bash
# Label namespace for automatic injection
kubectl label namespace default istio-injection=enabled

# Verify label is set
kubectl get namespace default --show-labels

# Pods must be recreated after labeling
# Existing pods don't get sidecars — restart them:
kubectl rollout restart deployment <name>
```

### Verify Sidecars Are Running

```bash
kubectl get pods
# Each pod should show 2/2 READY — app + istio-proxy
```

---

## 12. The Full Picture — How Everything Connects

```
User creates: kubectl apply -f virtual-service.yaml
                │
                ▼
        Kubernetes API Server
                │
         stored in etcd
                │
                ▼
    istiod (controller) detects change
    via Kubernetes watch mechanism
                │
    Translates VirtualService + DestinationRule
    → xDS configuration (LDS/RDS/CDS/EDS)
                │
                ▼
    Pushes config to all Envoy sidecars
    in affected namespaces
                │
                ▼
    Next request from Catalog → Payment:
      Catalog app → [Catalog Envoy sidecar]
                          │ mTLS handshake
                          │ applies routing rules
                          │ records metrics
                          ▼
                  [Payment Envoy sidecar] → Payment app
```

---

## 13. Key Takeaways & Interview Answers

### What is a service mesh?
An infrastructure layer that manages service-to-service (east-west) communication — adding mTLS, traffic management, observability, and resilience without changing application code.

### What is east-west vs. north-south traffic?
- **North-South:** external → cluster (handled by Ingress/Gateway)
- **East-West:** service → service within the cluster (handled by service mesh)

### How does Istio inject sidecars?
Via a **Mutating Admission Webhook**. istiod registers a `MutatingWebhookConfiguration` that tells Kubernetes to forward every Pod creation request to istiod's `/inject` endpoint. istiod mutates the Pod spec to add the Envoy sidecar before the Pod is stored in etcd.

### Does Istio have controllers?
Yes — **istiod is the controller**. It watches Istio CRDs (VirtualService, DestinationRule, etc.) and pushes translated xDS configuration to all Envoy sidecars.

### VirtualService vs. DestinationRule?
- **VirtualService** = routing rules (where does traffic go, what weights, what timeouts)
- **DestinationRule** = destination behavior (which pods are in a subset, circuit breaking thresholds, mTLS policy)
- They work together: VirtualService routes to a subset name, DestinationRule defines what that subset is.

### Istio Gateway vs. Kubernetes Ingress?
Gateway is Istio's equivalent of Ingress — but it's more powerful (unified with VirtualService, supports full Istio feature set) and requires Istio to be installed. Standard Ingress works independently with any Ingress Controller.

### mTLS permissive vs. strict?
- **Permissive:** accepts both mTLS and plain HTTP (safe during migration)
- **Strict:** only accepts mTLS (full security, rejects unencrypted connections)

### What is xDS?
The protocol istiod uses to push configuration to Envoy sidecars. Sidecars don't pull config — istiod pushes it whenever Istio CRDs change. No Pod restart needed.

---

## Resources

- [Istio Official Documentation](https://istio.io/latest/docs/)
- [Istio Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Kubernetes Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Kubernetes Admission Controllers Reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Envoy xDS API](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
- [Kiali — Istio Observability](https://kiali.io/)
- [SPIFFE — Service Identity Standard](https://spiffe.io/)
