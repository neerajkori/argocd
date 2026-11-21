# Argo CD Installation Guide

reference : https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/

All commands assume a configured `kubectl` (run `kubectl cluster-info` first).

---

## 0. The two types of installations (summary)

**two types of installations: multi-tenant and core.**

- **Multi-Tenant (the most common):** one Argo CD instance serving multiple
  developer teams through the Web UI or the `argocd` CLI.

- **Core:** a headless GitOps engine — no API server, no Web UI — for cluster
  administrators who don't need multi-tenancy. 

Bundle contents, per the manifests:

- `install.yaml`: API server + Web UI (`argocd-server`), Application
  Controller, ApplicationSet Controller, Repo Server, Redis, Dex (OIDC),
  Notifications Controller — all in ns `argocd` with cluster-admin RBAC.
- `namespace-install.yaml`: same component set, but only namespace-level RBAC.
- `ha/install.yaml`, `ha/namespace-install.yaml`: same components, multiple
  replicas for the supported components (tuned for HA/resiliency).
- `core-install.yaml`: Application Controller, ApplicationSet Controller,
  Redis, Repo Server — lightweight, non-HA, no API server/UI.

---

## 1. Multi-Tenant — `install.yaml` (non-HA, standard)

> "Standard Argo CD installation with cluster-admin access… if you plan to
> use Argo CD to deploy applications in the same cluster that Argo CD runs
> in… It will still be able to deploy to external clusters with inputted
> credentials."


### 1.1 Architecture

```
        ┌───────────────────────────────────────────┐
        │  End users (developer teams)               │
        │  Web UI or `argocd` CLI + `argocd login`   │
        └──────────────────┬───────────────────────┘
                           │ HTTPS / gRPC
                           ▼
┌──────────────────────────────────────────────────────┐
│  K8s cluster where Argo CD runs  (ns: argocd)        │
│                                                      │
│   ┌─────────────┐ ┌──────┐ ┌──────────┐              │
│   │ Argo CD API │ │ Redis│ │   Repo   │              │
│   │ Server + UI │ │(cache)│ │  Server  │              │
│   ├─────────────┤ └──────┤ ├──────────┤              │
│   │ App. Ctrl   │        │ │  Dex     │              │
│   │ AppSet Ctrl │        │ │ (OIDC)   │              │
│   │ (StatefulSet│        │ ├──────────┤              │
│   │  argocd-app-│        │ │ Notif.   │              │
│   │  lication-  │        │ │  Ctrl    │              │
│   │  controller)│        │ └──────────┘              │
│   └─────────────┘        │                            │
│         │                │                            │
│         └─ in-cluster ──► deploys workloads into      │
│             THIS cluster (kubernetes.svc.default)     │
│             (can also target external clusters with   │
│              inputted credentials)                     │
│                                                       │
│   RBAC: cluster-admin — ClusterRoleBinding → SA      │
│         argocd-application-controller (ns: argocd)   │
└──────────────────────────────────────────────────────┘
```

### 1.2 Step-by-step

```bash
# 1. Create the namespace
kubectl create namespace argocd

# 2. Apply the standard multi-tenant manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait until ready (argocd-server is the API server + Web UI deployment)
kubectl -n argocd wait --for=condition=available deployment/argocd-server --timeout=300s
kubectl -n argocd get pods

# 4. Access the Web UI / API server
kubectl -n argocd port-forward svc/argocd-server 8080:80
#    → open http://localhost:8080 (the Web UI)

#    Or, if you use a LoadBalancer, get the external address:
kubectl -n argocd get svc argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# 5. Log in with the CLI
argocd login localhost:8080 --insecure

#    Initial admin password (stored in the argocd-secret secret, username "admin"):
kubectl -n argocd get secret argocd-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## 2. Multi-Tenant — `namespace-install.yaml` (non-HA, namespace-scoped)

> "Installation of Argo CD which requires only namespace level privileges
> (does not need cluster roles)… if you do not need Argo CD to deploy
> applications in the same cluster that Argo CD runs in, and will rely solely
> on inputted cluster credentials… 
>
> 📌 Note: "Argo CD CRDs are not included into namespace-install.yaml and
> have to be installed separately."

### 2.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Argo CD (ns: argocd) — namespace-level RBAC only         │
│                                                           │
│   ┌────────────┐ ┌────────┐ ┌──────────────────────────┐  │
│   │ App. Ctrl  │ │  Redis │ │      Repo Server         │  │
│   ├────────────┤ └────────┤ ├──────────────────────────┤  │
│   │ AppSet Ctrl│  (cache)│ │  (clones Git repos)      │  │
│   ├────────────┤          │ └──────────────────────────┘  │
│   │ API Server │          │                            │
│   │ + Web UI   │          │                            │
│   ├────────────┤          │                            │
│   │ Dex, Notif.│          │                            │
│   │  Controllers│         │                            │
│   └────────────┘          │                            │
│         │                                                │
│         └─ argocd cluster add <CONTEXT> --in-cluster     │
│            (or inputted cluster credentials)             │
└───────────────────────────┬──────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────┐
│  External K8s clusters — where real app workloads are     │
│  deployed by Argo CD                                      │
│                                                           │
│   ┌────────────┐   ┌────────────┐       ┌────────────┐    │
│   │  Team A    │   │  Team B    │  ...  │  Team N    │    │
│   │ (App/Cfg)  │   │ (App/Cfg)  │       │  (App/Cfg) │    │
│   └────────────┘   └────────────┘       └────────────┘    │
└──────────────────────────────────────────────────────────┘

  📌 CRDs are NOT included in namespace-install.yaml:
  kubectl apply --server-side --force-conflicts -k https://github.com/argoproj/argo-cd/manifests/crds?ref=stable
```

### 2.2 Step-by-step

```bash
# 1. Install the Argo CD CRDs (separately — not in namespace-install.yaml)
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -k https://github.com/argoproj/argo-cd/manifests/crds?ref=stable
#    (fallback — the individual CRD files in manifests/crds, all verified:)
#    kubectl apply --server-side --force-conflicts \
#      -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/application-crd.yaml \
#      -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml \
#      -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/appproject-crd.yaml

# 2. Apply the namespace-scoped manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/namespace-install.yaml

# 3. Verify
kubectl -n argocd get pods
```

---

## 3. Multi-Tenant — HA (`ha/install.yaml`, `ha/namespace-install.yaml`)

> "High Availability installation is recommended for production use. This
> bundle includes the same components but tuned for high availability and
> resiliency."
>
> - `ha/install.yaml` — "the same as install.yaml but with multiple replicas
>   for supported components."
> - `ha/namespace-install.yaml` — "the same as namespace-install.yaml but
>   with multiple replicas for supported components."

### 3.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│  HA Multi-Tenant — production (ha/install.yaml or         │
│  ha/namespace-install.yaml)                               │
│                                                           │
│  Same components as install.yaml / namespace-install.yaml,│
│  but MULTIPLE REPLICAS for supported components, e.g.:    │
│                                                           │
│   ┌────────────┐  ┌────────────┐                          │
│   │ API Server │  │ API Server │        (2+)              │
│   ├────────────┤  ├────────────┤                          │
│   │ App. Ctrl  │  │ App. Ctrl  │        (2+)              │
│   ├────────────┤  ├────────────┤                          │
│   │ AppSet Ctl │  │ AppSet Ctl │        (2+)              │
│   ├────────────┤  ├────────────┤                          │
│   │ Repo Serv. │  │ Repo Serv. │        (2+)              │
│   ├────────────┤  ├────────────┤                          │
│   │    Redis   │  │    Redis   │                          │
│   └────────────┘  └────────────┘                          │
│                                                           │
│  Tuned for high availability and resiliency               │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Step-by-step

```bash
# HA — standard bundle (production-recommended)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml

# HA — namespace-scoped bundle
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -k https://github.com/argoproj/argo-cd/manifests/crds?ref=stable
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/namespace-install.yaml

# Verify + access (same as §1.2 steps 3–4):
kubectl -n argocd get pods
```

---

## 4. Argo CD Core — `core-install.yaml` (headless)

> "The Argo CD Core installation is primarily used to deploy Argo CD in
> headless mode.

### 4.1 Architecture

```
┌──────────────────────────────────────────────────────┐
│  Argo CD Core (core-install.yaml) — headless mode     │
│                                                       │
│   ┌────────────┐ ┌──────────┐ ┌─────────────────────┐ │
│   │ App. Ctrl  │ │   Redis  │ │     Repo Server     │ │
│   ├────────────┤ └──────────┤ ├─────────────────────┤ │
│   │ AppSet Ctl │   (cache)  │ │  (clones Git repos) │ │
│   └────────────┘            │ └─────────────────────┘ │
│                                                       │
│   NO: Argo CD API/UI, Argo CD RBAC, notifications     │
│       controller, OIDC. Lightweight, non-HA.          │
│                                                       │
│   Interact via Application/ApplicationSet CRDs (GitOps)│
└──────────────────────────────────────────────────────┘
```

### 4.2 Step-by-step

```bash
# Install 
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml

# Verify — lightweight, non-HA components:
#   argocd-application-controller (StatefulSet)
#   argocd-applicationset-controller
#   argocd-redis
#   argocd-repo-server
kubectl -n argocd get pods

# Use Core (per the Argo CD Core docs):
kubectl config set-context --current --namespace=argocd
argocd login --core
argocd admin dashboard -n argocd   # Web UI at http://localhost:8080
```

---

## 5. Kustomize approach

> "The Argo CD manifests can also be installed using Kustomize. It is
> recommended to include the manifest as a remote resource and apply
> additional customizations using Kustomize patches."

Install kustomize if you don't have it (e.g.
`go install sigs.k8s.io/kustomize/kustomize/v5@latest`; `kubectl kustomize`
works as a shortcut).

`kustomization.yaml` (example from the docs):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Apply:

```bash
kustomize build . | kubectl apply -f -
# or:
kubectl apply -k .
```

**Installing Argo CD in a custom namespace** — you must patch the
ClusterRoleBinding to reference your namespace (this is why the install.yaml
note warns about the ClusterRoleBinding):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <your-custom-namespace>
resources:
  - https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

patches:
  - patch: |-
      - op: replace
        path: /subjects/0/namespace
        value: <your-custom-namespace>
    target:
      kind: ClusterRoleBinding
```

---

## 6. Helm approach

> "The Argo CD can be installed using Helm. The Helm chart is currently
> community maintained and available at
> [argo-helm/charts/argo-cd](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)."

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo-cd/argo-cd -n argocd --create-namespace

# Verify:
helm -n argocd list
kubectl -n argocd get pods
```

---

## 8. Quick reference — which approach for which situation

| # | Approach | Manifest(s) | Recommended when |
| --- | --- | --- | --- |
| 1 | Multi-Tenant, non-HA | `install.yaml` | Evaluation/demo/testing; same-cluster GitOps with cluster-admin; teams use Web UI / CLI |
| 2 | Multi-Tenant, non-HA | `namespace-install.yaml` | Multiple team instances; namespace-level RBAC; real deployments to external clusters |
| 3 | Multi-Tenant, HA | `ha/install.yaml`, `ha/namespace-install.yaml` | Production — same bundles with multiple replicas |
| 4 | Core (headless) | `core-install.yaml` | Cluster admins; GitOps engine without API server/UI/multi-tenancy |
| 5 | Kustomize | — | You want manifests as remote resources + your own patches (great for custom namespaces) |
| 6 | Helm | `argo-cd` chart | You manage the install via Helm |

---
