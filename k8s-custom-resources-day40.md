# Kubernetes Custom Resources, CRDs & Custom Controllers — Deep Dive

> **Source:** [DAY-40 | KUBERNETES CUSTOM RESOURCES | CUSTOM CONTROLLER | DEEP DIVE](https://www.youtube.com/watch?v=alGEPSQxbLg)  
> **Course:** Complete DevOps Course (Day 40)  
> **Instructor:** Abhishek

---

## Overview

This session covers three core concepts for extending Kubernetes:

1. **Custom Resource Definition (CRD)** — defines a new API type in Kubernetes
2. **Custom Resource (CR)** — an instance of that new API type submitted by a user
3. **Custom Controller** — watches for CRs and acts on them

---

## Why Extend the Kubernetes API?

Kubernetes ships with native resources out of the box: `Deployment`, `Service`, `Pod`, `ConfigMap`, `Secret`, `Ingress`, etc.

However, the ecosystem has grown far beyond these primitives. Many tools and platforms need to introduce their own resource types into the cluster:

| Tool | Purpose |
|------|---------|
| **Istio** | Service mesh, Mutual TLS, east-west traffic routing |
| **Argo CD** | GitOps continuous delivery |
| **Keycloak** | Identity & Access Management, OAuth/OIDC |
| **Kyverno / kube-bench** | Security policies and compliance |
| **Prometheus** | Metrics and monitoring |
| **Crossplane** | Infrastructure-as-code via Kubernetes |
| **Flux / Spinnaker** | Continuous delivery pipelines |

Kubernetes cannot bake in the logic for every possible third-party tool — there are thousands of them across the CNCF ecosystem. Instead, Kubernetes provides a mechanism to **extend its API**, letting anyone add new resource types without touching the core control plane.

---

## The Three Actors

| Actor | Responsibility |
|-------|---------------|
| **DevOps Engineer** | Deploys the CRD and the Custom Controller |
| **User (Developer / DevOps)** | Creates Custom Resources (instances) |

---

## Concept 1: Custom Resource Definition (CRD)

A CRD is a YAML file submitted to Kubernetes that **defines a new API type**.

**Analogy with native resources:**

- When you write a `deployment.yaml`, Kubernetes validates it against its internal `Deployment` resource definition.
- If you add an unknown field (e.g., `xyz`), Kubernetes immediately returns:  
  `error: field xyz is not known`
- This validation happens because Kubernetes knows the exact schema for `Deployment`.

**With custom resources**, the same validation model applies — but *you* (or the tool vendor) must first provide the schema by submitting a CRD.

```
CRD  ──→  defines the schema / allowed fields for a Custom Resource
CR   ──→  an instance created by a user, validated against the CRD
```

**Example (Istio Virtual Service):**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService        # Custom Resource Kind
metadata:
  name: my-vs
spec:
  # fields defined by Istio's CRD
  ...
```

The `VirtualService` CR is validated against the `VirtualService` CRD that the Istio team published.

---

## Concept 2: Custom Resource (CR)

A CR is an **instance** of a CRD — exactly like how a `Deployment` object is an instance of Kubernetes' built-in `Deployment` resource definition.

- Users look up the tool's documentation to find the correct CR YAML format.
- The Kubernetes API server intercepts the request, validates it against the corresponding CRD, and (if valid) stores it in etcd.
- **At this point, nothing happens yet** — the CR just sits in the cluster.

---

## Concept 3: Custom Controller

This is the "brain" that watches for Custom Resources and **acts on them**.

**Analogy:**
- A `Deployment` is acted upon by the `DeploymentController` (built into Kubernetes).
- A CR has no built-in controller — you must deploy one separately.

**Without a custom controller:** deploying a CR is like deploying an `Ingress` resource without an Ingress controller — it does nothing.

**The controller's job:**
1. Watch for create / update / delete events on the CR
2. Reconcile the desired state expressed in the CR
3. Take the corresponding action (e.g., configure Envoy sidecars for Istio)

---

## End-to-End Flow

```
DevOps Engineer
    │
    ├─ 1. Deploy CRD  ────────────────────────────────┐
    │   (from tool docs, via Helm / manifest / operator) │
    │                                                   ▼
    │                                    Kubernetes API extended with
    │                                    new resource type (e.g. VirtualService)
    │
    ├─ 2. Deploy Custom Controller ───────────────────┐
    │   (from tool docs, via Helm / manifest / operator) │
    │                                                   ▼
    │                                    Controller running in cluster,
    │                                    watching for CRs
    │
User / Developer
    │
    └─ 3. Deploy Custom Resource (CR) ───────────────▶ Validated against CRD
                                                        Controller detects CR
                                                        Controller performs action
```

---

## Comparison: Native vs. Custom Resources

| Aspect | Native (e.g., Deployment) | Custom (e.g., Istio VirtualService) |
|--------|--------------------------|-------------------------------------|
| Resource Definition | Built into Kubernetes | CRD (deployed separately) |
| Resource Instance | `deployment.yaml` | Custom Resource YAML |
| Controller | Built-in `DeploymentController` | Custom Controller (deployed separately) |
| Who deploys schema | Kubernetes team | Tool vendor / DevOps engineer |

---

## Writing a Custom Controller

Most DevOps engineers will **never write** a custom controller, but it's useful to understand how they work.

### Preferred Language: Go (Golang)

- Kubernetes itself is written in Go.
- The primary client library is **`client-go`** — the official Go client for the Kubernetes API.
- Other language clients exist (Python, Java) but the Go ecosystem is by far the most mature.

### How a Controller Works (High Level)

1. **Set up Watchers** — register listeners on specific resource types (e.g., `VirtualService` create/update/delete events).
2. **Reflector** (from `client-go`) — detects changes via the watcher and enqueues them.
3. **Worker Queue** — a queue of resource objects that need to be reconciled.
4. **Reconcile Loop** — processes each item in the queue and applies the required configuration to the cluster.

### Useful Frameworks

| Framework | Description |
|-----------|-------------|
| [`controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime) | Go library maintained by the Kubernetes community; simplifies controller/operator development |
| [Operator SDK](https://sdk.operatorframework.io/) | Higher-level framework for writing Kubernetes Operators |
| [`kubernetes/sample-controller`](https://github.com/kubernetes/sample-controller) | Official Kubernetes example of a custom controller |

---

## Practical Demo: Istio via Helm

The demo walks through installing Istio (CRD + controller) using Helm:

```bash
# Add the Istio Helm repo
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# Create the istio-system namespace
kubectl create namespace istio-system

# Install Istio base (deploys CRDs)
helm install istio-base istio/base -n istio-system

# Verify CRDs were created
kubectl get crd | grep istio
```

After installation, `kubectl get crd` will show all Istio-related CRDs (e.g., `virtualservices.networking.istio.io`, `destinationrules.networking.istio.io`, etc.).

---

## DevOps Engineer Responsibilities

When your organization adopts a tool that uses CRDs (Istio, Argo CD, Prometheus, etc.):

1. **Deploy the CRD** — follow the tool's installation docs (Helm chart, plain manifests, or operator).
2. **Deploy the Custom Controller** — usually bundled in the same Helm chart.
3. **Support users** creating their own Custom Resources.
4. **Debug issues** — check controller logs, describe CR status, verify CR spec against the CRD.

> Writing controllers or CRDs from scratch is typically only required in roles where you contribute to or build Kubernetes-native tooling (e.g., open-source contributors, platform engineering teams).

---

## Key Takeaways

- **CRD** = schema definition for a new Kubernetes resource type (extend the API).
- **CR** = instance of that new type (submitted by a user).
- **Custom Controller** = the process that watches CRs and acts on them.
- Without a controller, a CR is inert — just data stored in etcd.
- The CNCF landscape is almost entirely composed of custom Kubernetes controllers.
- Go / `client-go` is the dominant language for writing controllers.
- As a DevOps engineer, your primary job is **deployment and debugging**, not writing controllers.

---

## Resources

- [Kubernetes Sample Controller (GitHub)](https://github.com/kubernetes/sample-controller)
- [controller-runtime (GitHub)](https://github.com/kubernetes-sigs/controller-runtime)
- [Operator SDK](https://sdk.operatorframework.io/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [Kubernetes CRD Docs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
