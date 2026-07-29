# TOPIC 3 — Management of Pods & Services (~35 min)

## 3.1 — Pod Deep Dive

> **Ref:** https://kubernetes.io/docs/concepts/workloads/pods/

**Say THIS:**

We talked about Pods in Topic 1 — now let's go deep. A Pod is the smallest thing K8s can deploy.
Most of the time it's one container, but it can hold multiple containers that need to work
tightly together.

**Key points — elaborate on each:**

- **One Pod = one or more containers sharing the same network and storage.**
- All containers in a Pod share `localhost` — they can talk to each other via `127.0.0.1`.
- They can also share storage volumes mounted into each container.

- **Pod lifecycle:** Pending → Running → Succeeded/Failed

- **Pending:** scheduler is finding a node, or images are being pulled.
- **Running:** at least one container is running.
- **Succeeded:** all containers exited with code 0 (for Jobs).
- **Failed:** at least one container exited with a non-zero code.

- A running Pod can also be in `CrashLoopBackOff` — the container keeps crashing and K8s keeps
restarting it with exponential backoff (10s, 20s, 40s, ... up to 5 min).

- **Multi-container patterns:**
- **Sidecar:** helper alongside main app (logging agent, metrics exporter, proxy).
- **Init container:** runs before the main container starts (database migration, config
setup). Must complete successfully before the app starts.

- **Pods are ephemeral — NEVER create bare Pods in production.**
- If a Pod dies, it stays dead. No one recreates it.
- Always use a **Deployment** (or Job, DaemonSet, StatefulSet) to manage Pods.

> **Fun Fact:** A Pod can technically hold hundreds of containers, but the practical limit is
~10-20 due to resource overhead. Since K8s 1.28+, there's native **Sidecar Container** support
via `restartPolicy: Always` on init containers — they start before the main container and stay
running for the Pod's lifetime.

> **Special Note:** When you `kubectl exec` into a multi-container Pod, you **must** specify `-c
<container-name>`. Without it, K8s picks the first container — which might not be the one you
want.

**LIVE DEMO — switch to shared screen:**

```bash
# Apply the multi-container pod (sidecar pattern)
kubectl apply -f examples/03-multi-container-pod.yaml

# Wait and check status — notice READY shows 2/2 (both containers running)
kubectl get pods

# View logs from each container separately
kubectl logs multi-container -c app
kubectl logs multi-container -c sidecar

# Exec into the app container
kubectl exec -it multi-container -c app -- /bin/sh

# Inside: ls /var/log/ (shared volume with sidecar)
# Type 'exit' to leave

# Clean up
kubectl delete -f examples/03-multi-container-pod.yaml
```

**While demoing, point out:**
- READY 2/2 means both containers in the Pod are running.
- The sidecar sees the same files as the app — because they share the `shared-logs` volume.

### Q&A Bullets — Section 3.1

- **Q: What's the difference between a sidecar and an init container?**
- Init container: runs **before** the main container, must finish successfully, then exits.
Use for setup tasks (DB migrations, waiting for dependencies).
- Sidecar: runs **alongside** the main container for the Pod's lifetime. Use for log shipping,
proxying, metrics.

- **Q: What happens when a container in a Pod crashes?**
- K8s restarts it (within the same Pod, on the same node). The restart policy is `Always` by
default for Deployments. After repeated crashes, it enters `CrashLoopBackOff` with exponential
backoff delays.

- **Q: Can containers in a Pod have different images?**
- Yes! That's the whole point. E.g., main app is `my-api:2.0`, sidecar is `fluentd:latest`.
Each container has its own image, command, ports, and resource limits.

- **Q: How do I debug a Pod that's stuck in `Pending`?**
- `kubectl describe pod <name>` — check the **Events** section at the bottom. Common causes:
insufficient CPU/memory on nodes, image pull errors, unschedulable nodes.

- **Q: What's `ImagePullBackOff`?**
- K8s can't pull the container image. Common causes: wrong image name/tag, private registry
without `imagePullSecrets`, network issues, or registry rate limits (Docker Hub has a pull rate
limit for anonymous users). Check Events in `kubectl describe pod`.

- **Q: Can I SSH into a Pod?**
- Not SSH — Pods don't run an SSH daemon. Use `kubectl exec -it <pod> -- /bin/sh` (or
`/bin/bash`). This attaches to the container's process namespace. For multi-container Pods, add
`-c <container-name>`.

## 3.2 — Deployments, ReplicaSets & Scaling

> **Ref:** https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

**Say THIS:**

You should almost never create Pods directly. Instead, you create a **Deployment**, which
manages everything for you.

Here's the hierarchy:

```
Deployment (you manage this)
└── ReplicaSet (K8s manages this)
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
```

**Key points — elaborate on each:**

- Defines: how many replicas, which container image, update strategy, resource limits.
- When you change the image version, the Deployment creates a **new ReplicaSet** and gradually
shifts Pods from old to new — this is a **rolling update**.

- **Deployment** is what you create and apply. It's the top-level controller.
- **ReplicaSet** is managed by the Deployment — you don't touch it directly.
- It ensures exactly N Pods are running. If a Pod dies, the ReplicaSet creates a replacement.
- Old ReplicaSets are kept around for rollback (default: 10 revisions, controlled by
`revisionHistoryLimit`).

- **Scaling:** change the `replicas` count. K8s adds/removes Pods automatically.

- **Rolling update strategy (from our YAML):**
- `maxSurge: 1` — allow 1 extra Pod during the update (4 Pods total if replicas=3).
- `maxUnavailable: 0` — never let available count drop below desired. Zero-downtime.
- This is the **safest** config. It's slower (one Pod at a time) but guarantees no downtime.

- **Rollback:** `kubectl rollout undo deployment/<name>` instantly reverts to the previous
ReplicaSet.

> **Fun Fact:** K8s keeps **10 old ReplicaSets** by default (controlled by
`revisionHistoryLimit`). Each one is a snapshot you can roll back to. Setting it to 0 saves etcd
storage but means no rollback. Most teams use 3-5 in production.

**LIVE DEMO — switch to shared screen:**

```bash
# Create the Deployment (3 replicas of nginx:1.26)
kubectl apply -f examples/03-deployment.yaml

# See the hierarchy: Deployment + ReplicaSet + Pods
kubectl get deploy,rs,pods

# SCALE UP — watch Pods appear
kubectl scale deployment nginx-deploy --replicas=5
kubectl get pods -w  # Ctrl+C to stop watching

# ROLLING UPDATE — change image from 1.26 to 1.27
kubectl set image deployment/nginx-deploy nginx=nginx:1.27
kubectl rollout status deployment/nginx-deploy

# See both old and new ReplicaSets
kubectl get rs

# ROLLBACK to previous version
kubectl rollout undo deployment/nginx-deploy
kubectl rollout history deployment/nginx-deploy

# Scale back down for next demos
kubectl scale deployment nginx-deploy --replicas=3
```

**While demoing, point out:**
- `kubectl get rs` shows two ReplicaSets — old (3 Pods) and new (3 Pods).
- `rollout history` shows revision numbers — you can roll back to any specific revision.

### Q&A Bullets — Section 3.2

- **Q: What's the difference between Deployment and StatefulSet?**
- Deployment: for **stateless** apps. Pods are interchangeable, can be created/destroyed
freely.
- StatefulSet: for **stateful** apps (databases). Provides stable hostnames ("pod-0", "pod-1")
and persistent storage that survives restarts.

- **Q: Can I do a blue-green or canary deployment?**
- Not natively with Deployments (they only support rolling updates and recreate). For blue-
green and canary, use Argo Rollouts or Flagger — or manage it with multiple Deployments +
Service selector switching.

- **Q: What's `kubectl rollout restart`?**
- It triggers a rolling restart of all Pods without changing the spec. Useful to pick up new
ConfigMap/Secret values — needed when using env vars (they never auto-update) AND when using
volume mounts with apps that don't re-read config files on the fly.

- **Q: How does Horizontal Pod Autoscaler (HPA) work?**
- HPA watches CPU/memory utilization and automatically adjusts `replicas`. It needs metrics-
server installed (which our k3s cluster already has). Config: `kubectl autoscale deployment
nginx-deploy --min=2 --max=10 --cpu-percent=80`.

## 3.3 — Health Probes

> **Ref:** https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/

**Say THIS:**

K8s can monitor your containers and take action when things go wrong. It does this via
**probes** — health checks that kubelet runs against your containers.

**Three types of probes:**

| Probe | Question it answers | What happens on failure |
|-------|---------------------|-------------------------|
| **startupProbe** | "Is the app done booting?" | Other probes are **paused** until this passes. |
| **livenessProbe** | "Is the process healthy?" | Container is **killed and restarted**. |
| **readinessProbe** | "Can it handle traffic?" | Pod is **removed from Service endpoints**. |

**Order of execution:** startup → (once passed) + liveness + readiness run in parallel.

**Three mechanisms for probes:**

- **httpGet** — send an HTTP GET to a path/port. Success = 2xx/3xx. Most common.
- **tcpSocket** — check if a TCP port is open. Good for databases, gRPC services.
- **exec** — run a command inside the container. Success = exit code 0. Use sparingly
(overhead).

**Key parameters:**

- `initialDelaySeconds` — wait before first probe (give the app time to start).
- `periodSeconds` — how often to probe (default 10s).
- `failureThreshold` — how many consecutive failures before taking action (default 3).

> **Fun Fact:** Before `startupProbe` was added (K8s 1.18), teams had to set long
`initialDelaySeconds` on liveness probes for slow-starting apps (Java Spring Boot can take 30-
60s). Problem: if the app crashed *after* startup, detection was equally delayed. `startupProbe`
elegantly decouples boot-time tolerance from runtime health checking.

**LIVE DEMO:**

```bash
kubectl apply -f examples/03-probes.yaml

kubectl describe pod probes-demo  # Show the probe config

kubectl get pods -w  # Watch — Pod should stay Running (nginx starts fast)
```

**While demoing `kubectl describe`, point out the probe section:**
- Liveness, Readiness, Startup — all configured with `httpGet` on port 80.
- The delay, period, and failure threshold values.

### Q&A Bullets — Section 3.3

- **Q: What if I don't set any probes?**
- K8s assumes the container is always healthy and always ready.
- Crashes are detected only when the process exits. Hung/deadlocked processes won't be restarted,
and broken instances keep receiving traffic.

- **Q: Should every container have all three probes?**
- At minimum: **readinessProbe** (to stop sending traffic to broken Pods). Add
**livenessProbe** for apps that can deadlock. Add **startupProbe** only if boot time is long or
variable.

- **Q: What's the difference between a container restart and a Pod restart?**
- There is no "Pod restart." K8s restarts **containers** within a Pod. The Pod stays on the
same node with the same IP. If the *node* fails, then the Deployment creates an entirely new Pod
on a different node.

- **Q: Can I use a custom health endpoint instead of `/healthz`?**
- Yes. The `path` in `httpGet` is completely up to you. Common patterns: `/health`,
`/healthz`, `/ready`, `/actuator/health` (Spring Boot). Some teams use separate endpoints for
liveness vs readiness with different logic.

- **Q: What about gRPC probes?**
- Since K8s 1.24, there's native **gRPC probe** support (set `grpc.port`). Before that, you'd use
an exec probe with `grpc_health_probe` binary. If your app speaks gRPC, use the native probe.

## 3.4 — Resource Requests & Limits

> **Ref:** https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

**Say THIS:**

Every container should declare how much CPU and memory it needs. This is critical for two
reasons: (1) the scheduler uses it to place Pods, and (2) it prevents runaway containers from
starving others.

```yaml
resources:
  requests:  # "I need at least this much" — scheduler guarantee
    cpu: "100m"    # 100 millicores = 10% of one CPU core
    memory: "128Mi" # 128 mebibytes
  limits:    # "Never exceed this" — hard ceiling
    cpu: "500m"
    memory: "256Mi"
```

**Key points to make:**

- **Requests** = guaranteed minimum. The scheduler only places the Pod on a node with this much
free capacity.

- **Limits** = hard ceiling. Exceed memory limit → **OOMKilled** (instant death). Exceed CPU
limit → **throttled** (slowed down, but keeps running).

- **Always set requests.** Without them, the scheduler is flying blind and Pods compete for
resources.

- CPU: `1000m` = 1 full core. `100m` = 10% of a core.
- Memory: use `Mi` (mebibytes, 1024-based), not `M` (megabytes, 1000-based). Avoid confusion.

**K8s Quality of Service (QoS) classes:**

| QoS Class | Condition | Eviction priority |
|-----------|-----------|-------------------|
| **Guaranteed** | requests == limits for all containers | Last to be evicted (highest) |
| **Burstable** | requests < limits (or only requests set) | Middle |
| **BestEffort** | No requests or limits set | First to be evicted (lowest) |

Our `03-deployment.yaml` uses Burstable (requests < limits). For critical production workloads,
use Guaranteed.

> **Fun Fact:** When a node runs low on memory, K8s starts **evicting** Pods. BestEffort Pods go
first, then Burstable (sorted by how much they exceed their requests), and Guaranteed Pods are
evicted last. This is why setting requests correctly is essentially a survival priority ranking.

### Q&A Bullets — Section 3.4

- **Q: What happens if I set limits but no requests?**
- K8s auto-sets requests = limits. The Pod becomes **Guaranteed** QoS. This is actually fine
and safe — just means no bursting above the limit.

- **Q: Should I set CPU limits?**
- Debated. CPU limits cause **throttling** which can hurt latency. Many teams set CPU
**requests** (for scheduling) but leave CPU limits off, only setting memory limits (to prevent
OOM). Google's internal best practice is to not set CPU limits.

- **Q: What does OOMKilled mean?**
- Out Of Memory Killed. The Linux kernel's OOM killer terminates the process when it exceeds
the memory limit. Check with `kubectl describe pod <name>` — you'll see "Reason: OOMKilled" in
the container's last state.

- **Q: How do I know what requests/limits to set?**
- Start with educated guesses, then observe. Use `kubectl top pods` (requires metrics-server)
to see actual CPU/memory usage. Tools like **Vertical Pod Autoscaler (VPA)** can recommend
values based on historical usage. Never guess in production — profile first.

- **Q: What's the difference between "Mi" and "M" in memory?**
- `Mi` = mebibytes (1024-based: 1 Mi = 1,048,576 bytes). `M` = megabytes (1000-based: 1 M =
1,000,000 bytes). Always use `Mi`/`Gi` to avoid subtle mismatches. K8s accepts both but they're
different quantities.

## 3.5 — Services

> **Ref:** https://kubernetes.io/docs/concepts/services-networking/service/

**Say THIS:**

Pods are ephemeral — they get new IPs every time they're recreated. So how does one app talk to
another? **Services** give Pods a **stable network identity** — a fixed IP and DNS name that
doesn't change even as Pods come and go.

**Four types of Services:**

| Type | How it works | Use case |
|------|-------------|----------|
| **ClusterIP** (default) | Internal IP, reachable only inside the cluster | Service-to-service communication |
| **NodePort** | Exposes on every node's IP at a static port (30000-32767) | Quick external access, dev/test |
| **LoadBalancer** | Provisions a cloud load balancer (or ServiceLB in k3s) | Production external traffic |
| **ExternalName** | DNS CNAME alias to an external service | Pointing to external databases |

**How Services find Pods — the label selector:**
- A Service doesn't directly "know" about specific Pods.
- It uses a **label selector** (e.g., `app: nginx`) and automatically routes traffic to all Pods
matching that selector.
- K8s continuously updates the list of matching Pods (called **Endpoints**).

**DNS inside the cluster:**

- Every Service gets a DNS entry: `<service-name>.<namespace>.svc.cluster.local`
- Pods in the **same namespace** can use just `<service-name>` (e.g., `curl http://nginx-
clusterip`).
- This is powered by **CoreDNS** running in `kube-system`.

> **Fun Fact:** K8s also creates DNS **SRV records** for named ports,
so you can technically
discover both the IP and port number of a Service via DNS. This is useful for service meshes and
advanced service discovery.

> **Special Note (k3s):** k3s ships with **ServiceLB** (formerly Klipper), so `LoadBalancer`
type Services actually get an external IP even without a cloud provider — it binds to the node's
IP. On our demo cluster, the NodePort service is accessible at `10.227.238.178:30080`. Try `curl
http://10.227.238.178:30080` during the live demo.

**LIVE DEMO:**

```bash
# Make sure the Deployment is running (3 Pods)
kubectl apply -f examples/03-deployment.yaml
kubectl get pods

# Create a ClusterIP Service
kubectl apply -f examples/03-service-clusterip.yaml

kubectl get svc

kubectl describe svc nginx-clusterip  # Point out: Endpoints list shows Pod IPs

# Create a NodePort Service
kubectl apply -f examples/03-service-nodeport.yaml
kubectl get svc nginx-nodeport  # Note the NodePort: 30080

# DNS resolution from inside the cluster
kubectl run tmp-dns --rm -i --restart=Never --image=busybox:1.36 \
-- nslookup nginx-clusterip.default.svc.cluster.local
```

**While demoing, point out:**

- `kubectl describe svc` shows **Endpoints** — these are the actual Pod IPs that back the
Service.
- If you scale the Deployment, the Endpoints update automatically (try it!).
- The DNS name resolves to the ClusterIP, not to individual Pods.

### Q&A Bullets — Section 3.5

- **Q: What if no Pods match the Service selector?**
- The Service exists but has 0 Endpoints. Requests to it will fail/timeout. Check with
`kubectl describe svc <name>` — Endpoints will be empty.

- **Q: What's the difference between "port" and "targetPort" in a Service?**
- `port`: the port the **Service** listens on (what clients connect to).
- `targetPort`: the port on the **container** where traffic is forwarded. They can be
different (e.g., Service port 80 + container port 8080).

- **Q: Can a Service point to something outside the cluster?**
- Yes. Use `ExternalName` type (DNS CNAME) for simple cases, or create a Service without a
selector and manually define Endpoints pointing to external IPs.

- **Q: What's an Ingress?**
- An Ingress is a layer above Services — it provides HTTP/HTTPS routing (host-based and path-
based) with a single external IP. Think of it as a smart reverse proxy. k3s includes Traefik as
the default Ingress controller. Covered in detail in [Topic 4 — Networking](04-networking.md).

- **Q: What is `terminationGracePeriodSeconds`?**
- When K8s stops a Pod, it sends `SIGTERM` and waits this many seconds (default 30) before
forcefully killing with `SIGKILL`. Set it high enough for your app to finish in-flight requests
and clean up. For HTTP servers, 30s is usually fine. For long-running batch jobs, you may need
120s or more. Goes in `spec.terminationGracePeriodSeconds`.

- **Q: What's `sessionAffinity` on a Service?**
- By default, Services round-robin requests across Pods. Setting `sessionAffinity: ClientIP`
ensures all requests from the same client IP go to the same Pod (sticky sessions). Useful for
legacy apps that store session state in memory, but **avoid if possible** — stateless apps with
external session stores (Redis) scale much better.

- **Q: What's the difference between `kubectl port-forward` and a Service?**
- `port-forward` creates a temporary tunnel from your local machine to a specific Pod or
Service — for debugging only. It bypasses kube-proxy and DNS. A Service is a permanent, in-
cluster routing mechanism. Never use `port-forward` in production — use Services, NodePort, or
Ingress instead.

---

**TRANSITION:** "We've deployed apps and exposed them via Services. Now let's go deeper into
networking — Ingress for HTTP routing and Network Policies for firewalling."

**PREV:** [Topic 2 — Helm Charts](02-helm-charts.md) | **NEXT:** [Topic 4 — Networking](04-networking.md)
