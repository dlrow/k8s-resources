# Kubernetes & Istio — Controllers vs. Config: The Full Picture

---

## The Core Distinction

| Category | What it is | Examples |
|----------|-----------|---------|
| **Controller / Running Code** | An actual process that watches resources and takes action | kube-scheduler, kubelet, istiod |
| **Config / Declarative Resource** | A YAML object stored in etcd — no logic, just desired state | Deployment, VirtualService, Service |

A config does nothing on its own. A controller watches configs and makes reality match them.

---

## Running Code (Controllers & Processes)

### Kubernetes Control Plane (master node)

| Component | Watches | Acts On |
|-----------|---------|---------|
| **kube-apiserver** | All incoming HTTP requests | Authenticates, admits, persists to etcd. Single entry point for everything. |
| **etcd** | — | Stores all cluster state (the source of truth) |
| **kube-scheduler** | Pods with no `nodeName` | Picks a node for the pod, writes `nodeName` to pod spec |
| **kube-controller-manager** | (runs all below controllers in one binary) | |
| &nbsp;&nbsp;↳ DeploymentController | Deployment resources | Creates and manages ReplicaSets |
| &nbsp;&nbsp;↳ ReplicaSetController | ReplicaSets | Creates/deletes Pods to match `replicas` count (auto-healing) |
| &nbsp;&nbsp;↳ DaemonSetController | DaemonSets | Ensures one Pod per node |
| &nbsp;&nbsp;↳ StatefulSetController | StatefulSets | Manages ordered, stateful pods with stable identity |
| &nbsp;&nbsp;↳ JobController | Jobs | Runs pods to completion, tracks success/failure |
| &nbsp;&nbsp;↳ CronJobController | CronJobs | Creates Jobs on a schedule |
| &nbsp;&nbsp;↳ EndpointsController | Services + Pods | Creates/updates Endpoints object (the list of pod IPs behind a Service) |
| &nbsp;&nbsp;↳ NodeController | Nodes | Detects and responds to node failures |
| &nbsp;&nbsp;↳ NamespaceController | Namespaces | Cleans up resources when a namespace is deleted |
| &nbsp;&nbsp;↳ PVController | PersistentVolumes / PVCs | Binds PVCs to PVs |
| &nbsp;&nbsp;↳ HPAController | HorizontalPodAutoscaler | Scales deployment replicas based on metrics |
| **cloud-controller-manager** | LoadBalancer Services, Nodes | Creates cloud LBs (AWS ELB, GCP LB), assigns external IPs |

### Kubernetes Worker Node

| Component | Watches | Acts On |
|-----------|---------|---------|
| **kubelet** | Pods assigned to its node (via API server) | Starts/stops containers via container runtime. Node-level agent. |
| **kube-proxy** | Service + Endpoints objects | Updates iptables/IPVS rules on the node so Service ClusterIPs route to correct pods |
| **container runtime** | Kubelet instructions | Actually runs containers (containerd, CRI-O) |

### Admission Controllers (compiled into kube-apiserver — not separate processes)

These intercept requests **before** objects are stored in etcd:

| Admission Controller | Type | What it does |
|---------------------|------|-------------|
| **DefaultStorageClass** | Mutating | Adds default storageClass to PVCs that don't specify one |
| **ResourceQuota** | Validating | Rejects pods that exceed namespace CPU/memory quota |
| **LimitRanger** | Mutating + Validating | Applies default resource limits, rejects over-limit requests |
| **NamespaceLifecycle** | Validating | Rejects resource creation in terminating namespaces |
| **ServiceAccount** | Mutating | Auto-mounts service account tokens into pods |
| **NodeRestriction** | Validating | Limits what kubelet can modify |
| **PodSecurity** | Validating | Enforces pod security standards |
| **MutatingAdmissionWebhook** | Special | Forwards requests to **external** mutation webhooks (e.g. istiod) |
| **ValidatingAdmissionWebhook** | Special | Forwards requests to **external** validation webhooks (e.g. istiod) |

### Ingress Controller (separate pod — not built into K8s)

| Component | Watches | Acts On |
|-----------|---------|---------|
| **nginx / HAProxy / Traefik pod** | Ingress resources | Updates its own load balancer config (nginx.conf, haproxy.conf) |

---

### Istio Control Plane (istio-system namespace)

| Component | Watches | Acts On |
|-----------|---------|---------|
| **istiod — Pilot** | Istio CRDs (VirtualService, DestinationRule, Gateway…), K8s Services + Endpoints | Translates to xDS config, pushes to every Envoy sidecar in the mesh |
| **istiod — Citadel** | Pod ServiceAccounts | Issues/rotates X.509 mTLS certificates (SPIFFE/SVID) for every sidecar |
| **istiod — Galley** | Istio CRD creation requests | Validates Istio configs; implements the `/inject` MutatingWebhook endpoint |
| **Envoy sidecar (istio-proxy)** | xDS config pushed from istiod | Executes routing rules, enforces mTLS, implements circuit breaking, reports metrics |
| **IngressGateway pod** | Gateway + VirtualService CRDs | Routes external (north-south) traffic into the mesh |
| **EgressGateway pod** | Gateway + VirtualService CRDs | Controls and routes outbound traffic leaving the mesh |
| **Kiali** | Prometheus metrics | Renders service graph and observability dashboard |

---

## Just Config (Declarative Resources — stored in etcd)

### Kubernetes Native Resources

| Resource | Watched/Acted On By |
|---------|-------------------|
| **Pod** | kubelet (starts it), kube-scheduler (places it), ReplicaSetController (creates it) |
| **Deployment** | DeploymentController |
| **ReplicaSet** | ReplicaSetController |
| **DaemonSet** | DaemonSetController |
| **StatefulSet** | StatefulSetController |
| **Job / CronJob** | JobController / CronJobController |
| **Service** | EndpointsController (creates Endpoints), kube-proxy (writes iptables), istiod Pilot (updates EDS) |
| **Endpoints** | kube-proxy (writes iptables rules), istiod Pilot (populates Envoy EDS) |
| **Ingress** | Ingress Controller (nginx, HAProxy, etc.) |
| **ConfigMap / Secret** | kubelet mounts them into pods |
| **PersistentVolume / PVC** | PVController (binds them together) |
| **Namespace** | NamespaceController |
| **ResourceQuota** | ResourceQuota admission controller |
| **LimitRange** | LimitRanger admission controller |
| **ServiceAccount** | ServiceAccount admission controller, Citadel (issues certs) |
| **HorizontalPodAutoscaler** | HPAController |
| **NetworkPolicy** | CNI plugin (Calico, Cilium — not kube-proxy) |
| **Role / ClusterRole / RoleBinding / ClusterRoleBinding** | kube-apiserver (enforces during AuthZ) |

### Istio CRDs

| Resource | Watched/Acted On By | Purpose |
|---------|-------------------|---------|
| **VirtualService** | istiod Pilot → pushes RDS to Envoy | Traffic routing rules (weights, timeouts, retries, fault injection) |
| **DestinationRule** | istiod Pilot → pushes CDS to Envoy | Subset definitions (which pods), circuit breaking, mTLS policy per destination |
| **Gateway** | istiod Pilot → configures IngressGateway/EgressGateway Envoy pods | Entry/exit point for mesh traffic |
| **PeerAuthentication** | istiod Pilot → pushes to Envoy | mTLS mode: PERMISSIVE / STRICT / DISABLE per namespace or workload |
| **AuthorizationPolicy** | istiod Pilot → pushes to Envoy | Who can call whom (service-level ACL) |
| **ServiceEntry** | istiod Pilot → pushes to Envoy | Registers external services so Istio can manage traffic to them |
| **Sidecar** (resource) | istiod Pilot | Limits which services an Envoy proxy is aware of (memory optimization) |
| **EnvoyFilter** | istiod Pilot | Low-level Envoy config patches (advanced, escape hatch) |
| **WorkloadEntry** | istiod Pilot | Registers non-Kubernetes workloads (VMs) into the mesh |

### K8s Resources Created By Istio (config, but enables code)

| Resource | Purpose |
|---------|---------|
| **MutatingWebhookConfiguration** (istio-sidecar-injector) | Tells kube-apiserver: "on every Pod CREATE, call istiod /inject" → enables sidecar injection |
| **ValidatingWebhookConfiguration** (istio-config-validator) | Tells kube-apiserver: "validate Istio CRDs by calling istiod" |

---

## The 4 Key Flows — How Everything Connects

### Flow 1: Pod Creation with Istio Sidecar Injection

```
kubectl apply -f pod.yaml
        │
        ▼
kube-apiserver
  ├── AuthN + AuthZ
  ├── Built-in admission controllers (ResourceQuota, LimitRanger…)
  ├── MutatingAdmissionWebhook
  │     reads MutatingWebhookConfiguration (installed by Istio)
  │     HTTP POST → istiod /inject
  │     istiod adds:
  │       - istio-init container  (sets iptables rules, then exits)
  │       - istio-proxy container (Envoy sidecar)
  │     returns mutated pod spec
  └── stores final pod spec in etcd
        │
kube-scheduler → picks a node → writes nodeName
        │
kubelet on node:
  ├── starts istio-init  → iptables: all traffic goes through Envoy
  ├── starts app container
  └── starts istio-proxy (Envoy)
        │
istiod detects new Envoy → pushes full xDS config (LDS/RDS/CDS/EDS)
```

---

### Flow 2: Service Created → Routing Set Up

```
kubectl apply -f service.yaml → etcd
        │
        ├── EndpointsController
        │     watches Service selector + Pod labels
        │     creates Endpoints object with matching pod IPs
        │             │
        │     kube-proxy on every node
        │             reads Endpoints → updates iptables
        │             (ClusterIP → pod IPs, round-robin)
        │
        └── istiod Pilot
              reads Service + Endpoints
              updates EDS table (real pod IPs for this service)
              pushes EDS to all Envoy sidecars
              (Envoys now know pod IPs directly, bypass kube-proxy for in-mesh traffic)
```

---

### Flow 3: VirtualService Applied → Live Traffic Shift (no restart)

```
kubectl apply -f virtual-service.yaml → etcd
        │
istiod Pilot watches VirtualService change
  reads VirtualService (80% → subset v1, 20% → subset v3)
  reads DestinationRule (subset v1 = pods with label version=v1)
  cross-refs with EDS (which pod IPs have version=v1)
  translates → xDS RDS config
  pushes to all relevant Envoy sidecars (no pod restart)
        │
Next request: Catalog app → http://reviews/
  iptables → Envoy sidecar intercepts
  Envoy checks local RDS: 80% v1 / 20% v3
  Envoy checks EDS: gets a pod IP with version=v1
  Envoy initiates mTLS (cert from Citadel)
  directly connects to target pod IP
  (kube-proxy iptables for this service is bypassed)
```

---

### Flow 4: mTLS — Certificate Lifecycle

```
istiod starts → Citadel generates root CA certificate

New pod with sidecar starts:
  Envoy contacts istiod Citadel
  "I am ServiceAccount X in namespace Y"
  Citadel verifies via K8s TokenReview API
  Issues X.509 SVID cert:
    spiffe://cluster.local/ns/default/sa/catalog

Catalog Envoy → Payment Envoy:
  Catalog presents its SVID cert
  Payment presents its SVID cert
  Both verify against istiod CA root
  Both valid → mTLS connection established
  PeerAuthentication=STRICT → plain HTTP rejected
```

---

## One-Line Job Description — Every Component

| Component | One-line job |
|-----------|-------------|
| **kube-apiserver** | The only door into the cluster — validates every request, persists to etcd |
| **etcd** | The cluster's database — stores all desired state |
| **kube-scheduler** | Decides *which node* a pod runs on |
| **DeploymentController** | Ensures the right ReplicaSet exists for each Deployment |
| **ReplicaSetController** | Ensures the exact pod count is always running (auto-healing) |
| **EndpointsController** | Keeps the pod IP list behind each Service always current |
| **kube-proxy** | Makes Service ClusterIPs work by writing iptables rules on every node |
| **cloud-controller-manager** | Bridges K8s to cloud APIs — creates real load balancers for LoadBalancer Services |
| **kubelet** | Node agent — actually starts and stops your containers |
| **Ingress Controller** | Watches Ingress resources, updates load balancer config (nginx.conf, etc.) |
| **istiod Pilot** | Translates Istio CRDs → xDS config → pushes to every Envoy sidecar |
| **istiod Citadel** | CA that issues and rotates mTLS certificates for every sidecar |
| **istiod Galley** | Validates Istio configs; injects Envoy sidecar into pods at creation time |
| **Envoy sidecar** | Executes routing, mTLS, circuit breaking, and metrics — the actual data plane |
| **IngressGateway** | Envoy pod at the mesh edge that handles incoming external traffic |
| **Deployment** | Config declaring desired pod template and replica count |
| **Service** | Config giving pods a stable DNS name and ClusterIP |
| **Ingress** | Config declaring HTTP routing rules for an Ingress Controller to read |
| **VirtualService** | Config declaring Istio traffic routing rules (weights, timeouts, retries) |
| **DestinationRule** | Config declaring pod subsets and circuit breaking policy for a destination |
| **Gateway** | Config declaring what traffic the IngressGateway/EgressGateway accepts |
| **PeerAuthentication** | Config declaring mTLS mode (STRICT/PERMISSIVE) for a namespace or workload |
| **AuthorizationPolicy** | Config declaring which services can call which other services |
| **MutatingWebhookConfiguration** | Config telling kube-apiserver to call istiod for sidecar injection on pod create |
