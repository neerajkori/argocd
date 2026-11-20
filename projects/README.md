# Projects

This directory holds **Argo CD AppProjects** (the access/security scopes that applications belong to) and the **Application** that deploys workloads from this repo into them.

## What is happening here now

- **`argo-dev` AppProject (`dev_project.yaml`)** — a new project for a dev team; this file shows as modified (in progress):
  - Destination: namespace `team1-kndev` on `https://kubernetes.default.svc`.
  - `sourceRepos`: only `https://github.com/neerajkori/argocd.git`.
  - Cluster resource whitelist: only `Namespace` resources with names matching `team1-*`.
  - Cluster resource blacklist: `Namespace` resources matching `kube-*`.
  - Role `dev-project-role` for group `some-user` with policy `p, proj:argo-dev:dev-project-role, applications, *, argo-dev/*, allow` — the dev team may create and manage applications within `argo-dev`.

- **`default` project hardening (`remove_permission_defaultproj.yml`)** — empties Argo CD's built-in `default` project: empty `sourceRepos` and `destinations`, plus a namespace resource blacklist for `*/*`. The goal is to stop using the permissive default project for team apps.

- **`nginx` Application in the `argo-dev` project (`directory-nginx-app.yml`)** — sources from path `projects/`, includes `*.yaml`, excludes the project definitions themselves (`dev_project.yaml`, `directory-nginx-app.yml`, `remove_permission_defaultproj.yml`), and syncs them to `team1-kndev` (`CreateNamespace=true`, automated sync).

- **`nginx-deployment.yaml` / `nginx-svc.yaml`** — the nginx workload this Application deploys: a 3-replica `nginx:latest` Deployment (label `app: nginx`) and a ClusterIP Service on port 80 (selector `run: nginx`).

## Notes

- `dev_project.yaml` is the file being actively worked on right now (modified per git).
- Observation: the Service selector (`run: nginx`) does not match the Deployment pod labels (`app: nginx`) as written — the Service would not select those pods until the selector is aligned.

## Deployment steps (implementation)

To deploy the applications in this directory, apply them against your Argo CD instance in this order.

### Prerequisites

- Argo CD installed in the `argocd` namespace and the `argocd` CLI available.
- Logged in to the server (for a local server, port-forward and log in, as in the other guides):
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443 &
  ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
  argocd login localhost:8080 --username admin --password "$ARGOCD_PWD" --insecure
  ```

### Steps

1. **Create the `argo-dev` AppProject** so the `nginx` Application has a project to belong to (Argo CD validates the app's project on sync):
   ```bash
   kubectl apply -f projects/dev_project.yaml -n argocd
   ```
   This grants destinations to `team1-kndev` on the default cluster, allows only this repo in `sourceRepos`, and adds the `argo-dev` applications role for the `dev-project-role`/`some-user` group.

2. **(Optional) Harden the built-in `default` project** — clear its `sourceRepos`/`destinations` and add the namespace blacklist, so nothing deploys into the default project:
   ```bash
   kubectl apply -f projects/remove_permission_defaultproj.yml -n argocd
   ```

3. **Deploy the `nginx` Application** — it sources `projects/` (the `nginx-deployment.yaml` + `nginx-svc.yaml` above) and syncs them to `team1-kndev`. `CreateNamespace=true` creates that namespace during the sync:
   ```bash
   kubectl apply -f projects/directory-nginx-app.yml -n argocd
   ```

4. **Verify the deployment**:
   ```bash
   argocd app get nginx          # status, sync state, health
   argocd app sync nginx         # manual sync (automated sync is also enabled, so it follows Git)
   kubectl get all -n team1-kndev
   ```

### Notes on the steps

- Order matters: `dev_project.yaml` must be applied before `directory-nginx-app.yml`, because the Application references `project: argo-dev`.
- The Application's directory source excludes the project definitions themselves, so Argo CD does not try to deploy them as workloads.
- Because the project destination allows only `team1-kndev` (whitelist `team1-*`), the app cannot deploy elsewhere, and the `kube-*` blacklist keeps system namespaces out.
