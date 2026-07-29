# Kubernetes Session — Delivery Guide

> **Presenter screen:** Keep this guide open on your private screen.
> **Shared screen:** Terminal + YAML files in your IDE.
>
> **Pre-read for attendees:** [kubernetes.io/docs/concepts](https://kubernetes.io/docs/concepts/)

## Session Flow

| # | Topic | Time | What you do on shared screen |
|---|-------|------|------------------------------|
| 0 | Docker, Images & Containers | ~15 min | Terminal: `docker` commands |
| 1 | Basic Concepts of K8s | ~30 min | Terminal: `kubectl` + whiteboard |
| 2 | Helm Charts & Deployment | ~30 min | Terminal: `helm` commands + YAML |
| 3 | Management of Pods & Services | ~35 min | IDE: YAML walkthroughs + `kubectl` |
| 4 | K8s Networking / Firewall Policies | ~30 min | IDE: YAML walkthroughs + `kubectl` |
| 5 | Secrets & Config | ~20 min | IDE: YAML walkthroughs + `kubectl` |
| 6 | High Availability | ~25 min | Whiteboard + YAML + `kubectl` |
| - | Q&A / Buffer | ~10 min | Open discussion |

**Total: ~195 min (split across multiple sessions as needed)**

# Table of Contents

- **[Topic 0 — Docker, Images & Containers](session-guide/00-docker-images-containers.md)**
  - 0.1 — What Problem Does Docker Solve?
  - 0.2 — Docker vs VM — Why Containers?
  - 0.3 — Key Docker Commands
  - 0.4 — So... Why Do We Need Kubernetes?
  - Q&A — Topic 0

- **[Topic 1 — Basic Concepts of K8s](session-guide/01-basic-concepts.md)**
  - 1.1 — K8s Architecture (+ Q&A)
  - 1.2 — Core Objects — The Building Blocks
  - 1.3 — Live Demo: Explore the Cluster (+ Q&A)
  - 1.4 — Declarative vs Imperative
  - 1.5 — Labels & Selectors (+ Q&A)

- **[Topic 2 — Helm Charts & Deployment](session-guide/02-helm-charts.md)**
  - 2.1 — What Problem Does Helm Solve?
  - 2.2 — Helm Concepts: Charts, Releases, Repositories
  - 2.3 — Chart Structure
  - 2.4 — Values & Templating
  - 2.5 — Live Demo: Install, Upgrade, Rollback
  - Q&A — Topic 2

- **[Topic 3 — Management of Pods & Services](session-guide/03-pods-and-services.md)**
  - 3.1 — Pod Deep Dive (+ Q&A)
  - 3.2 — Deployments, ReplicaSets & Scaling (+ Q&A)
  - 3.3 — Health Probes (+ Q&A)
  - 3.4 — Resource Requests & Limits (+ Q&A)
  - 3.5 — Services (+ Q&A)

- **[Topic 4 — K8s Networking / Firewall Policies](session-guide/04-networking.md)**
  - 4.1 — Kubernetes Networking Model
  - 4.2 — DNS & Service Discovery
  - 4.3 — Ingress & Ingress Controllers
  - 4.4 — Network Policies (Firewall Rules)
  - 4.5 — Live Demo: Ingress + Network Policies
  - Q&A — Topic 4

- **[Topic 5 — Secrets & Config](session-guide/05-secrets-config.md)**
  - 5.1 — ConfigMaps (+ Q&A)
  - 5.2 — Secrets
  - 5.3 — Best Practices & Production Tips
  - Q&A — Topic 5 (combined)

- **[Topic 6 — High Availability](session-guide/06-high-availability.md)**
  - 6.1 — What is HA in Kubernetes?
  - 6.2 — Control Plane HA
  - 6.3 — Application-Level HA
  - 6.4 — Live Demo: PDB & HPA
  - Q&A — Topic 6

- **[Closing](session-guide/closing.md)**
  - Recap
  - Recommended Next Steps
  - Session Cleanup
