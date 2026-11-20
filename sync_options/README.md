# Sync Options (`sync_options/`)

Learning the Argo CD **Sync Options** — knobs that customize how Argo CD syncs desired state into the cluster.

Reference: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/

## How sync options are set

Two scopes:

1. **Application level** — `spec.syncPolicy.syncOptions`: a list of strings applied to every resource of the app.
2. **Resource level** — the `argocd.argoproj.io/sync-options` annotation on the resource itself, comma-separated values. **A resource-level option overrides the application-level value.**

```yaml
# Application level
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true

# Resource level (annotation) — overrides app-level settings
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false,Replace=true
```

## Examples in this directory

| Application | Resource | Option | What it demos |
|---|---|---|---|
| `apps/create-namespace-app.yml` | `cm-create-ns` | `CreateNamespace=true` | Creates `spec.destination.namespace` (`team1-kndev`) during sync — run this first |
| `apps/prune-last-app.yml` | `cm-prune-last` | `PruneLast=true` | Pruning happens as the final wave after everything else is healthy |
| `apps/apply-out-of-sync-only-app.yml` | `cm-oos` | `ApplyOutOfSyncOnly=true` | During sync, only out-of-sync resources are applied |
| `apps/server-side-apply-app.yml` | `cm-ssa` | `ServerSideApply=true` | Uses `kubectl apply --server-side --force-conflicts` instead of client-side apply |
| `apps/delete-false-app.yml` | `cm-keep` | `Delete=false` | Resource is retained when the app is deleted (e.g. PVCs) |
| `apps/prune-false-app.yml` | `cm-noprune` (annotation) | `Prune=false` | Resource is never pruned when it leaves the app source |

## How to use them

```bash
# Prerequisites: logged into Argo CD (see the projects/ guides).

# 1. Create the namespace — the first app creates team1-kndev during its sync
kubectl apply -f sync_options/apps/create-namespace-app.yml -n argocd
argocd app sync create-namespace-demo
argocd app get create-namespace-demo
kubectl get ns team1-kndev

# 2. Apply the other example apps
kubectl apply -f sync_options/apps/prune-last-app.yml -n argocd
kubectl apply -f sync_options/apps/apply-out-of-sync-only-app.yml -n argocd
kubectl apply -f sync_options/apps/server-side-apply-app.yml -n argocd
kubectl apply -f sync_options/apps/delete-false-app.yml -n argocd
kubectl apply -f sync_options/apps/prune-false-app.yml -n argocd

# 3. Verify what got deployed
kubectl get cm -n team1-kndev
argocd app get prune-last-demo
```

Order: `create-namespace-app` first, because the other apps target `team1-kndev` and that namespace doesn't exist until the first app's sync creates it.

Demos to try:

- **Prune=false**: remove `sync_options/cm-noprune.yaml` from the repo and push (Argo CD reads the repo HEAD), then `argocd app get prune-false-demo` — the app shows OutOfSync and `cm-noprune` is **not** deleted from the cluster, because pruning is skipped for it.
- **Delete=false**: `kubectl delete -f sync_options/apps/delete-false-app.yml -n argocd`, then `kubectl get cm cm-keep -n team1-kndev` — it still exists after the app is deleted.

## Catalog of the sync options (from the Argo CD docs)

| Option | Where | Effect |
|---|---|---|
| `CreateNamespace=true` | app, resource | Create `spec.destination.namespace` during sync if missing. Combine with `managedNamespaceMetadata` to set labels/annotations on the created namespace |
| `Prune=false` | resource | Never prune this resource when it leaves the source |
| `Prune=confirm` | app or resource | Pruning requires manual confirmation: `argocd.argoproj.io/deletion-approved: <ISO timestamp>` |
| `Delete=false` | app or resource | Retain resources when the app is deleted |
| `Delete=confirm` | app or resource | Deleting on app deletion requires manual confirmation |
| `ApplyOutOfSyncOnly=true` | app or resource | Sync only out-of-sync resources |
| `PrunePropagationPolicy=background/foreground/orphan` | app | Propagation policy used when pruning |
| `PruneLast=true` | app or resource | Prune as the final wave, after all other resources are healthy |
| `Replace=true` | app or resource | Use `kubectl replace`/`create` instead of `apply` — destructive |
| `Force=true` | resource (with `Replace=true`) | Delete/recreate resources — destructive |
| `ServerSideApply=true` / `ServerSideApply=false` | app or resource | Use `kubectl apply --server-side --force-conflicts`; `=false` disables it per resource |
| `ClientSideApplyMigration` | app | Migrate field ownership from client-side apply to server-side apply |
| `Validate=false` | resource | Skip validation for objects that need `--validate=false` (RawExtension types) |
| `SkipDryRunOnMissingResource=true` | app or resource | Skip the dry run when the CRD isn't present yet (e.g. Gatekeeper ConstraintTemplates) |
| `FailOnSharedResource=true` | app | Fail sync if the resource is already applied by another app |
| `RespectIgnoreDifferences=true` | app | Apply `spec.ignoreDifferences` during the sync stage too |

> `Replace` and `Force` change how changes reach the cluster and can cause outages — try them deliberately, not habitually.

## Related concepts (outside the sync options list)

- **SelfHeal** — `spec.syncPolicy.automated.selfHeal: true`: revert manual cluster drift back to Git (see `auto_sync/Readme.md`)
- **Sync waves** — `argocd.argoproj.io/sync-wave`: order resources during a deploy
- **Sync retries** — retry/backoff settings during a sync

## Notes

- Pattern going forward: Application manifests under `apps/`, resources in the directory root.
- Resource-level options override app-level ones.
- To change an option later: edit `spec.syncPolicy.syncOptions` (or the resource annotation) and `kubectl apply -f` the app again.
