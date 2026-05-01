# K8s Traffic Flow — Whiteboard Walkthrough

Two services, two completely different shapes of traffic — together they cover ~99% of what flows into your cluster.

> Note: K8s networking knowledge based on a snapshot ~3 weeks old, but these foundational pieces don't change often. Specifics like exact NLB names match what's deployed.

---

## Part 1: The mental model

Forget Kubernetes for a second. When a request enters your cluster, **two unrelated problems** must be solved:

1. **"Which physical machine should this packet hit?"** — pure networking (TCP/IP). Solved by AWS load balancers.
2. **"Once inside, which microservice should handle this HTTP/gRPC request?"** — application routing. Solved by Istio.

Many devops teams confuse these because they live in the same YAML files. They're independent.

---

## Part 2: Cast of characters — config vs. code

Here's the table you'll want to memorize. **This is the key insight you asked for.**

| Thing | Config or code? | What it actually is |
|---|---|---|
| **AWS NLB / ALB** | Hardware-ish | A physical AWS load balancer box, sitting *outside* your cluster. You don't run it; AWS does. |
| **Kubernetes Service (`type: LoadBalancer`)** | Pure config (YAML) | A K8s object that *tells AWS* "please create me an NLB and point it at these pods". The YAML is the order form; the NLB is what AWS provisions in response. |
| **Kubernetes Ingress** | Pure config (YAML) | Same idea but for ALBs. The "AWS Load Balancer Controller" pod watches Ingress YAMLs and provisions ALBs. |
| **istiod** | Real code | A pod running in `istio-system`. It reads all your Istio YAML and pushes config to every Envoy proxy. The "brain". |
| **Envoy** | Real code (the most important code in your cluster) | A C++ proxy. There are TWO places it runs: (a) **ingress gateway pods** (the door) and (b) a **sidecar inside every microservice pod** (one extra container next to your Java app). Every byte of your traffic flows through Envoy somewhere. |
| **Gateway** (Istio CRD) | Pure config (YAML) | Tells an ingress Envoy "open port 443, expect mTLS". |
| **VirtualService** (Istio CRD) | Pure config (YAML) | Tells Envoy "if hostname=X and path=/api/tdctl/* → send to devicecontrol-service". A routing rule. |
| **DestinationRule** (Istio CRD) | Pure config (YAML) | Tells Envoy "when calling service X, use mTLS, prefer same AZ, eject sick pods". |
| **ServiceEntry** (Istio CRD) | Pure config (YAML) | Tells Envoy "this external host (e.g. AWS Redis) is allowed; treat it as a known destination." Without it Envoy blocks egress. |
| **EnvoyFilter** (Istio CRD) | Config + Lua snippets | Custom logic injected into Envoy's request pipeline. The Lua *is* code, but it runs inside Envoy. |
| **PeerAuthentication / RequestAuthentication / AuthorizationPolicy** | Pure config (YAML) | Security rules Envoy enforces. "Require valid JWT", "require mTLS", "allow path /login without auth". |

**Burn this in:** The only running processes are AWS LBs (outside), `istiod` (1 brain pod), Envoy (everywhere — gateways + sidecars), and your microservice pods. Everything else is YAML telling Envoy what to do.

A common misconception: "Istio is doing the routing." No — **Envoy is doing the routing**. Istio is just the YAML format and the controller (istiod) that programs Envoys. Istio without Envoy doesn't move a single byte.

---

## Part 3: Flow A — Device → alarm-dial-out-service

Devices in the field (radios, BNs, RNs) sit behind firewalls that only allow outbound. So the device dials *out* to TCC. This is the "device-initiated" flow.

```
[Field device, anywhere on the public internet]
        │
        │  gRPC over mTLS, port 443
        │  Destination: alarm.tcs-lab.cloud.taranawireless.com
        │
        │  ────────── HOP 1 ──────────
        ▼
[ AWS NLB — internet-facing, static EIP, dual-stack IPv4/IPv6 ]
        │
        │  ────────── HOP 2 ──────────
        ▼
[ device-public-ingressgateway pod ]   (Envoy code)
        │
        │  ────────── HOP 3 ──────────
        ▼
[ alarm-dial-out-service pod + its Envoy sidecar ]   (your Java + Envoy code)
        │
        │  ────────── HOP 4 ──────────
        ▼
[ Kafka — alarm topic ]
```

### Hop 1: AWS NLB (physical box, outside cluster)

**Why does this exist?** Without it, the only way a device could reach your cluster is to know the IP of a specific node and pray that node never reboots. The NLB gives a stable, redundant front door.

**What it does:**
- Has a fixed public DNS name and a static elastic IP
- Listens on TCP/443
- When a device opens a TCP connection, NLB picks one healthy K8s node and forwards raw bytes to it
- **It does NOT decrypt TLS.** That's why it's an NLB (Network LB) and not an ALB. Decryption costs CPU; for device traffic at scale, AWS just shovels bytes
- Does TCP-level health checks against the node ports

**Provisioned by which YAML?** The K8s Service `device-public-ingressgateway` in `istio-system` (with `type: LoadBalancer`). When that Service was created, K8s told AWS "I need an NLB". AWS provisioned it.

**Why specifically NLB and not ALB here?** Three reasons:
1. mTLS — the *device's* certificate must reach Envoy intact for cert-based auth. ALB would terminate TLS and the device cert would be lost.
2. gRPC bidirectional streams need raw TCP passthrough.
3. NLBs are cheaper and faster for high-volume long-lived connections.

### Hop 2: Istio Ingress Gateway pod (Envoy, real code)

The NLB delivered raw TCP bytes to a node, which forwarded them to one of the `device-public-ingressgateway` pods. There are 8–12 replicas; this is one of your most critical workloads.

**What's actually running in this pod?** A single C++ Envoy process. Same Envoy binary as the sidecars, but configured to act as a "front door".

**What does it do here?**
1. **Terminates the mTLS handshake.** This is where TLS is finally decrypted. The pod presents the gateway's server cert; the device presents its device cert. Both are validated. (Configured by an Istio `Gateway` CRD with `tls.mode: MUTUAL`.)
2. **Looks at the SNI** (Server Name Indication — the hostname the client connected with, like `alarm.tcs-lab...`).
3. **Looks up a VirtualService** that says "this hostname → alarm-dial-out-service".
4. **Forwards the now-decrypted request** internally to the alarm-dial-out-service pod.

**Configs that shape this hop:**
- `Gateway` CRD: which port, which TLS mode, which cert
- `VirtualService` CRD: hostname → backend mapping
- TLS certs are mounted as Kubernetes Secrets

**Why a separate ingress gateway pod and not just send NLB straight to the service?** Centralization. You have ~150 services. You don't want 150 LoadBalancers, 150 cert mounts, 150 mTLS configs. The gateway is the one place that handles the "outside world" cert handshake; everything inside is uniform.

### Hop 3: alarm-dial-out-service pod (Java + Envoy sidecar)

The pod has **two containers**:
1. Your Java Spring Boot `dialout-service` JAR
2. An Envoy sidecar (auto-injected because the namespace is labeled `istio-injection: enabled`)

Traffic flows: gateway Envoy → sidecar Envoy → Java app. The sidecar Envoy is why "everything is mTLS" without your code knowing — Envoy adds the mTLS, Envoy strips the mTLS, the Java app just sees plain HTTP/gRPC on localhost.

**What the Java code does:**
- `AuthorizationInterceptor` extracts the device serial from the cert CN (cert was forwarded as a header by the gateway)
- `DialTccService.pushSubscriptionUpdates()` — the gRPC method handling the streaming RPC
- `SubscribeResponseStreamObserver` — for each `SubscribeResponse` message: send `UpdateAck` back, hand to Kafka producer
- Listens on gRPC port 58081 (mTLS) or 58080 (plaintext, in-cluster only)

### Hop 4: Java code → Kafka

Now the alarm reaches Kafka (`alarm` topic). Downstream consumers process it. End of inbound flow for this service.

**Note:** This service only *receives* from devices and *publishes* to Kafka. It never replies with business logic. That's why it's "dial-out" — devices dial out *to it*.

---

## Part 4: Flow B — User/API → devicecontrol-service

This is the opposite flow shape. A human operator or external API client wants to reboot a device. They hit a REST API.

```
[ Operator's browser / external API client ]
        │
        │  HTTPS POST https://api.tcs-lab.cloud.taranawireless.com/api/tdctl/v1/reboot
        │
        │  ────────── HOP 1 ──────────
        ▼
[ AWS ALB — provisioned by the tcc-ingressgateway-alb Ingress ]
        │
        │  ────────── HOP 2 ──────────
        ▼
[ tcc-ingressgateway pod ]   (Envoy)
        │
        │  ────────── HOP 3 ──────────
        ▼
[ devicecontrol-service pod + sidecar ]   (Java + Envoy)
        │
        │  ────────── HOP 4 ──────────  (now this service has work to do)
        ▼
[ networkinfo-query-service / softwareinventory-service / Kafka / DB ]
        │
        │  ────────── HOP 5 ──────────  (egress back to a device)
        ▼
[ Device (via tnxi tunnel or direct gNOI) ]
```

### Hop 1: AWS ALB (different beast from NLB)

The Ingress called `tcc-ingressgateway-alb` is a YAML config that tells the AWS LB Controller to provision an **ALB** (Application Load Balancer).

**Differences from the NLB on the device path:**

| | NLB (device path) | ALB (user path) |
|---|---|---|
| Layer | TCP (Layer 4) | HTTP (Layer 7) |
| Decrypts TLS? | No | **Yes** — terminates the user's TLS using an ACM cert |
| Can read URL paths? | No | Yes |
| Can do HTTP→HTTPS redirect? | No | Yes |
| Cost | Lower | Higher |

**Why ALB here?** The user-facing portal needs HTTP-level features: HTTP→HTTPS redirect, ACM cert rotation, HTTP/2, longer idle timeouts, browser cookies. NLB can't do those.

**Important detail:** TLS is terminated *twice* on the user path — once at the ALB (user→ALB encrypted with ACM cert) and again inside the cluster (Envoy sidecars use ISTIO_MUTUAL mTLS). Not redundant — the first hop protects against the public internet, the inside hops protect against intra-cluster sniffing.

### Hop 2: tcc-ingressgateway pod (Envoy, again)

Same kind of pod as before — just Envoy with different config tuned for user traffic. This is where things get rich.

**What this Envoy does for every request:**

1. **EnvoyFilters** (the Lua-snippet things — these *are* code, custom code you wrote):
    - Preserves the `X-Request-ID` header — critical for tracing one request across many services
    - Adds `Cache-Control: no-cache` to API responses
    - Per-environment cookie handling (19 variants — separate filter per env)

2. **`RequestAuthentication`** (config): "validate the JWT in the `Authorization` header against these two Cognito issuers". If invalid → 401. The Java code never sees an unauthenticated request.

3. **`AuthorizationPolicy`** (config): allow-list of public paths (`/login`, `/refresh`, `/operator-portal*`, `/.well-known/security.txt`). All others require valid JWT.

4. **`VirtualService` routing** (config): looks at the URL prefix and picks the backend:
    - `/api/tdctl/v1/*` → **devicecontrol-service**
    - `/api/tdc/v1/*` → deviceconfig-service
    - `/api/tni/v1/*` → networkinfo-service-v2
    - `/api/tcs/v1/*` → security-service
    - …and so on, 152 VirtualServices total

The single Envoy pod, configured by ~1000 lines of total YAML, handles authentication, routing, tracing, and policy for every external request to your cluster. **No code you wrote runs at this hop** (except the Lua filters, which you wrote once and forgot about).

### Hop 3: devicecontrol-service pod

Same pattern as Flow A: two containers, Envoy sidecar terminates intra-cluster mTLS, Java sees plain HTTP. Java app receives the REST request on `/api/tdctl/v1/reboot`.

### Hop 4: Service-to-service calls (the lateral flow)

Now devicecontrol-service has to do work. It needs to:
- Call **networkinfo-query-service** to look up the device's BN endpoint
- Maybe call **softwareinventory-service** for SW install metadata
- Write to **PostgreSQL** for the operation history
- Maybe consume **Kafka** events for auto-upgrade

**This traffic NEVER touches an NLB or ingress gateway.** It's pod-to-pod inside the cluster.

```
devicecontrol-service Java app
   │  http://networkinfo-query-service/api/tnq/v1/devices/...
   ▼
devicecontrol-service Envoy SIDECAR     ← iptables rules in the pod
   │  redirect outbound traffic into the sidecar
   │
   │  Sidecar applies the default DestinationRule:
   │    *.default.svc.cluster.local → tls.mode: ISTIO_MUTUAL
   │    → automatic mTLS handshake (sidecar↔sidecar)
   │    → locality-aware load balancing (prefers same AZ)
   │    → outlier detection (auto-eject sick pods)
   ▼
networkinfo-query-service Envoy SIDECAR
   │  Validates mTLS, enforces AuthorizationPolicy
   ▼
networkinfo-query-service Java app
```

**The Java app never knew about mTLS.** It made a plain HTTP call to a hostname; iptables rules (set up by an init container at pod start) silently rerouted that traffic into the sidecar; the sidecar wrapped it in mTLS; the receiving sidecar unwrapped it. This is the most magical thing about a service mesh and the reason it exists. **Encryption with zero application code changes.**

The single `default` DestinationRule applying to `*.default.svc.cluster.local` is what makes this universal — one YAML, every service automatically encrypted.

### Hop 5: Egress back out to a device

Now devicecontrol-service needs to actually reboot the device. Two paths:

**(a) Direct gNOI path** — service opens a gRPC connection straight to the device's IP over OpenVPN.
**(b) Tnxi tunnel path** — service sends the request to `tnxi-dial-out-service`, which already has a persistent stream open to the device (because the device dialed in earlier, just like in Flow A).

For path (a) — the egress to AWS infra and external hosts goes through `tcc-egressgateway` and is permitted by `ServiceEntry` resources (e.g., `mesh-external-service-entry` for `*.compute.internal`).

For external Redis/S3/etc. — same pattern: `ServiceEntry` to allow it, `DestinationRule` to set policy (locality, keepalive, connection pool).

---

## Part 5: Side-by-side comparison of the two flows

| | alarm-dial-out-service | devicecontrol-service |
|---|---|---|
| Who initiates? | Field device | Operator/API client |
| Protocol | gRPC streaming | HTTP REST |
| Front door | Public NLB (`device-public-ingressgateway`) | Public ALB (`tcc-ingressgateway-alb`) |
| TLS termination | At Istio gateway pod (NLB passthrough) | At AWS ALB |
| Auth | mTLS (device cert) | JWT (Cognito) |
| What handles auth? | Envoy gateway + Java `AuthorizationInterceptor` | Envoy `RequestAuthentication` (zero Java code) |
| Routing key | SNI hostname | URL path prefix |
| Connection lifetime | Long-lived stream | Short request/response |
| Inside the pod | Java + Envoy sidecar | Java + Envoy sidecar |
| Output | Kafka | DB + downstream services + device egress |

---

## Part 6: One-page mental model

```
                  EVERYTHING OUTSIDE THE CLUSTER
                  ─────────────────────────────────
                  AWS NLB                 AWS ALB
                  (TCP passthrough)       (HTTP, terminates TLS)
                  device traffic          user traffic
                       │                       │
══════════════════════ ╪ ═════ cluster edge ═══╪ ═════════════════════
                       ▼                       ▼
                 Istio Ingress Gateway pods (Envoy code)
                  • Gateway CRD = "open port 443, mTLS"
                  • VirtualService = "hostname/path → service"
                  • EnvoyFilter = custom Lua (JWT, API keys)
                  • RequestAuthentication = JWT validation
                  • AuthorizationPolicy = who can hit what
                       │
                       ▼
                  ┌─────────────────────────────────────┐
                  │ Microservice pod                    │
                  │   container 1: your Java app        │
                  │   container 2: Envoy sidecar        │
                  │   (added automatically by Istio)    │
                  └────────────┬────────────────────────┘
                               │
              sidecar ↔ sidecar (auto mTLS via DestinationRule)
                               │
                               ▼
                       Other services
                       Kafka / DB
                       External (S3/Redis) — needs ServiceEntry
                       Cross-cluster (TCS) — via egress gateway
```

---

## Part 7: The one-paragraph summary

When a request arrives, AWS hardware (NLB or ALB) gets it onto a node; the node forwards it into an Istio ingress-gateway pod, which is just an **Envoy** (real C++ proxy) with a stack of YAMLs telling it how to route, decrypt, and authenticate; Envoy then forwards to your service pod, where another Envoy (the sidecar) wraps everything in cluster-internal mTLS before it touches your Java code. The Java code makes plain HTTP calls; sidecars handle encryption, retries, and observability transparently. **All the YAML — Gateway, VirtualService, DestinationRule, ServiceEntry, AuthorizationPolicy — is config that programs Envoy. The only running code in the data path is Envoy and your microservice.**
