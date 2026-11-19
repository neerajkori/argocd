# Argo CD Drift Detection & Local Diff (`driffting`)

This directory demonstrates how to detect configuration drift between local/Git manifests and a live Kubernetes cluster using Argo CD's diff strategies and `argocd app diff`.

## Files

```text
driffting/
├── drifting-app.yml        # Argo CD Application manifest targeting this directory
├── nginx-deployment.yaml   # Nginx Deployment manifest
└── nginx-svc.yaml          # Nginx Service manifest
```

## Prerequisites

- Kubernetes cluster with Argo CD installed in the `argocd` namespace
- [Argo CD CLI](https://readthedocs.io "Argo CD CLI Installation") installed and logged in

## Usage

### 1. Create the Argo CD Application

Apply `drifting-app.yml` to register the `nginx` application in Argo CD:

```bash
kubectl apply -f drifting-app.yml -n argocd
```

*(Note: Ensure `spec.syncPolicy.automated` in `drifting-app.yml` uses valid Argo CD fields such as `prune: true` and `selfHeal: true` rather than `enabled: true`, and update `exclude: '{drifting-app.yml}'` if excluding the Application file.)*

### 2. Simulate Configuration Drift

Modify a field in `nginx-deployment.yaml` or `nginx-svc.yaml` locally (for example, changing `replicas` or adding a label), **without** pushing the change to Git yet.

### 3. Preview Drift Locally Before Pushing

Run `argocd app diff` from the **repository root** (`~/argocd`) so Argo CD can resolve `spec.source.path: driffting`:

```bash
argocd app diff nginx --local "$(dirname "$PWD")" --server-side-generate --local-include "*.yaml"
```

- **No output (exit code `0`)**: Local manifests match the live cluster state (`InSync`).
- **Diff output (exit code `1`)**: Displays the exact `+` / `-` changes that would be applied on the next sync.

### 4. Output of apply command.
```bash
 argocd app diff nginx --local "$(dirname "$PWD")" --server-side-generate --local-include "*.yaml"

===== apps/Deployment default/nginx ======
99c99
<   replicas: 10
---
>   replicas: 1
```

### 5. Reconcile Drift

- **Git-driven sync**: Commit and push your local changes to `main` so Argo CD automatically syncs them.
- **Live cluster drift**: If changes were made directly via `kubectl edit` or `kubectl scale`, Argo CD detects the drift (`OutOfSync`) and automatically reverts it if `selfHeal: true` is enabled.
