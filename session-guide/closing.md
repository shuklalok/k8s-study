# CLOSING (~10 min)

## Recap

| Topic | Key Takeaway |
|-------|--------------|
| **Docker & Containers** | Containers package apps portably. Docker runs them on one machine. |
| **Why K8s** | K8s orchestrates containers across a cluster: scheduling, self-healing, scaling. |
| **Architecture** | Control Plane (apiserver, etcd, scheduler, controllers) + Worker Nodes. |
| **Helm** | Package manager for K8s. Charts, releases, values, upgrade/rollback. |
| **Pods & Deployments** | Never run bare Pods. Use Deployments for scaling, rolling updates, rollbacks. |
| **Networking** | CNI, DNS, Ingress for HTTP routing, Network Policies for firewall rules. |
| **Services** | Give Pods a stable network identity. ClusterIP, NodePort, LoadBalancer. |
| **ConfigMaps & Secrets** | Externalize config. Secrets are base64, NOT encrypted — protect with RBAC. |
| **High Availability** | HA at every layer: control plane, nodes, apps. PDB + HPA + anti-affinity. |

## Recommended Next Steps

- **Read:** [kubernetes.io/docs/concepts](https://kubernetes.io/docs/concepts/) — comprehensive
and well-written.
- **Try:** [Interactive Tutorials](https://kubernetes.io/docs/tutorials/) — browser-based, no
setup needed.
- **Practice:** Set up a local cluster with **k3s** (see `k3s-setup.md` in this repo).
- **Reference:** Keep the `kubectl-cheatsheet.md` and `yaml-keyword-reference.md` handy.
- **Helm:** Explore charts on [Artifact Hub](https://artifacthub.io/) — try installing Prometheus
or Grafana.

## Remaining Topics

| # | Topic | Covers | Builds on |
|---|-------|--------|-----------|
| 7 | Testing Services via K8s | Everything above | All previous topics |

## Session Cleanup (run after session)

```bash
# Delete all demo resources (YAML-based)
kubectl delete -f examples/ --ignore-not-found

# Delete Helm releases (if any remain from Topic 2 demo)
helm uninstall my-nginx 2>/dev/null || true

# Delete ad-hoc Pods (from networking demo)
kubectl delete pod test-client --ignore-not-found
```

---

**PREV:** [Topic 6 — High Availability](06-high-availability.md) | **HOME:** [Session Guide
Index](README.md)
