# TOPIC 2 — Helm Charts & Deployment (~30 min)

## 2.1 — What Problem Does Helm Solve?

> **Ref:** [Helm Documentation](https://helm.sh/docs/) | [Artifact Hub](https://artifacthub.io/)

**Say THIS:**

Imagine you want to deploy a typical web application to Kubernetes. You need a Deployment, a
Service, a ConfigMap, maybe a Secret, an Ingress, and a ServiceAccount. That's 5-6 YAML files,
each referencing the others by name. Now multiply that by dev, staging, and prod - each with
different replica counts, image tags, and config values. Managing all those files manually is
painful.

**Helm** solves this by treating a set of Kubernetes resources as a single **package** called a
**chart**. Think of it like `apt` or `yum` for Kubernetes — you install, upgrade, and rollback
entire applications with a single command.

**Key points — elaborate on each:**

- **Without Helm**, deploying a multi-resource app means: write 5+ YAML files, `kubectl apply`
them in the right order, track which version is deployed, manually edit files per environment.
Error-prone and tedious.

- **With Helm:** One command: `helm install my-app ./my-chart --set replicas=3`. Helm renders
the templates, applies them to the cluster, and tracks the release with versioned history.

- **Helm is the de-facto standard.** Most open-source projects ship Helm charts (Prometheus,
Grafana, nginx-ingress, cert-manager, ArgoCD). You'll rarely deploy infra components without it.

> **Fun Fact:** Helm v2 had a server-side component called **Tiller** that ran inside the
cluster and had God-mode RBAC access — a security nightmare. Helm v3 removed Tiller entirely and
is purely client-side. If someone mentions Tiller, it's legacy.

## 2.2 — Helm Concepts: Charts, Releases, Repositories

**Key points — elaborate on each:**

- **Chart** — A package of templated Kubernetes YAML files. Contains everything needed to deploy
an application: templates, default values, metadata, optional dependencies.

- **Release** — A specific installation of a chart in a cluster. You can install the same chart
multiple times with different release names (e.g., "redis-dev", "redis-prod"). Each release is
independently managed and tracked.

- **Repository** — A place where charts are stored and shared. Like Docker Hub for images.
Common repos: Bitnami, Artifact Hub. You can also host your own (S3, GCS, OCI registries).

- **Values** — Configuration inputs that customize the chart for each deployment. Defaults
live in `values.yaml`; you override them with `--set` or `-f custom-values.yaml`.

- **Release Revision** — Every `helm upgrade` creates a new revision. You can roll back to any
previous revision with `helm rollback`.

**HELM WORKFLOW**

```
Chart (template) + Values (config) → Rendered YAML → K8s (kubectl apply)

my-chart/
  templates/
    deployment.yaml
    service.yaml
    ingress.yaml
  values.yaml
  Chart.yaml
```

## 2.3 — Chart Structure

**Say THIS:**

Let's look at what's inside a Helm chart. Every chart follows a standard directory structure.

```
my-chart/
├── Chart.yaml          # Metadata: name, version, description, dependencies
├── values.yaml         # Default configuration values
├── charts/             # Sub-charts (dependencies)
├── templates/          # Kubernetes manifests with Go template syntax
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── helpers.tpl     # Reusable template snippets (partials)
│   └── hpa.yaml
├── NOTES.txt           # Post-install message shown to the user
├── tests/
│   └── test-connection.yaml
└── .helmignore         # Files to exclude from packaging (like .gitignore)
```

**Key files to explain:**

- **`values.yaml`** — Default values. Users override these per environment. This is the main
knob for customization.

- **`Chart.yaml`** — The chart's identity card. Contains `name`, `version` (chart version),
`appVersion` (the app being deployed), and optional `dependencies`.

- **`templates/`** — Kubernetes YAML files with Go template syntax (`{{ .Values.replicas }}`).
Helm renders these by injecting values, then applies the result to the cluster.

- **`helpers.tpl`** — Named templates (partials) for reusable snippets like labels, fullname,
etc. Convention: files starting with `_` are not rendered as K8s resources.

## 2.4 — Values & Templating

**Say THIS:**

The power of Helm is in templating. Instead of hardcoding values, you use placeholders that get
filled in at install time.

**Show this example:**

```yaml
# templates/deployment.yaml (inside a Helm chart)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      containers:
      - name: {{ .Values.app.name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
```

```yaml
# values.yaml (defaults)
replicaCount: 2
app:
  name: my-web-app
image:
  repository: nginx
  tag: "1.27"
service:
  type: ClusterIP
  port: 80
  targetPort: 80
```

**Key points:**

- `{{ .Values.xxx }}` — References values from `values.yaml` or `--set` overrides.
- `{{ .Release.Name }}` — Built-in: the release name you gave at `helm install`.
- `{{ .Chart.Name }}` — Built-in: the chart name from `Chart.yaml`.
- **Conditionals:** `{{ if .Values.ingress.enabled }}...{{ end }}` — Include/exclude resources.
- **Loops:** `{{ range .Values.env }}...{{ end }}` — Generate repeated blocks.
- **Override precedence:** `--set` > `-f custom.yaml` > `values.yaml`. Most specific wins.

> **Fun Fact:** Helm uses Go's `text/template` engine under the hood. The Sprig library adds
100+ extra functions (string manipulation, math, date, crypto). `{{ .Values.name | upper | quote
}}` converts a value to uppercase and wraps it in quotes.

## 2.5 — Live Demo: Install, Upgrade, Rollback

**On shared screen — run these commands:**

```bash
# 1. Add a popular chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. Search for available charts
helm search repo bitnami/nginx
helm search repo bitnami/nginx --versions  # Show all available versions

# 3. Install a chart (creates a "release")
helm install my-nginx bitnami/nginx --set service.type=NodePort

# 4. Check what's deployed
helm list                                      # Shows all releases
kubectl get all -l app.kubernetes.io/instance=my-nginx  # K8s resources created

# 5. Inspect the rendered templates (what Helm actually applied)
helm get manifest my-nginx | head -60

# 6. Check the values used
helm get values my-nginx                      # User-supplied values
helm get values my-nginx --all               # All values (defaults + overrides)

# 7. Upgrade the release (change replica count)
helm upgrade my-nginx bitnami/nginx --set service.type=NodePort --set replicaCount=3

# 8. Check revision history
helm history my-nginx

# 9. Rollback to previous revision
helm rollback my-nginx 1
helm list                                      # Notice the rollback revision

# 10. Clean up
helm uninstall my-nginx
```

**While demoing, point out:**

- `helm install` creates a **release** - a tracked, versioned deployment. Unlike `kubectl apply`,
Helm knows what it deployed and can diff/rollback.

- `helm list` shows all releases with their status, revision, and chart version.

- `helm upgrade` is idempotent — you can run it repeatedly.

- `helm history` shows every revision — great for audit trails.

- `helm uninstall` removes ALL resources created by the release. No orphaned resources.

### Q&A Bullets — Topic 2

- **Q: Helm vs `kubectl apply` — when should I use which?**
- `kubectl apply` is fine for simple, one-off resources (a single Pod, a quick test). Helm
shines when you have multi-resource apps, need environment-specific config, or want
upgrade/rollback history. In production, almost everything uses Helm (or a GitOps tool that
wraps Helm, like ArgoCD or Flux).

- **Q: What's the difference between `helm install` and `helm upgrade --install`?**
- `helm install` fails if the release already exists. `helm upgrade --install` (the `-i`
shorthand) installs if it doesn't exist, upgrades if it does. In CI/CD pipelines, always use
`helm upgrade --install` for idempotency.

- **Q: Can I see what changes a Helm upgrade will make before applying?**
- Yes. Use `helm diff upgrade` (requires the `helm-diff` plugin) to see a colored diff. Or use
`helm upgrade --dry-run --debug` to see the rendered templates without applying. `--dry-run`
also validates against the API server.

- **Q: How do I create my own chart?**
- `helm create my-chart` scaffolds a complete chart with best-practice templates. Customize
`values.yaml` and templates, then install with `helm install my-release ./my-chart`.

- **Q: What are chart dependencies?**
- A chart can depend on other charts (e.g., your app chart depends on a PostgreSQL chart).
Defined in `Chart.yaml` under `dependencies`. Run `helm dependency update` to pull them into
`charts/`. Dependency values can be overridden from the parent chart's `values.yaml`.

- **Q: How do I pass secrets to Helm without putting them in `values.yaml`?**
- Use `--set` from a CI/CD secret variable: `helm install app ./chart --set
db.password=$DB_PASS`. Or use `helm-secrets` plugin which encrypts values files with SOPS/Age.
Never commit plain-text secrets to Git.

- **Q: What's the difference between `chart version` and `appVersion` in Chart.yaml?**
- `version` is the chart's packaging version (the Helm package itself). `appVersion` is the
version of the application being deployed (e.g., nginx 1.27). They're independent — you can bump
the chart version without changing the app, and vice versa.

- **Q: Can Helm deploy CRDs (Custom Resource Definitions)?**
- Yes. Place CRD YAML files in a `crds/` directory at the chart root. Helm installs CRDs on
first install but does NOT upgrade or delete them (by design, to prevent data loss). For CRD
lifecycle management, many projects use separate CRD charts.

- **Q: What is `helm template`?**
- It renders the chart templates locally and outputs the YAML to stdout — without talking to
the cluster. Perfect for debugging templates, CI/CD pipelines, or generating static YAML for
GitOps workflows. Usage: `helm template my-release ./my-chart -f values-prod.yaml`.

---

**TRANSITION:** "Now that you know how to package and deploy applications with Helm, let's go
deeper into managing Pods and Services directly."

**PREV:** [Topic 1 — Basic Concepts](01-basic-concepts.md) | **NEXT:** [Topic 3 — Pods &
Services](03-pods-and-services.md)
