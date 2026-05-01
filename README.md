# ArgoCD GitOps Demo — DevOps Class

> **Repo:** `https://github.com/hlaingminpaing/argocd-for-devops`

This repository demonstrates the **GitOps approach** using ArgoCD with two deployment strategies and three environments (dev / uat / prod).

| Approach | Folder | Description |
|---|---|---|
| **Helm Chart** | `helm-chart/my-app/` | Templated deployment with per-environment value files |
| **Raw K8s Manifests** | `k8s-manifests/` | Plain YAML files, no templating |
| **ArgoCD Apps** | `argocd/` | ArgoCD Application CRDs that wire everything together |

---

## 📁 Project Structure

```
argocd/
├── README.md
├── agent.md                            ← AI/agent context for this repo
│
├── argocd/                             # ArgoCD Application CRDs
│   ├── app-helm.yaml                   # (index) points to env-specific files below
│   ├── app-helm-dev.yaml               # Helm app → dev namespace
│   ├── app-helm-uat.yaml               # Helm app → uat namespace
│   ├── app-helm-prod.yaml              # Helm app → prod namespace
│   └── app-k8s.yaml                    # Raw K8s manifests app → my-app-k8s namespace
│
├── helm-chart/
│   └── my-app/                         # Helm chart for a simple nginx web app
│       ├── Chart.yaml                  # Chart metadata & version
│       ├── values.yaml                 # Base defaults (always applied)
│       ├── values-dev.yaml             # Dev overrides  (layered on top of base)
│       ├── values-uat.yaml             # UAT overrides  (layered on top of base)
│       ├── values-prod.yaml            # Prod overrides (layered on top of base)
│       └── templates/
│           ├── _helpers.tpl            # Reusable name/label helpers
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           └── hpa.yaml                # HorizontalPodAutoscaler (prod only)
│
└── k8s-manifests/                      # Plain Kubernetes YAML files
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## 🌍 Multi-Environment Overview (Helm)

| | Dev | UAT | Prod |
|---|---|---|---|
| **ArgoCD App** | `app-helm-dev.yaml` | `app-helm-uat.yaml` | `app-helm-prod.yaml` |
| **Namespace** | `my-app-dev` | `my-app-uat` | `my-app-prod` |
| **Replicas** | 1 | 2 | 3 (+ HPA up to 10) |
| **Service type** | NodePort | ClusterIP | ClusterIP |
| **Ingress class** | `traefik` | `nginx` | `nginx` |
| **Ingress host** | `app.demo.local` | `my-app.uat.example.com` | `my-app.example.com` |
| **Log level** | `debug` | `info` | `warn` |
| **Auto-sync** | ✅ Yes | ✅ Yes | ✅ Yes |
| **HPA** | ❌ | ❌ | ✅ (CPU 70%) |
| **Anti-affinity** | ❌ | ❌ | ✅ |

### Values file layering

```
values.yaml          ← 1st: base chart defaults (always applied)
values-<env>.yaml    ← 2nd: environment overrides (wins on conflict)
```

---

## 🚀 Quick Start

### Prerequisites
- A running Kubernetes cluster (minikube, kind, k3s, etc.)
- `kubectl` configured
- ArgoCD installed in the cluster

### 1. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080  (admin / <password above>)
```

### 2. Deploy Helm apps (all 3 environments)
```bash
kubectl apply -f argocd/app-helm-dev.yaml
kubectl apply -f argocd/app-helm-uat.yaml
kubectl apply -f argocd/app-helm-prod.yaml
```

### 3. Deploy via Raw K8s Manifests
```bash
kubectl apply -f argocd/app-k8s.yaml
```

### 4. Watch sync status
```bash
kubectl get applications -n argocd
```

---

## 🔄 GitOps Flow

```
Git Repo (hlaingminpaing/argocd-for-devops)
      │
      │  ArgoCD polls every 3 min (or webhook on push)
      ▼
  ArgoCD Server (namespace: argocd)
      │
      ├─── my-app-dev   → namespace: my-app-dev
      ├─── my-app-uat   → namespace: my-app-uat
      ├─── my-app-prod  → namespace: my-app-prod
      └─── my-app-k8s   → namespace: my-app-k8s
```

Any change **committed and pushed** to this repo will be detected and automatically synced by ArgoCD (all envs have `selfHeal: true`).

---

## 🧪 Local Testing with Helm CLI

```bash
# Dry-run render for dev
helm template my-app helm-chart/my-app \
  -f helm-chart/my-app/values.yaml \
  -f helm-chart/my-app/values-dev.yaml

# Dry-run render for prod
helm template my-app helm-chart/my-app \
  -f helm-chart/my-app/values.yaml \
  -f helm-chart/my-app/values-prod.yaml

# Install directly (bypassing ArgoCD)
helm install my-app-dev helm-chart/my-app \
  -n my-app-dev --create-namespace \
  -f helm-chart/my-app/values.yaml \
  -f helm-chart/my-app/values-dev.yaml
```

---

## 📝 Key Concepts

| Concept | Description |
|---|---|
| **Application** | ArgoCD CRD linking a Git source to a K8s destination |
| **Sync** | ArgoCD reconciling live cluster state with Git |
| **Self-Heal** | Automatically corrects manual cluster changes |
| **Prune** | Removes K8s resources that are deleted from Git |
| **valueFiles** | Helm value files layered per environment |
| **HPA** | Kubernetes autoscaler (production only) |
| **GitOps** | Git is the single source of truth for cluster state |
