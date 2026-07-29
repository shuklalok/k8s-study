# TOPIC 4 — K8s Networking / Firewall Policies (~30 min)

## 4.1 — Kubernetes Networking Model

> **Ref:** https://kubernetes.io/docs/concepts/cluster-administration/networking/

**Say THIS:**

Kubernetes has a simple but powerful networking model built on three fundamental rules:

1. **Every Pod gets its own IP address.** No NAT between Pods — Pod A can reach Pod B directly
by IP, even across nodes.

2. **All Pods can communicate with all other Pods** without NAT (by default — Network Policies
change this).

3. **Nodes can communicate with all Pods** (and vice versa) without NAT.

**Key points — elaborate on each:**

- **Pod networking** — Each Pod gets a unique cluster-internal IP from a CIDR range (e.g.,
`10.42.0.0/16` in k3s). Containers within the same Pod share this IP and communicate via
`localhost`.

- **CNI (Container Network Interface)** — The plugin that implements the networking model.
Popular options: **Flannel** (simple overlay, k3s default), **Calico** (supports Network
Policies + BGP), **Cilium** (eBPF-based, high performance, advanced policies), **Weave Net**.

- **Service networking** — Services get a virtual IP from a separate CIDR (e.g., `10.43.0.0/16`
in k3s). `kube-proxy` (or its replacement) programs iptables/IPVS rules to route traffic from
the Service IP to Pod IPs.

- **The networking model is NOT a firewall.** By default, all Pods can talk to all Pods. This is
intentional for simplicity, but in production you MUST add Network Policies to restrict traffic.

```
K8s Networking Layers

Layer 7: Ingress (HTTP/HTTPS routing, TLS)
Layer 4: Service (Stable IP, load balancing)
Layer 3: Pod Network (CNI — Flannel, Calico, Cilium)
Layer 3: NetworkPolicy (Pod-to-Pod firewall rules)
Layer 2: Node Network (Physical/virtual NIC)
```

## 4.2 — DNS & Service Discovery

> **Ref:** https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

**Say THIS:**

When you create a Service, K8s automatically registers a DNS name for it. You never hardcode IPs
— you use DNS names.

**Key points:**

- **Service DNS format:** `<service-name>.<namespace>.svc.cluster.local`
- Short form: `<service-name>` (same namespace) or `<service-name>.<namespace>` (cross-
namespace).
- Example: `nginx-clusterip.default.svc.cluster.local` → `nginx-clusterip` from within
`default` namespace.

- **CoreDNS** — The cluster DNS server (runs as Pods in `kube-system`). Every Pod's
`/etc/resolv.conf` points to CoreDNS. It resolves Service names to ClusterIPs.

- **Headless Services** (`clusterIP: None`) — No ClusterIP assigned. DNS returns the individual
Pod IPs directly. Used by StatefulSets for stable per-Pod DNS (e.g., `my-pod-0.my-
service.default.svc.cluster.local`).

- **ExternalName Services** — Creates a CNAME record pointing to an external DNS name. Useful
for referencing external databases without hardcoding hostnames in your app.

## 4.3 — Ingress & Ingress Controllers

> **Ref:** https://kubernetes.io/docs/concepts/services-networking/ingress/

**Say THIS:**

A Service (NodePort or LoadBalancer) exposes one app per port. But what if you have 50 apps? You
can't allocate 50 external IPs. **Ingress** solves this — it's a smart reverse proxy that routes
HTTP/HTTPS traffic to different Services based on the hostname or URL path.

**Key points — elaborate on each:**

- **Ingress resource** — A YAML object that declares routing rules. It does **nothing** by
itself — you need an **Ingress Controller** to implement the rules.

- **Ingress Controller** — A Pod running a reverse proxy (nginx, Traefik, HAProxy, Envoy) that
watches Ingress resources and configures itself automatically. k3s ships with **Traefik** pre-
installed.

- **Host-based routing** — Route by hostname: `api.example.com` → API service, `web.example.com`
→ frontend service.

- **Path-based routing** — Route by URL path: `/api` → API service, `/dashboard` → dashboard
service.

- **TLS termination** — Ingress can terminate TLS using a Kubernetes Secret containing the
certificate and key. Upstream traffic to Pods is plain HTTP.

```yaml
# Example: 04-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    # Traefik annotation (k3s default). For nginx-ingress, use:
    # nginx.ingress.kubernetes.io/rewrite-target: /
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik  # k3s default; use "nginx" for nginx-ingress
  rules:
  - host: demo.local  # Route by hostname
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-clusterip
            port:
              number: 80
  # Optional: TLS termination
  # tls:
  # - hosts:
  #   - demo.local
  #   secretName: demo-tls-secret
```

## 4.4 — Network Policies (Firewall Rules)

> **Ref:** https://kubernetes.io/docs/concepts/services-networking/network-policies/

**Say THIS:**

By default, all Pods can talk to all Pods — no restrictions. In production, that's a security
risk. A compromised Pod could scan the entire cluster. **Network Policies** are the K8s firewall
— they control which Pods can communicate with which.

**Key points — elaborate on each:**

- **Network Policies are namespace-scoped.** They select Pods using labels and define allowed
ingress (incoming) and/or egress (outgoing) traffic.

- **Default deny** — Once you apply ANY NetworkPolicy to a Pod, all traffic not explicitly
allowed is denied. This is the critical mental model: "if a policy selects a Pod, it denies
everything except what the policy allows."

- **Requires a CNI that supports them.** Flannel does NOT support Network Policies.
**Calico**, **Cilium**, **Weave Net**, or similar. k3s uses Flannel by default — to use Network
Policies, install Calico or use k3s with `--flannel-backend=none` and install your own CNI.

- **Policy selectors use labels** — Same label-matching system you already know.

```yaml
# Example: 04-networkpolicy-deny-all.yaml — Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}  # Selects ALL Pods in the namespace
  policyTypes:
  - Ingress  # Deny all incoming traffic
  # No ingress rules = nothing is allowed
```

```yaml
# Example: 04-networkpolicy-allow-web.yaml — Allow only specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api  # Apply to Pods with label app=api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web  # Allow traffic FROM Pods with app=web
    ports:
    - protocol: TCP
      port: 8080  # Only on port 8080
```

**Key Network Policy patterns:**

- **Deny all ingress** — Apply an empty ingress policy. Good baseline.
- **Allow from same namespace** — Use `namespaceSelector: {}` to allow all Pods in the same
namespace.
- **Allow from specific namespace** — Use `namespaceSelector` with label matching.
- **Egress policies** — Control outbound traffic. Important for restricting which external
services Pods can reach.
- **Allow DNS** — When restricting egress, always allow UDP port 53 to `kube-system` (CoreDNS),
or DNS resolution breaks.

> **Fun Fact:** Network Policies are additive — if multiple policies select the same Pod, the
UNION of all their rules applies. There's no "deny" rule in Network Policies — only "allow".
Deny is implicit when any policy selects a Pod.

## 4.5 — Live Demo: Ingress + Network Policies

**On shared screen — run these commands:**

```bash
# Ensure the nginx Deployment and ClusterIP Service are running (from Topic 3)
kubectl apply -f examples/03-deployment.yaml
kubectl apply -f examples/03-service-clusterip.yaml

# --- INGRESS DEMO ---

# Apply the Ingress resource
kubectl apply -f examples/04-ingress.yaml

# Check Ingress status
kubectl get ingress
kubectl describe ingress demo-ingress

# Test access (k3s Traefik listens on port 80 of the node)
# Add "demo.local" to /etc/hosts pointing to node IP, or use curl with Host header:
curl -H "Host: demo.local" http://10.227.238.178/

# --- NETWORK POLICY DEMO ---

# NOTE: Network Policies require a CNI that supports them (Calico/Cilium, NOT Flannel).
# If using default k3s (Flannel), this demo shows the YAML but won't enforce.

# Show default: all Pods can talk to all Pods
kubectl run test-client --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec test-client -- wget -qO- --timeout=3 nginx-clusterip:80

# Apply deny-all policy
kubectl apply -f examples/04-networkpolicy-deny-all.yaml

# Now traffic should be blocked (if CNI supports it)
kubectl exec test-client -- wget -qO- --timeout=3 nginx-clusterip:80  # Should timeout

# Apply allow policy for specific Pods
kubectl apply -f examples/04-networkpolicy-allow-web.yaml

# Clean up
kubectl delete pod test-client --ignore-not-found
kubectl delete -f examples/04-networkpolicy-deny-all.yaml --ignore-not-found
kubectl delete -f examples/04-networkpolicy-allow-web.yaml --ignore-not-found
kubectl delete -f examples/04-ingress.yaml --ignore-not-found
```

**While demoing, point out:**

- Ingress is a layer above Services — it routes HTTP traffic based on hostname/path.
- k3s includes Traefik as the default Ingress controller — no extra install needed.
- Network Policies only work with a supporting CNI. k3s default Flannel does NOT enforce them.
For production, use Calico or Cilium.
- The default-deny pattern is the recommended starting point — deny everything, then whitelist.

### Q&A Bullets — Topic 4

- **Q: What's the difference between a Service and an Ingress?**
- Service operates at Layer 4 (TCP/UDP) — routes by port. Ingress operates at Layer 7 (HTTP) —
routes by hostname and URL path. You still need a Service behind the Ingress; the Ingress just
adds smart routing on top.

- **Q: Do I need an Ingress Controller?**
- Yes. The Ingress resource is just a declaration. Without a controller (nginx-ingress,
Traefik, etc.), nothing happens. k3s comes with Traefik; most managed K8s clusters let you
install your choice.

- **Q: Can I use Ingress for non-HTTP traffic?**
- Standard Ingress is HTTP/HTTPS only. For TCP/UDP, use Service type `LoadBalancer` or
`NodePort`. Some controllers (nginx-ingress, Traefik) support TCP/UDP via custom ConfigMaps or
CRDs. The newer **Gateway API** (K8s successor to Ingress) has better support for non-HTTP
protocols.

- **Q: What is the Gateway API?**
- The Gateway API is the next-generation replacement for Ingress. It's more expressive:
supports HTTP, gRPC, TCP, and UDP routing; has better multi-tenancy (Gateway vs Route
separation); and is more portable across implementations. It's currently stable (GA for HTTP
routing since K8s 1.29). If starting a new project, consider Gateway API over Ingress.

- **Q: What happens if no CNI supports Network Policies and I apply one?**
- The NetworkPolicy resource is created and stored in etcd, but nothing enforces it. Traffic
flows freely. This is a silent failure — dangerous in production. Always verify your CNI
supports policies. Test with a deny-all policy and confirm traffic is actually blocked.

- **Q: How do I allow DNS when using egress Network Policies?**
- You must explicitly allow egress to CoreDNS, or all DNS resolution breaks. Allow UDP port 53
to the `kube-system` namespace:
```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

- **Q: Can Network Policies block traffic between namespaces?**
- Yes. Use `namespaceSelector` in your policy rules. Without a `namespaceSelector`, the rule
only matches Pods in the same namespace. This is a powerful isolation mechanism — give each team
their own namespace and lock it down.

- **Q: What's the difference between `pathType: Prefix` and `pathType: Exact` in Ingress?**
- `Prefix` matches the URL path prefix (e.g., `/api` matches `/api`, `/api/users`,
`/api/v2/items`). `Exact` matches only the literal path (e.g., `/api` matches `/api` but NOT
`/api/users`). There's also `ImplementationSpecific` which defers to the Ingress controller's
own matching rules. Most apps use `Prefix`.

- **Q: Do annotations differ between Ingress controllers?**
- Yes — annotations are controller-specific. A Traefik annotation (e.g.,
`traefik.ingress.kubernetes.io/router.entrypoints`) has no effect on nginx-ingress, and vice
versa. This is one reason the **Gateway API** was created — to replace controller-specific
annotations with standardized API fields.

- **Q: What about service mesh (Istio, Linkerd)?**
- Service meshes add Layer 7 policies (mTLS, circuit breaking, retries, observability) using
sidecar proxies. They complement Network Policies (L3/L4) with application-layer controls.
Overkill for simple setups; valuable in large microservice architectures.

- **Q: Can I log Network Policy violations?**
- Not natively with K8s Network Policies. But CNIs like **Calico** and **Cilium** have
logging/audit features that can log denied connections. Cilium uses Hubble for network flow
observability.

---

**TRANSITION:** "We've deployed apps, exposed them via Ingress, and locked them down with
Network Policies. Now let's handle configuration and secrets properly — because hardcoding
config in images is a terrible idea."

**PREV:** [Topic 3 — Pods & Services](03-pods-and-services.md) | **NEXT:** [Topic 5 — Secrets &
Config](05-secrets-config.md)
