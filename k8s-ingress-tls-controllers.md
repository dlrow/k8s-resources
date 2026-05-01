# Kubernetes Ingress, TLS & Ingress Controllers — Deep Dive

> **Sources:**  
> - [Kubernetes Service, Ingress with TLS and Ingress Controllers with Live coding](https://www.youtube.com/watch?v=3YTU4EPjEh4) (index 10)  
> - [DAY-38 | KUBERNETES INGRESS](https://www.youtube.com/watch?v=47ck6bh6dfI) (index 11)  
> **Course:** Complete DevOps Course  
> **Instructor:** Abhishek

---

## Overview

This session picks up from Services and explains why Ingress exists, how Ingress Controllers work, and the three TLS termination strategies you must understand for production.

Topics covered:
- History: why Ingress was introduced in Kubernetes 1.1
- The VM-to-Kubernetes migration pain point
- Two core problems Ingress solves (important interview Q)
- What Ingress is and how it works
- Ingress Controllers — what they do and popular options
- Ingress Classes — running multiple controllers in one cluster
- One Ingress for many services (not 1:1)
- Host-based and path-based routing (with live demo)
- Wildcard host matching
- Basic authentication via annotations
- TLS termination: SSL Passthrough vs. SSL Offloading vs. SSL Bridging
- OpenShift Routes vs. Kubernetes Ingress
- Local testing with minikube and /etc/hosts

---

## History — Why Ingress Was Introduced

Before **Kubernetes 1.1 (released late 2015)**, Ingress did not exist. Teams running Kubernetes used only Services to expose their applications.

When organizations started migrating from **virtual machines / physical servers** to Kubernetes, they brought their existing stack — enterprise load balancers like nginx, F5, HAProxy, and Citrix. These load balancers had rich capabilities that went far beyond what Kubernetes Services could offer.

Users filed issues on the Kubernetes GitHub page: *"On VMs we had all these capabilities — now on Kubernetes we've lost them."* OpenShift had already shipped its own answer (**OpenShift Routes**) which acted as a catalyst. Kubernetes eventually agreed the problem was real and introduced the Ingress resource + the Ingress Controller architecture.

---

## Quick Recap: Services and Their Limitations

### The Dynamic IP Problem (recap)

Pods get a new IP every time they restart. A Service solves this via labels/selectors — it provides a stable DNS name and load balances across Pods.

### Three Service Types (recap)

| Type | Who can access |
|------|---------------|
| `ClusterIP` | Only within the Kubernetes cluster |
| `NodePort` | Anyone with access to a worker node IP |
| `LoadBalancer` | Anyone on the Internet (cloud providers only) |

---

## The Two Problems Ingress Solves

> **These are high-frequency interview questions.** Know both problems cold.

### Problem 1 — Kubernetes Services only do simple round-robin

Kubernetes Services (via kube-proxy + iptables) implement basic round-robin load balancing: 10 requests split evenly across available Pods. That's it.

Enterprise load balancers that teams were using on VMs offered much more:

| Feature | Enterprise LB (VM world) | K8s Service |
|---------|--------------------------|-------------|
| **Sticky sessions** | ✓ (same user always hits same pod) | ✗ |
| **Ratio-based routing** | ✓ (e.g. 30% to v1, 70% to v2) | ✗ |
| **Path-based routing** | ✓ (`/api` → service A, `/web` → service B) | ✗ |
| **Host-based routing** | ✓ (`amazon.com` vs `amazon.in`) | ✗ |
| **TLS / HTTPS** | ✓ | ✗ |
| **Web Application Firewall** | ✓ | ✗ |
| **Whitelisting / Blacklisting** | ✓ (block by country, IP range, etc.) | ✗ |
| **Rate limiting** | ✓ | ✗ |

Teams that migrated to Kubernetes went from a powerful load balancer to simple round-robin — a significant regression.

### Problem 2 — LoadBalancer service type is expensive at scale

Creating a Service of type `LoadBalancer` on a cloud provider provisions a **separate static public IP address** for each service. Cloud providers charge per static IP.

| Approach | IPs needed | Cost |
|----------|-----------|------|
| **VM world**: one load balancer, many apps via routing | 1 IP total | Low |
| **K8s LoadBalancer services**: one IP per service | 1 IP × N services | High (1000 services = 1000 IPs) |

Ingress solves this by routing all external traffic through **one IP address / one load balancer**, regardless of how many services are behind it.

---

## Why Not Just Use NodePort or LoadBalancer?

### NodePort Limitations

- Ports are assigned in a random range (`30000–32767`)
- You cannot predict which port will be assigned
- Opening this range on your firewall is a security risk — you're exposing your entire node to potential attack
- Not accessible if the node itself is not reachable externally

### LoadBalancer Limitations

- **Cost at scale:** Each `LoadBalancer` service creates a separate static public IP (charged by your cloud provider)
- For an e-commerce platform with 1000 services → 1000 separate public IPs → massive cloud cost
- No path-based or host-based routing — you get one IP per service, no routing intelligence

**Ingress fixes both problems:** single entry point IP + all the enterprise load balancing features, via a pluggable controller.

---

## What is Ingress?

Ingress is a Kubernetes resource that defines **routing rules** for HTTP/HTTPS traffic coming into the cluster.

```
Client
  │
  ▼
[ Ingress Resource ] ── defines rules ──▶ routes to correct Service
  │
  ▼
[ Ingress Controller ] ── watches Ingress resources, configures load balancer
  │
  ▼
[ Load Balancer ] (nginx, HAProxy, F5, etc.)
  │
  ▼
[ Service ] ──▶ [ Pods ]
```

**Key rule:** An Ingress resource alone does nothing. It is an **orphan** without an Ingress Controller watching it.

**One Ingress for many services:** Ingress is NOT a 1:1 mapping with services. A single Ingress resource can route to 100 different services via path or host rules — this is the whole point. You write the routing rules once and the controller handles everything behind it.

---

## Ingress Controllers

An Ingress Controller is a Kubernetes controller (custom controller pattern) that:
1. Watches for Ingress resources on the cluster
2. Translates them into load balancer configuration (e.g., `nginx.conf`)
3. Updates the load balancer dynamically when Ingress resources change

**The architecture agreement:** Kubernetes said "we'll define the Ingress resource API, but we can't implement logic for every load balancer in the market." So they told load balancer vendors: *you* write an Ingress Controller for your product, host it on GitHub with installation steps, and users will deploy whichever controller fits their needs. This is why there are 30–40 supported implementations.

### Popular Ingress Controllers

| Controller | Notes |
|-----------|-------|
| **NGINX** | Most common, runs as a Pod inside the cluster, updates `nginx.conf` |
| **HAProxy** | Enterprise-grade, updates `haproxy.conf` |
| **F5 / BIG-IP** | Enterprise, load balancer sits *outside* the cluster, communicates via vxlan tunnel |
| **Citrix** | Similar to F5, external load balancer |
| **Traefik** | Cloud-native, auto-discovery |
| **AWS ALB** | AWS-native, integrates with ALB |

> There are 30–40 officially supported Ingress Controllers listed in the Kubernetes docs, plus many more community implementations.

### How nginx Ingress Controller works internally

When you create an Ingress resource, the nginx controller detects it and updates `/etc/nginx/nginx.conf` with new routing entries:

```nginx
server {
    listen 80;
    server_name food.bar.com;

    location /first {
        proxy_pass http://http-svc:80;
    }

    location /second {
        proxy_pass http://miyab-svc:80;
    }
}
```

For enterprise load balancers (F5, Citrix) that are not containerized, the load balancer sits outside the cluster and connects via a **vxlan tunnel**.

---

## Ingress Classes

If you run multiple Ingress Controllers in the same cluster (e.g., nginx for team A, HAProxy for team B), you need a way to tell each controller which Ingress resources to watch.

**Ingress Class** solves this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx   # only nginx controller watches this
  ...
```

- The controller is configured to watch only Ingresses with a specific `ingressClassName`
- If no class is set, the controller can be configured to watch **all** Ingresses without a class
- This enables multi-tenancy: different teams use different Ingress Controllers in the same cluster

---

## Ingress YAML Structure

### Basic Ingress (no TLS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: food.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

> **Note:** The `apiVersion` changed from `extensions/v1beta1` to `networking.k8s.io/v1` in Kubernetes 1.21. The old version is deprecated.

---

## Routing Types

### 1. Host-Based Routing

Route traffic based on the HTTP `Host` header:

```yaml
rules:
  - host: food.bar.com       # → routes to service-a
  - host: payments.bar.com   # → routes to service-b
```

```bash
# Must pass host header to test locally
curl -H "Host: food.bar.com" http://<ingress-ip>
```

### 2. Path-Based Routing

Route traffic based on the URL path, all under the same host:

```yaml
rules:
  - host: food.bar.com
    http:
      paths:
        - path: /first
          pathType: Prefix
          backend:
            service:
              name: http-svc
              port:
                number: 80
        - path: /second
          pathType: Prefix
          backend:
            service:
              name: miyab-svc
              port:
                number: 80
```

```bash
curl -H "Host: food.bar.com" http://<ingress-ip>/first    # → http-svc
curl -H "Host: food.bar.com" http://<ingress-ip>/second   # → miyab-svc
```

### 3. Wildcard Host Matching

Match any subdomain — useful for multi-tenant setups (like `slides.google.com`, `docs.google.com`, `mail.google.com`):

```yaml
rules:
  - host: "*.bar.com"     # note: must be quoted in YAML
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: http-svc
              port:
                number: 80
```

```bash
curl -H "Host: food.bar.com" http://<ingress-ip>      # ✓
curl -H "Host: example.bar.com" http://<ingress-ip>   # ✓
curl -H "Host: anything.bar.com" http://<ingress-ip>  # ✓
```

> **Gotcha:** Always use double quotes around wildcard hosts in YAML (`"*.bar.com"`) — missing quotes will cause subtle parsing errors.

### 4. Basic Authentication

Restrict access to a service — only authorized users can reach it:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Restricted"
```

---

## TLS with Ingress

### Storing Certificates as Secrets

Unlike OpenShift Routes (which embed certificates directly in the resource), Kubernetes Ingress stores TLS certificates in **Secrets**:

```bash
# Create TLS secret from cert files
kubectl create secret tls my-tls-secret \
  --cert=server.crt \
  --key=server.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
    - hosts:
        - food.bar.com
      secretName: my-tls-secret     # reference the secret
  rules:
    - host: food.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: http-svc
                port:
                  number: 80
```

This is an **advantage of Ingress over OpenShift Routes** — the certificates live in a Secret (which can be managed separately, rotated, and not committed to Git).

---

## TLS Termination Strategies

The three ways to handle encryption between client, load balancer, and your application:

### 1. SSL Passthrough

```
Client ──(encrypted)──▶ Load Balancer ──(encrypted, unchanged)──▶ App
```

- The load balancer acts as a **pure TCP proxy** — it does not inspect packets
- Encryption happens end-to-end between client and application
- The application must handle decryption for every request

**Pros:**
- Maximum security — no decryption in the middle
- Simple to configure

**Cons:**
- Load balancer cannot inspect packets → **no WAF, no path-based routing, no cookie-based routing**
- Application bears the full cost of SSL decryption under load → **higher latency at peak traffic**
- If a malicious request arrives, the load balancer passes it through unchecked
- Only provides **L4 (TCP) load balancing**, not L7 (HTTP)

**Use when:** You specifically need L4 load balancing and end-to-end encryption without any L7 features.

---

### 2. SSL Offloading (Edge Termination)

```
Client ──(encrypted)──▶ Load Balancer ──(plain HTTP)──▶ App
```

- Load balancer decrypts the request, then sends plain HTTP to the application
- Application never sees encrypted traffic

**Pros:**
- Application doesn't bear SSL decryption cost → **lower latency**
- Load balancer can use full L7 features (path routing, WAF, cookies)

**Cons:**
- Traffic between load balancer and application is **unencrypted**
- If the load balancer sits at the edge of your network, you're sending plain HTTP across your internal network → **man-in-the-middle risk**
- **Not recommended** when security is a key requirement

**Use when:** Internal services where you trust your network and need low latency, and the load balancer is close to the application.

---

### 3. SSL Bridging (Re-encrypt / SSL Bridge)

```
Client ──(encrypted)──▶ Load Balancer ──(re-encrypted)──▶ App
                              │
                    decrypts → inspects → re-encrypts
```

- Load balancer decrypts the incoming request, **inspects it**, then re-encrypts and forwards to the app
- Both the client-to-LB and LB-to-app channels are encrypted

**Pros:**
- **Full L7 inspection**: WAF, path routing, cookie handling all work
- **Security**: load balancer can detect and block malicious requests before they reach the app
- Both legs of communication are encrypted

**Cons:**
- Application must still handle SSL decryption (re-encrypted traffic)
- More CPU-intensive at the load balancer level

**Use when:** You need both security and L7 features — this is the recommended approach for most production scenarios.

---

### TLS Strategy Comparison

| | SSL Passthrough | SSL Offloading | SSL Bridging |
|--|----------------|----------------|--------------|
| **Client → LB** | Encrypted | Encrypted | Encrypted |
| **LB → App** | Encrypted (pass-through) | Plain HTTP | Re-encrypted |
| **L7 routing** | No (L4 only) | Yes | Yes |
| **WAF / inspection** | No | Yes | Yes |
| **App decryption burden** | High | None | High |
| **Security** | High (but LB blind) | Low | High |
| **Recommended** | Rare cases only | Only if security can be compromised | **Yes (most cases)** |

---

## OpenShift Routes vs. Kubernetes Ingress

OpenShift has its own equivalent called **Routes**, backed by HAProxy. The TLS naming is different:

| Kubernetes Ingress Term | OpenShift Route Term |
|------------------------|---------------------|
| SSL Offloading | **Edge termination** |
| SSL Bridging | **Re-encrypt termination** |
| SSL Passthrough | **Passthrough termination** |

### Key difference — certificate storage

**OpenShift Routes (HAProxy):** Certificates must be embedded directly in the Route resource → cannot be stored in Git safely.

**Kubernetes Ingress:** Certificates are stored in a Kubernetes Secret → the Ingress only references the secret name → **safe to store in Git**.

This is why many teams use Ingress even on OpenShift clusters, despite native Route support being available.

### OpenShift Service Serving Certificates

For internal cluster communication, OpenShift offers a shortcut — annotate your Service and OpenShift's built-in CA automatically generates and mounts a certificate:

```yaml
metadata:
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: my-cert-secret
```

The CA cert is stored in a ConfigMap in the same namespace and can be mounted into Pods that need to trust it.

---

## Installation

### On minikube (local development)

The simplest way — minikube ships nginx Ingress Controller as an add-on:

```bash
minikube addons enable ingress

# Verify the controller pod is running
kubectl get pods -n ingress-nginx
```

That single command installs and starts the nginx Ingress Controller. No Helm chart needed for local dev.

### On production clusters (EKS, GKE, OpenShift, etc.)

Each controller has its own Helm chart / manifest. Go to the controller's official docs and follow their installation steps. Example for nginx:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx
```

---

## Local Domain Testing

In production, your `host:` value (e.g. `amazon.com`) is a real DNS domain. Locally, you're using fake domains like `food.bar.com` that don't exist in DNS. Two ways to fake it:

### Option 1 — Update /etc/hosts

```bash
# Get the Ingress IP
kubectl get ingress

# Add to /etc/hosts
sudo vim /etc/hosts
# Add this line:
192.168.64.11  food.bar.com
```

Now `curl food.bar.com` resolves to your Ingress IP.

### Option 2 — Pass host header in curl (no /etc/hosts change needed)

```bash
curl --resolve "food.bar.com:80:192.168.64.11" http://food.bar.com/bar
# or
curl -H "Host: food.bar.com" http://192.168.64.11/bar
```

---

## Debugging Ingress Issues

### Step 1 — Check the ADDRESS field

```bash
kubectl get ingress
```

- **ADDRESS is empty** → Ingress Controller has not picked up this resource yet. Either the controller isn't running, or the `ingressClassName` doesn't match.
- **ADDRESS is populated** → Controller is watching it and a load balancer IP is assigned.

### Step 2 — Check controller logs

The controller logs show whether it detected and synced your Ingress resource:

```bash
kubectl logs -n ingress-nginx <controller-pod-name>
```

Look for a line like:
```
Successfully synced "default/ingress-example"
```

If there's no such line, the controller is ignoring the resource (class mismatch, namespace issue, etc.).

### Step 3 — Check nginx.conf directly

```bash
kubectl exec -n ingress-nginx <pod-name> -- cat /etc/nginx/nginx.conf | grep food.bar.com
```

If your host/path rules are missing from the config, the routing won't work regardless of DNS.

For **HAProxy** (OpenShift): check `/etc/haproxy/haproxy.conf` in the router pod the same way.

### Common Issues

| Symptom | Likely Cause |
|---------|-------------|
| `ADDRESS` column empty | No Ingress Controller running, or `ingressClassName` mismatch |
| `404 Not Found` | No matching rule for the host/path you're accessing |
| TLS connection refused | Secret missing or wrong `secretName` in `spec.tls` |
| Host not matched | Forgot to pass `Host` header or update `/etc/hosts` in local testing |
| Rules missing from nginx.conf | Ingress class annotation missing or wrong |

---

## Key Takeaways

### Why Ingress exists (interview answers)
- **Problem 1:** Kubernetes Services only do round-robin. Enterprise load balancers on VMs offered sticky sessions, ratio-based routing, path/host routing, WAF, whitelisting, TLS — all missing from Services.
- **Problem 2:** `LoadBalancer` service type creates one static public IP per service. Cloud providers charge per IP. 1000 services = 1000 IPs = massive cost. VMs had one load balancer for everything.
- Ingress solves both: one entry IP + pluggable enterprise LB features via controllers.

### Architecture
- Kubernetes defines the **Ingress resource** (the API). Load balancer vendors write **Ingress Controllers** (the implementation). You choose which controller to deploy.
- An Ingress resource **does nothing without an Ingress Controller** — it is an orphan.
- **One Ingress can handle 100 services** — it's not 1:1.

### Operations
- **Ingress Classes** let you run multiple controllers in one cluster (e.g., nginx for team A, HAProxy for team B).
- On minikube: `minikube addons enable ingress` installs nginx controller in one command.
- Local testing: fake domains with `/etc/hosts` or `curl -H "Host: ..."`.
- Debugging order: check `ADDRESS` field → check controller logs → check `nginx.conf`.

### TLS
- **SSL Bridging (Re-encrypt)** — recommended. Decrypts, inspects, re-encrypts. Both legs secure, full L7 features.
- **SSL Offloading (Edge)** — fast but plain HTTP to backend. Only if you can accept the security trade-off.
- **SSL Passthrough** — L4 only. No L7 routing, no WAF. Rarely appropriate.
- Kubernetes Ingress stores certs in Secrets (Git-safe). OpenShift Routes embed them inline (not Git-safe — reason many teams use Ingress even on OpenShift).

### Routing
- **Path-based:** `food.bar.com/a` → service A, `food.bar.com/b` → service B
- **Host-based:** `food.bar.com` → service A, `payments.bar.com` → service B
- **Wildcard:** `"*.bar.com"` matches any subdomain (must double-quote in YAML)

---

## Resources

- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers List](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [MetalLB (LoadBalancer for bare metal)](https://metallb.universe.tf/)
- [OpenShift Routes](https://docs.openshift.com/container-platform/latest/networking/routes/route-configuration.html)
