# ArgoCD GitOps Demo — DevOps Class

This repository demonstrates the **GitOps approach** using ArgoCD with two deployment strategies:

| Approach | Folder | Description |
|---|---|---|
| **Helm Chart** | `helm-chart/my-app/` | Templated, reusable Kubernetes deployment |
| **Raw K8s Manifests** | `k8s-manifests/` | Plain YAML files, no templating |
| **ArgoCD Apps** | `argocd/` | ArgoCD Application CRDs that wire everything together |

---

## 📁 Project Structure

```
argocd/
├── README.md
├── argocd/                         # ArgoCD Application definitions
│   ├── app-helm.yaml               # App pointing to the Helm chart
│   └── app-k8s.yaml                # App pointing to raw K8s manifests
│
├── helm-chart/
│   └── my-app/                     # Helm chart for a simple web app
│       ├── Chart.yaml
│       ├── values.yaml             # Default values (override per environment)
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
│
└── k8s-manifests/                  # Plain Kubernetes YAML files
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
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

### 2. Deploy via Raw K8s Manifests
```bash
kubectl apply -f argocd/app-k8s.yaml
```

### 3. Deploy via Helm Chart
```bash
kubectl apply -f argocd/app-helm.yaml
```

### 4. Watch sync status
```bash
kubectl get applications -n argocd
```

---

## 🔄 GitOps Flow

```
Git Repo (this repo)
      │
      │  ArgoCD watches the repo
      ▼
  ArgoCD Server
      │
      │  Detects drift & syncs
      ▼
  Kubernetes Cluster
```

Any change pushed to this repo will be **automatically detected and applied** by ArgoCD.

---

## 📝 Key Concepts

- **Application** — ArgoCD CRD that links a Git source to a K8s destination
- **Sync** — ArgoCD reconciling the live cluster state with what's in Git
- **Self-Heal** — ArgoCD automatically fixes drift when enabled
- **Helm** — Templated K8s manifests; values can be overridden per environment
- **GitOps** — Git is the single source of truth for cluster state
