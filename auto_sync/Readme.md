# Automated Sync Policy in Argo CD (`Auto-Sync`, `Auto-Prune`, and `Self-Heal`)

This guide outlines how to configure and test Argo CD's automated synchronization features (`Auto-Sync`, `Auto-Prune`, and `Self-Heal`) using a sample Nginx application. 

---

## Directory Structure
```text
argocd/
└── auto_sync/
    ├── auto-sync-app.yml        # Auto-Sync Application manifest
    ├── auto-prune-app.yml       # Auto-Sync + Auto-Prune manifest
    ├── auto-selfheal-app.yml    # Auto-Sync + Self-Heal manifest
    ├── nginx-deployment.yaml    # Nginx Deployment manifest
    ├── nginx-svc.yaml           # Nginx Service manifest
    └── README.md                # Documentation & test guide
```

---

## Prerequisites
* A Kubernetes cluster with Argo CD installed (`argocd` namespace).
* Configured `kubectl` and `argocd` CLI.
* Local clone of your Git repository (`cd ~/argocd/auto_sync`).

---

## Core Scenarios

### 1. Auto-Sync (Git Changes → Cluster)
Monitors Git and automatically applies updates to the cluster without manual intervention.
* **Configuration:** Set `spec.syncPolicy.automated.enabled: true`.
* **Test Flow:** Apply the manifest (`kubectl apply -f auto-sync-app.yml`), update replica counts in `nginx-deployment.yaml`, push to GitHub, and verify via `argocd app get nginx --refresh`. Argo CD syncs the new replica count automatically.
```bash
      # 1. Apply manifest and verify
      kubectl apply -f auto-sync-app.yml
      argocd app get nginx
      kubectl get pods -l app=nginx

      # 2. Modify deployment in Git, commit, and push
      git add nginx-deployment.yaml
      git commit -m "Update nginx replica count"
      git push origin main

      # 3. Refresh and verify sync
      argocd app get nginx --refresh
      kubectl get pods -l app=nginx

```

* **Cleanup:** `kubectl delete -f auto-sync-app.yml`

```bash
      # Cleanup
      kubectl delete -f auto-sync-app.yml
```

### 2. Auto-Prune (Deleted Git Manifests → Cluster Deletion)
Ensures resources removed from Git are purged from the cluster rather than left orphaned.
* **Configuration:** Enable `prune: true` under `automated`.
* **Test Flow:** Apply `auto-prune-app.yml`, delete `nginx-deployment.yaml` from Git, and push. Argo CD automatically deletes the deployment from the cluster while keeping un-deleted resources (like `nginx-svc`) intact.
```bash
      # 1. Apply manifest and check resources
      kubectl apply -f auto-prune-app.yml
      kubectl get deployments,svc

      # 2. Remove deployment from Git, commit, and push
      git rm nginx-deployment.yaml
      git commit -m "Remove nginx-deployment.yaml to test auto-prune"
      git push origin main

      # 3. Refresh and verify auto-prune
      argocd app get nginx --refresh
      kubectl get deployments,svc
```

* **Cleanup:** Revert the Git commit and run `kubectl delete -f auto-prune-app.yml`.
```bash
      # Cleanup
      git revert HEAD --no-edit && git push origin main
      kubectl delete -f auto-prune-app.yml
```


### 3. Self-Heal (Manual Cluster Drift → Reverted to Git)
Watches for out-of-band changes (`kubectl scale`, `kubectl edit`) and reverts them to match Git state within ~5 seconds.
* **Configuration:** Enable `selfHeal: true` alongside `enabled: true` and `prune: true`.
* **Test Flow:** Apply `auto-selfheal-app.yml`, manually scale the deployment (`kubectl scale deployment nginx --replicas=5`), and observe Argo CD automatically scale it back down to the Git-defined state.
```bash
      # 1. Apply manifest and check deployment
      kubectl apply -f auto-selfheal-app.yml
      kubectl get deployment nginx

      # 2. Introduce manual drift
      kubectl scale deployment nginx -n default --replicas=5

      # 3. Watch pods revert automatically (~5 seconds)
      kubectl get pods -l app=nginx -w
```


* **Cleanup:** `kubectl delete -f auto-selfheal-app.yml`
```bash
      # Cleanup
      kubectl delete -f auto-selfheal-app.yml
```

---

## Feature Comparison

| Feature | YAML Setting | What Triggers It | Default Behavior (Off) |
| :--- | :--- | :--- | :--- |
| **Auto-Sync** | `automated.enabled: true` | New commit in Git | Marks app `OutOfSync` (manual sync required) |
| **Auto-Prune** | `automated.prune: true` | Manifest deleted from Git | Leaves orphaned resources in cluster |
| **Self-Heal** | `automated.selfHeal: true` | Manual change via `kubectl` | Preserves manual cluster edits |

### Recommended Production Snippet
```yaml
spec:
  syncPolicy:
    automated:
      enabled: true     # Applies new and updated manifests from Git
      prune: true       # Deletes resources removed from Git
      selfHeal: true    # Reverts manual kubectl changes back to Git
```
