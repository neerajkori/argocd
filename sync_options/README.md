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

## Directory structure

Each concept gets its own directory under `apps/`:

- `<concept>-app.yml` — the **Application** manifest (registers the app with Argo CD)
- `cm-<concept>.yaml` — a **ConfigMap** resource the app deploys (the learning resource for the option)
- `nginx-deployment.yaml` / `nginx-svc.yaml` — the standard nginx workload files

## Concepts and their options

| Directory | Option | Resource file | Notes |
|---|---|---|---|
| `apps/apply-out-of-sync/` | `ApplyOutOfSyncOnly=true` | `cm-apply-out-of-sync.yaml` | Only out-of-sync resources are applied during a sync |
| `apps/create-namespace/` | `CreateNamespace=true` | `cm-create-namespace.yaml` | Destination namespace is created during sync if missing |
| `apps/server-side-apply/` | `ServerSideApply=true` | `cm-server-side-apply.yaml` | Uses `kubectl apply --server-side --force-conflicts` instead of client-side apply |
| `apps/prune-confirm/` | `Prune=confirm` | `cm-prune-confirm.yaml` | Pruning needs the `argocd.argoproj.io/deletion-approved: <ISO timestamp>` annotation on the resource |
| `apps/prune-false/` | `Prune=false` | `cm-prune-false.yaml` | Resource is never pruned when it leaves the app source |
| `apps/delete-confirm/` | `Delete=confirm` | `cm-delete-confirm.yaml` | Deleting on app deletion needs the confirmation annotation |
| `apps/delete-false/` | `Delete=false` | `cm-delete-false.yaml` | Resource is retained when the app is deleted (e.g. PVCs) |
| `apps/prune-last/` | `PruneLast=true` | `cm-prune-last.yaml` | Pruning happens as the final wave, after all other resources are healthy |
| `apps/replace-true/` | `Replace=true` | `cm-replace-true.yaml` | `kubectl replace`/`create` is used instead of `apply` — destructive |
| `apps/client-side-apply/` | `ClientSideApplyMigration=true` | `cm-client-side-apply.yaml` | Moves field ownership from client-side apply to Argo CD's manager (transition to server-side apply) |

## How to use them

```bash
# Prerequisites: logged into Argo CD (see the projects/ guides)

# Example — create-namespace
kubectl apply -f sync_options/apps/create-namespace/create-namespace-app.yml -n argocd
argocd app sync create-namespace
argocd app get create-namespace
kubectl get cm -n default
```

Repeat for the other concepts:

```bash
kubectl apply -f sync_options/apps/apply-out-of-sync/apply-out-of-sync-app.yml -n argocd
kubectl apply -f sync_options/apps/server-side-apply/server-side-apply-app.yml -n argocd
kubectl apply -f sync_options/apps/prune-confirm/prune-confirm-app.yml -n argocd
kubectl apply -f sync_options/apps/prune-false/prune-false-app.yml -n argocd
kubectl apply -f sync_options/apps/delete-confirm/delete-confirm-app.yml -n argocd
kubectl apply -f sync_options/apps/delete-false/delete-false-app.yml -n argocd
kubectl apply -f sync_options/apps/prune-last/prune-last-app.yml -n argocd
kubectl apply -f sync_options/apps/replace-true/replace-true-app.yml -n argocd
kubectl apply -f sync_options/apps/client-side-apply/client-side-apply-app.yml -n argocd
```

To observe a sync option: `argocd app get <name>` (status, health), `kubectl get cm -n default`, and the Argo CD UI — the UI highlights resources that have sync options applied.

To try confirmations: add/update the `argocd.argoproj.io/deletion-approved: <ISO timestamp>` annotation on the resource — the sync/delete then proceeds.

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

- Pattern going forward: one concept dir under `apps/` per option; Application manifests inside, resources next to them.
- Resource-level options override app-level ones.
- To change an option later: edit `spec.syncPolicy.syncOptions` (or the resource annotation) and `kubectl apply -f` the app again.
