# agent.md — Repo Context for AI Agents & Copilots

> This file describes the repository structure, conventions, and rules for AI coding assistants
> working on this project. Always read this file before making changes.

---

## Repository Identity

| Key | Value |
|---|---|
| **Repo** | `https://github.com/hlaingminpaing/argocd-for-devops` |
| **Purpose** | DevOps class demo — GitOps with ArgoCD |
| **Cluster** | Single in-cluster (`https://kubernetes.default.svc`) |
| **ArgoCD namespace** | `argocd` |

---

## Repository Layout

```
argocd/
├── argocd/           ← ArgoCD Application CRDs only (no app code here)
├── helm-chart/       ← Helm chart source (templated approach)
│   └── my-app/
│       ├── values.yaml          ← base defaults
│       ├── values-dev.yaml      ← dev overrides
│       ├── values-uat.yaml      ← uat overrides
│       └── values-prod.yaml     ← prod overrides
└── k8s-manifests/    ← Raw plain YAML approach (no templating)
```

---

## Environments & Namespaces

| Env | ArgoCD App file | K8s Namespace | Ingress class | Auto-sync |
|---|---|---|---|---|
| dev | `argocd/app-helm-dev.yaml` | `my-app-dev` | `traefik` | ✅ |
| uat | `argocd/app-helm-uat.yaml` | `my-app-uat` | `nginx` | ✅ |
| prod | `argocd/app-helm-prod.yaml` | `my-app-prod` | `nginx` | ✅ |
| k8s (raw) | `argocd/app-k8s.yaml` | `my-app-k8s` | `nginx` | ✅ |

---

## Helm Value File Layering Convention

Every ArgoCD Helm Application stacks **two** value files in this exact order:

```yaml
helm:
  valueFiles:
    - values.yaml          # 1st — base defaults (never env-specific)
    - values-<env>.yaml    # 2nd — env overrides (always wins on conflict)
```

**Rule:** Never put environment-specific values in `values.yaml`. Base defaults only.

---

## Coding Conventions

### ArgoCD Application files (`argocd/`)
- One file per environment per approach.
- Always set `namespace: argocd` on the `Application` metadata.
- Always include the cascade-delete finalizer:
  ```yaml
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  ```
- Always set `syncPolicy.syncOptions: [CreateNamespace=true]` so namespaces are auto-created.
- All apps currently use `automated: {prune: true, selfHeal: true}`.

### Helm chart templates (`helm-chart/my-app/templates/`)
- All resource names must use `{{ include "my-app.fullname" . }}` (defined in `_helpers.tpl`).
- All resources must include `{{- include "my-app.labels" . | nindent 4 }}`.
- Conditional resources must be guarded: `{{- if .Values.<flag>.enabled }}`.
- Currently defined templates in `_helpers.tpl`:
  - `my-app.fullname` — `<release>-<chart-name>` truncated to 63 chars
  - `my-app.labels` — full label set including chart version
  - `my-app.selectorLabels` — stable subset used in selectors

### Values files
- `values.yaml` — base, no env-specific data
- `values-dev.yaml` — 1 replica, NodePort, traefik ingress, host `app.demo.local`, `pullPolicy: Always`
- `values-uat.yaml` — 2 replicas, ClusterIP, nginx ingress, host `my-app.uat.example.com`
- `values-prod.yaml` — 3 replicas, ClusterIP, nginx ingress, HPA (3→10), pod anti-affinity

### Raw manifests (`k8s-manifests/`)
- All resources target namespace `my-app-k8s`.
- Do not use Helm templating here — pure YAML only.
- `namespace.yaml` must be applied first (handled automatically by ArgoCD).

---

## Adding a New Environment

1. Create `helm-chart/my-app/values-<newenv>.yaml` with only the overrides needed.
2. Create `argocd/app-helm-<newenv>.yaml`:
   - Set a unique `metadata.name`
   - Set a unique `destination.namespace` (`my-app-<newenv>`)
   - Use `valueFiles: [values.yaml, values-<newenv>.yaml]`
3. Apply: `kubectl apply -f argocd/app-helm-<newenv>.yaml`
4. Update the environment table in both `README.md` and this `agent.md`.

---

## Adding a New Helm Template

1. Create `helm-chart/my-app/templates/<resource>.yaml`.
2. Use `include "my-app.fullname"` for the name and `include "my-app.labels"` for labels.
3. Add the toggle key to `values.yaml` with a sensible default.
4. Override the key per environment in the relevant `values-<env>.yaml`.

---

## Do Not

- ❌ Put secrets or credentials in any file in this repo — use Kubernetes Secrets or an external secrets operator.
- ❌ Use `latest` as an image tag anywhere.
- ❌ Add env-specific values to `values.yaml`.
- ❌ Deploy ArgoCD `Application` resources to any namespace other than `argocd`.
- ❌ Hardcode namespace names inside Helm templates — always drive from `destination.namespace` in the ArgoCD Application.
