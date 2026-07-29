# TOPIC 6 — High Availability (~25 min)

## 6.1 — What is HA in Kubernetes?

> **Ref:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/

**Say THIS:**

High Availability means your application stays up even when things fail — and things WILL fail.
Nodes crash, disks die, network partitions happen. HA is about building resilience at every
layer: the control plane, the worker nodes, and the applications themselves.

**Key points — elaborate on each:**

- **HA is not one feature — it's a pattern across layers:**
- **Control Plane HA** — Multiple API servers, etcd members, schedulers.
- **Worker Node HA** — Spread Pods across nodes so one node failure doesn't take down the app.
- **Application HA** — Multiple replicas, Pod anti-affinity, Pod Disruption Budgets, auto-
scaling.

- **Single points of failure (SPOF)** — Any component that exists in exactly one instance is a
SPOF. In a non-HA setup: one API server, one etcd, one scheduler. If any of these dies, the
cluster is degraded or down.

- **k3s demo cluster caveat:** Our k3s setup is a single-node cluster — inherently NOT HA. In
production, you'd have 3+ control plane nodes and multiple workers. We'll discuss the
architecture and demo the application-level HA features.

```
HA Kubernetes Architecture

Load Balancer
  (HAProxy/keepalived/
   cloud LB for API)

etcd quorum: 2/3
```

## 6.2 — Control Plane HA

**Key points — elaborate on each:**

- **etcd** — The brain of the cluster. Stores ALL cluster state. For HA, run 3 or 5 etcd members
(always odd for quorum). Quorum = majority must agree. With 3 members, 1 can fail. With 5, 2 can
fail. Running 2 members is WORSE than 1 — no quorum if 1 fails.

- **API Server** — Stateless. Run multiple replicas behind a load balancer. Any API server can
handle any request. This is the easiest component to make HA.

- **Scheduler & Controller Manager** — Run multiple replicas, but only ONE is active (leader
election). If the leader dies, another takes over within seconds. No manual intervention needed.

- **Stacked vs External etcd topology:**

- **Stacked** — etcd runs on the same nodes as the control plane. Simpler to set up, fewer
machines, but a node failure takes out both a control plane member AND an etcd member.
- **External** — etcd runs on dedicated machines. More resilient but requires more
infrastructure.

- **Load Balancer for the API** — kubectl and kubelets talk to the API server. With multiple API
servers, you need a load balancer (HAProxy, keepalived, cloud LB, or k3s's built-in agent load
balancer) in front.

> **Fun Fact:** etcd uses the **Raft consensus algorithm**. Fun to know: the name "Raft" was
chosen by the Stanford team because it's a simpler alternative to Paxos — like choosing a raft
instead of building a yacht. The etcd project itself is named after the Unix `/etc` directory +
`d` for distributed.

## 6.3 — Application-Level HA

> **Ref:** https://kubernetes.io/docs/concepts/workloads/pods/disruptions/

**Say THIS:**

Even with a perfectly HA control plane, your application can still go down if all its Pods land
on the same node and that node crashes. Application-level HA is about ensuring your Pods are
spread out and protected from disruptions.

**Key points — elaborate on each:**

- **Multiple Replicas** — The most basic HA pattern. Run 2+ replicas via Deployment. If one Pod
dies, the others keep serving. K8s self-heals by replacing the dead Pod.

- **Pod Anti-Affinity** — Tell K8s to spread replicas across different nodes (or zones). Without
this, the scheduler might place all replicas on the same node.

```yaml
# Example: Pod anti-affinity - spread replicas across nodes
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: nginx
          topologyKey: kubernetes.io/hostname  # Spread across nodes
          # topologyKey: topology.kubernetes.io/zone  # Spread across AZs
```

- **Pod Disruption Budgets (PDB)** — Limits how many Pods can be disrupted simultaneously during
voluntary disruptions (node drain, cluster upgrade, maintenance). It does NOT protect against
involuntary disruptions (node crash, OOM kill).

```yaml
# Example: 06-pdb.yaml — At least 2 Pods must always be running
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2  # OR use maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
```

- **Horizontal Pod Autoscaler (HPA)** — Automatically scales replicas based on CPU, memory, or
custom metrics. Prevents overload by adding Pods when demand increases, and scales down when
demand drops.

```yaml
# Example: 06-hpa.yaml — Auto-scale based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up when avg CPU > 70%
```

- **Topology Spread Constraints** — More flexible than anti-affinity. Ensures even distribution
of Pods across zones, nodes, or any topology domain with configurable skew.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: nginx
```

**Key HA checklist for applications:**

- ✓ 2+ replicas (3+ for critical services)
- ✓ Pod anti-affinity or topology spread constraints
- ✓ Pod Disruption Budget (PDB)
- ✓ Health probes (liveness + readiness)
- ✓ Resource requests & limits
- ✓ HPA for auto-scaling under load
- ✓ Graceful shutdown handling (`terminationGracePeriodSeconds`)

## 6.4 — Live Demo: PDB & HPA

**On shared screen — run these commands:**

```bash
# Ensure the Deployment is running with 3 replicas
kubectl apply -f examples/03-deployment.yaml
kubectl scale deployment nginx-deploy --replicas=3

kubectl get pods -o wide  # Note which nodes Pods are on

# --- PDB DEMO ---

# Apply the PDB (minimum 2 Pods must always be available)
kubectl apply -f examples/06-pdb.yaml
kubectl get pdb

kubectl describe pdb nginx-pdb  # Shows allowed disruptions

# Try draining a node (simulates maintenance) — PDB limits evictions
# kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# (Skip actual drain on single-node k3s — just show the PDB object)

# --- HPA DEMO ---

# NOTE: Requires metrics-server installed (see k3s-setup.md)

# Apply the HPA
kubectl apply -f examples/06-hpa.yaml
kubectl get hpa  # Shows current metrics and targets

# Watch HPA in action (in a separate terminal)
kubectl get hpa -w

# Generate load to trigger scaling (optional - requires load generator)
# kubectl run load-gen --image=busybox:1.36 --restart=Never -- \
#   /bin/sh -c "while true; do wget -qO- http://nginx-clusterip; done"

# Clean up
kubectl delete -f examples/06-hpa.yaml --ignore-not-found
kubectl delete -f examples/06-pdb.yaml --ignore-not-found
```

**While demoing, point out:**

- PDB protects against voluntary disruptions only (node drain, rolling upgrade). Node crashes
bypass PDB.
- HPA needs metrics-server to read CPU/memory usage. Without it, HPA shows `<unknown>` for
metrics.
- HPA + Deployment + PDB work together: HPA scales replicas, Deployment manages rollouts, PDB
guards minimum availability.
- In production, combine HPA with **Cluster Autoscaler** (adds/removes nodes when Pods can't be
scheduled or nodes are underutilized).

### Q&A Bullets — Topic 6

- **Q: How many control plane nodes should I run?**
- 3 is the standard for HA. Gives you tolerance for 1 failure. 5 gives tolerance for 2
failures but adds latency (more etcd members = slower consensus). Never run 2 — it's worse than
1 because you lose quorum if either fails.

- **Q: What happens when etcd loses quorum?**
- The cluster becomes **read-only**. Existing workloads keep running (kubelet doesn't need
etcd), but you can't create, update, or delete any resources. The API server returns errors.
This is why etcd quorum is critical.

- **Q: What's the difference between `minAvailable` and `maxUnavailable` in PDB?**
- Two ways to express the same constraint. `minAvailable: 2` means "at least 2 Pods must be
running." `maxUnavailable: 1` means "at most 1 Pod can be down." For a Deployment with 3
replicas, both are equivalent. Use whichever reads more naturally.

- **Q: Does HPA work with custom metrics?**
- Yes. HPA v2 (autoscaling/v2) supports CPU, memory, custom metrics (e.g., requests per
second, queue depth), and even external metrics (from Prometheus, Datadog, etc.). You need a
metrics adapter installed (like Prometheus Adapter or KEDA).

- **Q: What's KEDA?**
- KEDA (Kubernetes Event-Driven Autoscaling) extends HPA with event-driven scaling. It can
scale based on external sources: message queue depth (Kafka, RabbitMQ), database queries, HTTP
request rate, cron schedules, and more. It can even scale to zero replicas.

- **Q: Can I run a single-node K8s cluster in production?**
- Technically yes, but it's NOT HA. If that node goes down, everything goes down. Single-node
is only for dev/test. For production: 3+ control plane nodes, 2+ worker nodes minimum.

- **Q: What's the difference between HPA and VPA?**
- **HPA** (Horizontal) scales the number of Pods (more replicas). **VPA** (Vertical Pod
Autoscaler) scales the resources of individual Pods (more CPU/memory). They address different
problems. HPA is GA and widely used. VPA is less mature and can't be used together with HPA on
the same metric without conflicts.

- **Q: What happens during a node failure?**
- The node controller marks the node as `NotReady` after ~40 seconds (`--node-monitor-grace-
period`). K8s then applies taints (`node.kubernetes.io/not-ready`,
`node.kubernetes.io/unreachable`). Pods have a default `tolerationSeconds: 300` (5 minutes) for
these taints — so after ~5 min total, Pods are evicted and the Deployment controller creates
replacements on healthy nodes. During that window, Pods on the failed node are unreachable but
not yet replaced.

- **Q: How does Cluster Autoscaler work?**
- When the scheduler can't place a Pod (not enough resources on any node), Cluster Autoscaler
requests the cloud provider to add a new node. When nodes are underutilized for a period
(default 10 min), it cordons, drains, and removes them. Works with AWS, GCP, Azure, and other
cloud providers.

- **Q: What is `terminationGracePeriodSeconds` and why does it matter for HA?**
- When K8s stops a Pod (rolling update, scale-down, node drain), it sends `SIGTERM` and waits
this many seconds (default 30) for the process to finish. After the grace period, it sends
`SIGKILL`. If your app needs more time to drain connections or finish in-flight requests,
increase this value. Combine with a **`preStop` hook** (e.g., `sleep 5`) to give the Service
time to remove the Pod from Endpoints before the app starts shutting down — this prevents
dropped requests during deploys.

- **Q: What's the difference between voluntary and involuntary disruptions?**
- **Voluntary:** admin-initiated — node drain, rolling update, scale-down, cluster upgrade.
PDB protects against these. **Involuntary:** unexpected — node crash, OOM kill, hardware
failure, kernel panic. PDB does NOT protect against these — only replicas spread across nodes
(anti-affinity) help here.

- **Q: What about multi-cluster HA?**
- For ultimate HA, run workloads across multiple clusters in different regions. Tools:
**Federation v2**, **Admiralty**, **Liqo**, or simply deploy via CI/CD to multiple clusters
behind a global load balancer (DNS-based failover or Cloudflare, AWS Global Accelerator).

---

**PREV:** [Topic 5 — Secrets & Config](05-secrets-config.md) | **NEXT:** [Closing](closing.md)
