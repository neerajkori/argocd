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
