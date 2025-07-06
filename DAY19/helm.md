# Helm in Kubernetes

Helm is the package manager for Kubernetes, similar to **`apt`** or **`yum`** for Linux distributions.  
It lets you **define, install, and manage** complex applications using *charts*.

---

## 1 — Key Concepts

| Term | Description |
|------|-------------|
| **Chart** | A collection of files that describe a *related* set of Kubernetes resources (pods, deployments, services, PVCs, etc.). |
| **Release** | A running *instance* of a chart in a cluster. Each upgrade creates a new release version, and you can roll back. |
| **Repository** | A collection of charts. Repos can be public (e.g., Artifact Hub) or private. |

---

## 2 — Architecture

| Version | Client–Server Model | Notes |
|---------|--------------------|-------|
| **Helm 2** | `helm` (client) + **Tiller** (server) deployed inside the cluster. | Tiller managed releases; required cluster‑admin rights. |
| **Helm 3** | **Client‑only** (`helm`) — no Tiller. | Uses Kubernetes APIs directly, RBAC‑friendly, and more secure. |

---

## 3 — Install & Remove Helm 3

```bash
# Download the installer
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
sh get_helm.sh
```

### Uninstall

```bash
sudo rm /usr/local/bin/helm
rm -rf ~/.config/helm
rm get_helm.sh
```

---

## 4 — Chart Directory Structure

```
jio-stg-om-chart/
├── Chart.yaml      # Chart metadata
├── values.yaml     # Default values
├── charts/         # Sub‑charts
└── templates/      # Kubernetes templates
    ├── deployment.yaml
    ├── service.yaml
    └── pv.yaml
```

---

## 5 — Templating

Helm uses **Go templates**. Placeholders (e.g., `{{ .Values.image.tag }}`) are filled with data from **`values.yaml`** or `--set` CLI overrides.

### Example `values.yaml`

```yaml
replicaCount: 2
image:
  repository: my-repo/my-app
  tag: "1.0.0"
service:
  type: LoadBalancer
  port: 80
resources:
  requests:
    cpu: "1"
    memory: 12Gi
```

---

## 6 — Best Practices

- **Version control** your charts.  
- Keep charts **re‑usable** across environments via `values.yaml`.  
- Run `helm lint` & `helm test`.  
- Store secrets in **SealedSecrets / External Secrets** instead of plain YAML.  

---

## 7 — Common Community Charts

- NGINX Ingress
- Prometheus
- Grafana
- MySQL / PostgreSQL
- metrics‑server
- Jenkins  
> Browse on <https://artifacthub.io/>

---

## 8 — Deploying Third‑Party Charts (metrics‑server Example)

### 8.1 — Pre‑flight

```bash
kubectl config current-context   # verify you’re on the right cluster
kubectl top nodes                # will fail until metrics‑server is installed
```

### 8.2 — Add the repo

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo ls
```

### 8.3 — Explore the chart

```bash
helm search repo metrics-server
helm show values metrics-server/metrics-server   # default values
helm template metrics-server/metrics-server      # render manifests locally
```

### 8.4 — Install / Upgrade

```bash
# Dry‑run with two replicas into namespace test-ns
helm upgrade --install metrics-server metrics-server/metrics-server \
  --set replicas=2 \
  -n test-ns --dry-run

# Real install
helm upgrade --install metrics-server metrics-server/metrics-server -n test-ns
```

### 8.5 — Scale & Roll Back

```bash
# Increase to 3 replicas
helm upgrade --install metrics-server metrics-server/metrics-server \
  --set replicas=3

# Roll back to previous version
helm rollback metrics-server
```

### 8.6 — Delete Release

```bash
helm uninstall metrics-server
```

> **Tip:** For many overrides use `-f custom-values.yaml` instead of long `--set` chains.

---

## 9 — Custom Values Example

```yaml
# metricservervalues.yaml
replicas: 2
resources:
  requests:
    cpu: 300m
    memory: 512Mi
```

Apply:

```bash
helm upgrade --install metrics-server -f metricservervalues.yaml \
  metrics-server/metrics-server
```

---

## 10 — Packaging Your Own Application (Java Web App)

### 10.1 — Scaffold a Chart

```bash
helm create javawebapp
```

### 10.2 — Edit `Chart.yaml`

```yaml
apiVersion: v2
name: javawebapp
description: Java web application
version: 1.0.0
appVersion: "1.0.1"
```

### 10.3 — Define `values.yaml`

```yaml
replicaCount: 2
image:
  repository: kkeducation123456/java-web-app
  pullPolicy: IfNotPresent
  tag: "1.0.1"
service:
  type: NodePort
  port: 80
  targetPort: 8080
```

### 10.4 — Update Templates

`templates/deployment.yaml` (excerpt):

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Release.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
```

`templates/service.yaml` (excerpt):

```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
  selector:
    app: {{ .Release.Name }}
```

### 10.5 — Install & Upgrade

```bash
helm install javawebapp ./javawebapp
helm upgrade --install javawebapp ./javawebapp --dry-run
```

---

## 11 — Chart Validation & Distribution

```bash
helm lint javawebapp               # check chart for issues
helm package javawebapp            # creates javawebapp-0.1.0.tgz
helm upgrade --install mavenwebapp mavenwebapp-0.1.0.tgz -n prod
helm status mavenwebapp --show-resources -n prod
helm uninstall mavenwebapp -n prod
```

---

### Cheatsheet

| Command | Purpose |
|---------|---------|
| `helm repo ls` | List repos |
| `helm search repo <keyword>` | Search charts in added repos |
| `helm show values <chart>` | View default `values.yaml` |
| `helm template <chart>` | Render manifests locally |
| `helm ls -A` | List releases (all namespaces) |
| `helm rollback <release>` | Roll back to previous version |

---

**Happy Helming!** :ship::helm:
