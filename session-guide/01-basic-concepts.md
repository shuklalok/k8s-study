# TOPIC 1 — Basic Concepts of K8s (~30 min)

## 1.1 — K8s Architecture

> **Ref:** https://kubernetes.io/docs/concepts/architecture/

**SAY THIS:**

A Kubernetes cluster has two types of machines: the **Control Plane** (the brain) and **Worker
Nodes** (the muscle). Let me draw this out.

*(show this diagram - share your screen or whiteboard)*

```text
  +---------------------------------------------+
  |               Control Plane                 |
  | kube-apiserver etcd scheduler controller-mgr|
  +---------------------------------------------+
        |                 |               |
        v                 v               v
+----Node 1-----+ +----Node 2-----+ +----Node 3-----+
|   kubelet     | |   kubelet     | |   kubelet     |
|  kube-proxy   | |  kube-proxy   | |  kube-proxy   |
|    [Pods]     | |    [Pods]     | |    [Pods]     |
+---------------+ +---------------+ +---------------+
```

**Walk through each component - explain in your own words:**

- **kube-apiserver**
  - The front door. Every `kubectl` command, every internal component, talks to the API server.
  - It's a REST API — you can actually `curl` it. kubectl is just a fancy HTTP client.

- **etcd**
  - The cluster's database. Stores all state: what Pods exist, what Services are defined, who
has access.
  - It's a key-value store using the **Raft consensus** algorithm for consistency.
  - Rule of thumb: if it's not in etcd, K8s doesn't know about it.

- **scheduler**
  - When you create a Pod, the scheduler picks which node it runs on.
  - It considers: available CPU/memory, affinity/anti-affinity rules, taints & tolerations.
  - The scheduler does NOT run the Pod — it just assigns it to a node. The kubelet on that node
does the rest.

- **controller-manager**
  - Runs dozens of control loops, each watching a specific resource type.
  - Deployment controller: ensures the right number of ReplicaSets and Pods.
  - Node controller: detects when a node goes down.
  - Each controller follows the same pattern: observe → compare → act (the **reconciliation loop**).

- **kubelet**
  - Agent running on every worker node.
  - Receives Pod specs from the API server, tells the container runtime (containerd) to start
containers.
  - Reports back: "Container is running" or "Container crashed."

- **kube-proxy**
  - Manages networking rules on each node so that Services (stable IPs) can route to Pods
(ephemeral IPs).
  - Uses iptables or IPVS under the hood.

> **Fun Fact:** **etcd** stands for "/etc distributed" - because `/etc` is where Unix stores
config files. Lose etcd without a backup, and your cluster's state is gone. That's why
production setups run etcd as a 3- or 5-node Raft cluster.

> **Special Note (k3s):** Our k3s cluster replaces etcd with **SQLite** by default (for single-
node setups). For HA, k3s can use embedded etcd or external Postgres/MySQL. Same ops.

## Q&A Bullets - Section 1.1

- **Q: Can the control plane and worker be on the same machine?**
- Yes, in single-node setups like our k3s demo, one machine runs everything. In production,
they're separated for reliability.

- **Q: What happens if the API server goes down?**
- Existing Pods keep running (kubelet manages them locally). But you can't create, update, or
delete anything. No `kubectl` commands work. It's the single most critical component.

- **Q: How is etcd backed up?**
- `etcdctl snapshot save`. In k3s with SQLite, you just copy the SQLite file at
`/var/lib/rancher/k3s/server/db/*. Always have backups.

- **Q: What's the difference between scheduler and controller-manager?**
- Scheduler: "Where should this Pod run?" (placement decision).
- Controller-manager: "Are there enough Pods running? Is the actual state matching desired
state?" (enforcement).

- **Q: What is `kubectl proxy` and when would I use it?**
- `kubectl proxy` creates a local proxy to the API server on `localhost:8001`. You can then
use plain `curl` to hit the K8s REST API without handling TLS certs or auth tokens. Useful for
debugging and exploring the API.

- **Q: Does K8s support Windows nodes?**
- Yes, since K8s 1.18 (stable). You can have a mixed cluster with Linux control plane and
Windows worker nodes. Windows containers run on Windows nodes. But most workloads and tooling
are Linux-first.

## 1.2 — Core Objects — The Building Blocks

> **Ref:**
> - https://kubernetes.io/docs/concepts/overview/working-with-objects/
> - https://kubernetes.io/docs/tutorials/kubernetes-basics/

**SAY THIS:**

Everything in Kubernetes is an **object** — a record of intent you create via YAML. Here are the
ones you'll work with most. We'll go deep on the bold ones in Topics 3 and 5.

**Walk through this table — one object at a time:**

| Object | What it is | Covered in |
|---------|-------------|-------------|
| **Pod** | Smallest deployable unit. One or more containers sharing network + storage. | Topic 3 |
| **ReplicaSet** | Ensures N identical Pods are running at all times. | Topic 3 (via Deployment) |
| **Deployment** | Manages ReplicaSets. Handles rolling updates, rollbacks, scaling. | Topic 3 |
| **Service** | Stable network endpoint (IP + DNS) that routes to a set of Pods. | Topic 3 |
| **Namespace** | Virtual cluster - logical isolation (like folders for K8s objects). | Topic 5 |
| **ConfigMap** | Externalized configuration (key-value pairs or files). | Topic 5 |
| **Secret** | Like ConfigMap, but for sensitive data (passwords, tokens, certs). | Topic 5 |

**Elaborate on Namespaces (since it's not covered later):**
- Namespace: **virtual clusters** inside one physical cluster
- Default namespaces: "default" (your stuff), "kube-system" (K8s internals), "kube-public"
- RBAC, resource quotas, and network policies can all be scoped per namespace.
- Two objects in different namespaces can have the same name without conflict.
- In our demos, we'll use the "default" namespace.

> **Special Note:** You can discover all available resource types with `kubectl api-resources`.
In our cluster you'll also see Traefik CRDs (IngressRoute, Middleware) — those come pre-installed.

## 1.3 — Live Demo: Explore the Cluster

**Switch to shared screen — terminal. Run these one by one, explain each output:**

```bash
# What cluster are we connected to?
kubectl cluster-info

# What nodes (machines) are in the cluster?
kubectl get nodes -o wide

# What namespaces exist?
kubectl get namespaces

# What's running in kube-system? (K8s internal components)
kubectl get pods -n kube-system

# Deploy a simple Pod and inspect it
kubectl apply -f examples/01-basic-pod.yaml
kubectl get pods

kubectl describe pod nginx-basic

# Clean up
kubectl delete -f examples/01-basic-pod.yaml
```

**While running `kubectl describe pod`, point out:**
- Events at the bottom (Scheduled + Pulling + Pulled + Created → Started)
- Labels, IP address, Node assignment
- Container image and port

### Q&A Bullets — Section 1.2 / 1.3

- **Q: What's the difference between a Pod and a container?**
- A container is a single process (like one Docker container). A Pod is a K8s wrapper around
one or more containers that share the same network namespace (same IP, same "localhost") and can
share storage volumes.

- **Q: Why not just use containers directly? Why wrap them in Pods?**
- Pods allow co-located containers (sidecar pattern). They also give K8s a unit to schedule,
monitor, and manage. Health checks, resource limits, and networking are all Pod-level concepts.

- **Q: What are CRDs?**
- Custom Resource Definitions — a way to extend the K8s API with your own object types.
Traefik uses CRDs to define IngressRoutes. Operators (like database operators) use CRDs to
manage stateful apps.

- **Q: Can I run databases in K8s?**
- Yes, using **StatefulSets** (not covered today). StatefulSets provide stable network
identities and persistent storage. But many teams prefer managed databases (RDS, Cloud SQL) and
only run stateless apps in K8s.

## 1.4 — Declarative vs Imperative

> **Ref:** https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/

**Say THIS:**

There are two ways to talk to K8s:

| Approach | How | When to use |
|----------|-----|-------------|
| **Imperative** | `kubectl run`, `kubectl expose` | Quick one-offs, debugging |
| **Declarative** | `kubectl apply -f pod.yaml` | Everything in production |

Always use YAML files checked into Git. Imperative commands are for debugging only.

- Declarative = you describe **what you want** (desired state). K8s figures out **how** to get
there.
- Imperative = you tell K8s **what to do** (create this, delete that). No record of intent.

> **Fun Fact:** `kubectl apply` uses a **three-way merge** — it compares (1) the last-applied
config stored as an annotation, (2) the live state in the cluster, and (3) your new YAML. This
lets it handle fields you didn't change. If you see a `kubectl.kubernetes.io/last-applied-
configuration` annotation on any object, that's the three-way merge at work.

## 1.5 — Labels & Selectors - The Glue

> **Ref:** https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

**Say THIS:**

Labels are how objects **find each other** in Kubernetes. They're simple key-value pairs
attached to any object.

- A **Service** finds its Pods by matching labels.
- A **Deployment** manages ReplicaSets and Pods by matching labels.
- You can filter `kubectl` output by labels: `kubectl get pods -l app=nginx`

```yaml
# On the Pod
metadata:
  labels:
    app: my-api
    env: staging

# On the Service — "send traffic to Pods matching these labels"
selector:
  app: my-api
```

**Key points:**
- Labels are arbitrary — you choose the key names.
- Common ones: `app`, `env`, `version`, `team`. Helm uses `app.kubernetes.io/name`, etc.
- You'll see **equality** (`app=nginx`) and **set-based** (`env in (staging, prod)`) selectors.

> **Fun Fact:** You can have up to **256 labels** per object. Each key is up to 63 characters
(with an optional 253-char domain prefix like `example.com/my-label`). Despite this flexibility,
most objects only have 5-10 labels.

### Q&A Bullets — Sections 1.4 / 1.5

- **Q: What's `kubectl apply` vs `kubectl create` vs `kubectl replace`?**
- `create`: imperative, fails if object exists. `replace`: overwrites the entire object.
- `apply`: declarative three-way merge, safe to re-run. **Always prefer `apply`.**

- **Q: Can I use `kubectl apply` on a whole directory?**
- Yes: `kubectl apply -f examples/` applies every YAML file in the directory. Also supports `-R`
for recursive.

- **Q: What if two Services have the same selector?**
- Both will route to the same Pods. This is valid and sometimes intentional (e.g., one
internal ClusterIP, one external LoadBalancer pointing to the same Pods).

- **Q: What are annotations? How are they different from labels?**
- Annotations are also key-value pairs, but they're for **metadata** (build info, git SHA,
tool configs). They're NOT used for selection — you can't filter with `kubectl -l` on
annotations. Labels = for selection. Annotations = for metadata.

- **Q: What's a DaemonSet?**
- A controller that ensures exactly **one Pod runs on every node** (or a subset of nodes).
Common uses: log collectors (Fluentd), monitoring agents (Prometheus Node Exporter), network
plugins (CNI). Unlike Deployments where you set `replicas`, DaemonSets scale automatically — add
a node, get a Pod; remove a node, lose a Pod.

- **Q: What are Jobs and CronJobs?**
- **Job:** runs a Pod to completion (exit code 0) and doesn't restart it. Use for batch tasks:
data migrations, report generation, backups. Supports parallelism (run N Pods in parallel) and
completion count (ensure N successful runs).
- **CronJob:** creates Jobs on a schedule (cron syntax: `"0 2 * * *"` = 2 AM daily). Use for
recurring tasks. K8s keeps a configurable number of successful/failed Job history for debugging.

- **Q: What are taints and tolerations?**
- **Taints** are set on nodes to repel Pods: `kubectl taint nodes node1 gpu=true:NoSchedule`.
**Tolerations** are set on Pods to allow them onto tainted nodes. Think: taints push Pods away,
tolerations let specific Pods through. Common use: dedicate GPU nodes to ML workloads, keep
control plane nodes Pod-free, or mark nodes for maintenance.

**TRANSITION:** "Now that you know the building blocks, let's talk about Helm — the package
manager that makes deploying complex applications much easier."

---

**PREV:** [Topic 0 - Docker, Images & Containers](00-docker-images-containers.md)
**NEXT:** [Topic 2 - Helm Charts & Deployment](02-helm-charts.md)
