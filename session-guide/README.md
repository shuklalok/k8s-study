# Kubernetes Session — Training Materials

Hands-on Kubernetes training session covering containers, core concepts, workload management,
and configuration. Designed for **online delivery** with a dual-screen setup (presenter guide on
one screen, live demos on the shared screen).

## Topics Covered

| # | Topic | Time |
|---|-------|------|
| 0 | Docker, Images & Containers | ~15 min |
| 1 | Basic Concepts of Kubernetes | ~30 min |
| 2 | Helm Charts & Deployment | ~30 min |
| 3 | Management of Pods & Services | ~35 min |
| 4 | K8s Networking / Firewall Policies | ~30 min |
| 5 | Secrets & Config | ~20 min |
| 6 | High Availability | ~25 min |

> **Note:** Topic 7 (Testing Services via K8s) will be covered in a future session.

## Repository Structure

```
session-guide/              # Presenter guide (split by topic)
├── README.md              # Index — Session Flow + Table of Contents
├── 00-docker-images-containers.md    # Topic 0: Docker, Images & Containers
├── 01-basic-concepts.md               # Topic 1: Basic Concepts of K8s
├── 02-helm-charts.md                 # Topic 2: Helm Charts & Deployment
├── 03-pods-and-services.md           # Topic 3: Pods, Deployments, Services
├── 04-networking.md                  # Topic 4: Networking & Firewall Policies
├── 05-secrets-config.md              # Topic 5: Secrets & Config
├── 06-high-availability.md            # Topic 6: High Availability
├── closing.md                        # Recap, next steps, cleanup
├── yaml-keyword-reference.md         # Alphabetical YAML keyword definitions
├── kubectl-cheatsheet.md            # kubectl quick reference for attendees
├── k3s-setup.md                      # Local k3s cluster setup instructions
└── examples/
    ├── 01-basic-pod.yaml             # Minimal Pod
    ├── 03-multi-container-pod.yaml   # Sidecar pattern
    ├── 03-deployment.yaml            # Deployment with rolling update
    ├── 03-probes.yaml                # Liveness, readiness, startup probes
    ├── 03-service-clusterip.yaml     # ClusterIP Service
    ├── 03-service-nodeport.yaml      # NodePort Service
    ├── 04-ingress.yaml               # Ingress with Traefik
    ├── 04-networkpolicy-deny-all.yaml    # Default deny NetworkPolicy
    ├── 04-networkpolicy-allow-web.yaml   # Allow specific traffic
    ├── 05-configmap.yaml             # ConfigMap definition
    ├── 05-pod-configmap-env.yaml     # Pod consuming ConfigMap as env vars
    ├── 05-pod-configmap-volume.yaml  # Pod consuming ConfigMap as volume
    ├── 05-secret.yaml                # Secret definition
    ├── 05-pod-secret.yaml            # Pod consuming Secret
    ├── 06-pdb.yaml                   # PodDisruptionBudget
    └── 06-hpa.yaml                   # HorizontalPodAutoscaler
```

## Getting Started

### Prerequisites

- Docker installed locally
- kubectl installed
- A running Kubernetes cluster (see [k3s-setup.md](k3s-setup.md) for local setup)

### Session Flow

1. Start with **Topic 0** — Docker basics
2. Progress through **Topics 1-6** in order
3. Each topic includes live demos and Q&A
4. End with **Closing** — recap and cleanup

### For Presenters

- Keep [SESSION-DELIVERY-GUIDE.md](SESSION-DELIVERY-GUIDE.md) open on your private screen
- Use the shared screen for terminal commands and YAML file walkthroughs
- Follow the suggested timing in the delivery guide
- Encourage questions during Q&A sections

### For Attendees

- Pre-read: [kubernetes.io/docs/concepts](https://kubernetes.io/docs/concepts/)
- Have kubectl and Docker installed locally
- Follow along with the live demos
- Reference the [kubectl-cheatsheet.md](kubectl-cheatsheet.md) for common commands

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Artifact Hub](https://artifacthub.io/)
- [k3s Documentation](https://docs.k3s.io/)

## License

This training material is provided for educational purposes.
