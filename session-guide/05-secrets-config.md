# TOPIC 5 — Secrets & Config (~20 min)

## 5.1 — ConfigMaps

> **Ref:** https://kubernetes.io/docs/concepts/configuration/configmap/

**Say THIS:**

In production, you should **never bake configuration into your container image**. The same image
should run across dev, staging, and production — only the configuration changes. This is the [12-
Factor App](https://12factor.net/config) principle.

Kubernetes provides **ConfigMaps** for this — they store configuration data as key-value pairs,
and Pods consume them as environment variables or mounted files.

**Key points — elaborate on each:**

- **ConfigMaps decouple configuration from images.**
- Same `my-app:2.0` image in dev, staging, prod — only the ConfigMap differs.
- Contains simple key-value pairs and/or multi-line files (like `app.properties`).

- **Two ways to consume a ConfigMap:**

- **Environment variables** — injected at Pod startup. Simple, but env vars do NOT auto-update
when the ConfigMap changes. You must restart the Pod (`kubectl rollout restart
deployment/<name>`).

- **Volume mounts** — each key becomes a file in a directory. Volume-mounted ConfigMaps DO
auto-update the **files** (within 30-60 seconds) without restarting the Pod. K8s uses
**symlinks** to swap the data atomically.

- **Critical caveat on volume mount auto-update:**

- Kubernetes updates the **files on disk** — that's it. It does NOT signal, restart, or reload
your application.
- Your app must **actively re-read the files** to pick up changes (e.g., file watcher,
periodic re-read, SIGHUP handler).
- Many apps (nginx, Java Spring Boot without actuator, etc.) read config only at startup — for
those, a volume update alone does **nothing** until the Pod is restarted.
- If your app doesn't support config reload, use `kubectl rollout restart deployment/<name>`
after updating the ConfigMap.

- **Size limit:** 1 MiB (1,048,576 bytes) per ConfigMap. For larger configs, use a
PersistentVolume.

> **Fun Fact:** When mounted as a volume, you'll see symlinks like `..data ->
2026_06_23_09_48_53.xxxxx`. K8s creates a new timestamped directory and atomically swaps the
`..data` symlink. This ensures your app never sees a partially-written file.

**LIVE DEMO:**

```bash
# Create the ConfigMap
kubectl apply -f examples/05-configmap.yaml

kubectl describe configmap app-config  # Show the stored key-value data

# Pod consuming ConfigMap as ENVIRONMENT VARIABLES
kubectl apply -f examples/05-pod-configmap-env.yaml
kubectl logs configmap-env-demo  # Shows APP_ENV=staging, LOG_LEVEL=debug, etc.

# Pod consuming ConfigMap as MOUNTED FILES
kubectl apply -f examples/05-pod-configmap-volume.yaml

kubectl exec configmap-vol-demo -- cat /etc/config/app.properties

kubectl exec configmap-vol-demo -- ls -la /etc/config/  # Show the symlink structure
```

**While demoing, point out:**

- `kubectl describe configmap` shows the raw data — no encoding, plain text.
- The demo uses both `env` (specific keys) AND `envFrom` (all keys). `envFrom` injects all keys,
including the multi-line `app.properties` as an env var. This is usually not done in production;
pick one approach.
- Volume the mount shows symlinks (`..data`, `...`) — explain the atomic swap mechanism.
- Env var approach is simpler but doesn't auto-update. Volume mounts auto-update, but **your app
must re-read them** — K8s does NOT reload your app.

### Q&A Bullets — Section 5.1

- **Q: When should I use env vars vs volume mounts?**
- **Env vars:** for simple key-value config (database URL, feature flags). Easy to use but
doesn't auto-update.
- **Volume mounts:** for config files (app.properties, nginx.conf). Auto-updates and better
for complex configs.

- **Q: How do I update a ConfigMap and have Pods pick it up?**
- Volume-mounted ConfigMaps: K8s updates the **files on disk** automatically in 30-60 seconds.
**But K8s does NOT reload or restart your app.** Your app must re-read the files itself (file
watcher, periodic reload, etc.). If it doesn't, you still need `kubectl rollout restart`
deployment/<name>`.
- Env vars: never auto-update. Always requires Pod restart: `kubectl rollout restart
deployment/<name>`.

- **Q: Can I create a ConfigMap from an existing file?**
- Yes: `kubectl create configmap my-config --from-file=app.properties`. The filename becomes
the key, file contents become the value.

- **Q: What's an immutable ConfigMap?**
- Set `immutable: true` in the ConfigMap spec. After creation, it can never be modified — only
deleted and recreated. This prevents accidental changes and improves performance (kubelet
doesn't need to watch for updates).

- **Q: What happens if I delete a ConfigMap that a Pod references?**
- Existing Pods keep running with whatever values they already have (env vars are baked at
startup, volume-mounted files remain until the Pod restarts). But new Pods or restarted Pods
will fail to start — you'll see `CreateContainerConfigError` in events.

- **Q: Does `subPath` volume mount auto-update?**
- **No.** This is a common gotcha. If you mount a ConfigMap using `subPath` (to mount a single
file without replacing the whole directory), K8s does NOT auto-update that file. Only full-
directory mounts auto-update. This catches many people off guard.

- **Q: What happens if a ConfigMap value is too large?**
- ConfigMaps are limited to 1 MiB total. If you exceed this, the API server rejects the
create/update. For larger data, use a PersistentVolume or bake the file into the image.

## 5.2 — Secrets

> **Ref:** https://kubernetes.io/docs/concepts/configuration/secret/

**Say THIS:**

Secrets are like ConfigMaps, but for **sensitive data** — Passwords, API keys, TLS certificates,
SSH keys.

But here's the critical thing to understand: **Kubernetes Secrets are NOT encrypted by default.
They're base64-encoded.** Base64 is an encoding, not encryption — anyone can decode it. I'll
prove it in the demo.

**Key points — elaborate on each:**

- **Secrets store sensitive data as base64-encoded key-value pairs.**
- `echo -n "password123" | base64` → `cGFzc3dvcmQxMjM=`
- `echo "cGFzc3dvcmQxMjM=" | base64 -d` → `password123`
- Anyone with `kubectl get secret -o yaml` access can decode them.

- **Types of Secrets:**
- `Opaque` — generic, most common (our example).
- `kubernetes.io/tls` — TLS certificate + private key.
- `kubernetes.io/dockerconfigjson` — Docker registry credentials (for private image pulls).

- **Two ways to consume (same as ConfigMap):**

- **Environment variables** — simple, but env vars can
leak into logs, stack traces, and crash dumps. Do NOT auto-update; requires Pod restart.

- **Volume mounts** — stored on **tmpfs** (RAM-backed), never written to disk on the node.
More secure. Files auto-update, but same caveat as ConfigMaps: **your app must re-read the
files** — K8s does NOT reload your app.

- **`stringData` vs `data`:**
- `data` field: you provide base64-encoded values.
- `stringData` field: you provide plain text, K8s auto-encodes on creation. But `stringData`
is **write-only** — when you `kubectl get secret -o yaml`, it always shows `data` (base64).

> **Fun Fact:** When Secrets are mounted as volumes, K8s uses **tmpfs** (RAM-backed filesystem) —
the secret data never hits disk on the node. This is a subtle but important security advantage
over env vars.

**LIVE DEMO:**

```bash
# Create the Secret
kubectl apply -f examples/05-secret.yaml

kubectl get secret db-credentials  # Shows TYPE: Opaque, DATA: 2
kubectl get secret db-credentials -o yaml  # Shows base64-encoded values

# PROVE that base64 is NOT encryption
echo "cGFzc3dvcmQxMjM=" | base64 -d  # Output: password123

# Pod consuming the Secret (both env vars and volume mount)
kubectl apply -f examples/05-pod-secret.yaml
kubectl logs secret-demo  # Shows decoded values

# Show the tmpfs mount
kubectl describe pod secret-demo | grep -A2 "Mounts"
```

**While demoing, point out:**

- `kubectl get secret` doesn't show values by default (only keys and size) — that's intentional.
- `kubectl get secret -o yaml` reveals the base64 values — emphasize that RBAC should restrict who can do this.
- The `base64 -d` demo is the "aha" moment — make sure everyone sees how trivial decoding is.

**Clean up demo Pods (they have `restartPolicy: Never`, so they stay in `Completed` state):**

```bash
kubectl delete pod configmap-env-demo configmap-vol-demo secret-demo --ignore-not-found
```

## 5.3 — Best Practices & Production Tips

> **Ref:** https://kubernetes.io/docs/concepts/security/secrets-good-practices/

**Say THIS:**

Now that you've seen how Secrets work, here are the non-negotiable production practices:

1. **Never commit real Secrets to Git.**
- The YAML in our `examples/` has dummy values for demo purposes.
- In production, use **Sealed Secrets** (Bitnami), **SOPS** (Mozilla), or **External Secrets
Operator** (syncs from Vault, AWS Secrets Manager, etc.).

2. **Use RBAC to restrict who can read Secrets.**
- Not every developer or service account should have `kubectl get secret` access.
- Scope RBAC roles per namespace.

3. **Enable encryption at rest** in etcd.
- Without it, Secrets are stored as base64 in etcd (or SQLite in k3s).
- K8s supports `EncryptionConfiguration` to encrypt before writing to storage.

4. **Prefer volume mounts over env vars for Secrets.**
- Env vars can leak into logs, error reports, and child processes.
- Volume-mounted Secrets use tmpfs (RAM) and are not written to disk.

5. **Use `immutable: true`** for Secrets and ConfigMaps that shouldn't change.
- Prevents accidental modifications. Improves kubelet performance (no watch polling).

6. **Namespace isolation** — keep Secrets in the namespace that needs them.
- A Secret in namespace A is invisible to Pods in namespace B (unless you explicitly grant
RBAC access).

> **Fun Fact:** The `stringData` field in Secret specs accepts plain text and K8s auto-converts
to base64 on creation. But it's write-only — `kubectl get secret -o yaml` always shows the
`data` field. Don't be fooled: the stored value is still just base64.

### Q&A Bullets — Topic 5

- **Q: Are Secrets encrypted in transit?**
- Yes. All communication with the API server uses TLS. Secrets are encrypted in transit
(HTTPS) but may not be encrypted at rest (depends on your cluster config).

- **Q: What's Sealed Secrets?**
- A Bitnami project. You encrypt Secrets on your laptop with a public key, commit the
encrypted `SealedSecret` to Git, and a controller in the cluster decrypts it with the private
key. Only the cluster can decrypt.

- **Q: What's External Secrets Operator?**
- It syncs secrets from external sources (AWS Secrets Manager, HashiCorp Vault, Azure Key
Vault) into K8s Secrets automatically. You define an `ExternalSecret` resource, and the operator
creates and refreshes the K8s Secret.

- **Q: Can Pods from different namespaces share a Secret?**
- No. Secrets are namespace-scoped. If two namespaces need the same Secret, you create a copy
in each (or use External Secrets Operator to sync from a central source).

- **Q: What about environment-specific configs (dev vs staging vs prod)?**
- Use **Kustomize** (built into kubectl) or **Helm** values files to generate different
ConfigMaps/Secrets per environment from the same base templates.

- **Q: Do Secret volume mounts auto-update like ConfigMaps?**
- Yes, same behavior: K8s updates the files on disk (30-60s delay), but your app must re-read
them. Same `subPath` caveat applies — `subPath` mounts do NOT auto-update.

- **Q: What happens if I accidentally delete a Secret that Pods are using?**
- Same as ConfigMaps: running Pods keep their current values (env vars are static, volume
files persist until Pod restarts). But new Pods or restarts will fail with
`CreateContainerConfigError`.

- **Q: Why not just use ConfigMaps for everything, including passwords?**
- ConfigMap data is stored in plain text in etcd and shows up unmasked in `kubectl describe`.
Secrets at least hide values behind base64 in `kubectl get` output and can be encrypted at rest
with `EncryptionConfiguration`. Also, RBAC can be scoped separately for Secrets vs ConfigMaps.

- **Q: What is Kustomize and how is it different from Helm?**
- **Kustomize** is built into `kubectl` (`kubectl apply -k ./`). It works by overlaying
patches on base YAML — no templating, no `{{ }}` syntax. You define a `kustomization.yaml` that
references base resources and applies patches per environment. **Helm** uses Go templates and is
better for packaging reusable charts. Kustomize is simpler for internal apps where you own all
the YAML. Many teams use both: Helm for third-party charts, Kustomize for in-house services.

- **Q: How do I force Pods to restart when a ConfigMap or Secret changes?**
- Best practice: use a **hash suffix** in the ConfigMap/Secret name (e.g., `app-config-
abc123`). When the data changes, the name changes, forcing the Deployment to restart Pods.

---

**TRANSITION:** "We've covered how to configure and secure our applications. Now let's talk about
keeping them running — High Availability."

**PREV:** [Topic 4 — Networking](04-networking.md) | **NEXT:** [Topic 6 — High Availability](06-high-availability.md)
