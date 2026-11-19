# Multi-Source Application Deployment in Argo CD (Helm Chart + External Values)

This guide demonstrates how to deploy a Helm chart using **Argo CD's Multiple Sources** feature by separating the **Helm chart repository** from the **environment-specific `values.yaml` repository**.

---

## Architecture & How It Works

In real-world GitOps workflows, upstream or shared Helm charts live in one repository while environment configurations (`dev`, `staging`, `prod`) live in a separate configuration repository.

In this setup:
1. **Source 1 (`https://github.com`)** — Contains the core `podinfo` Helm chart (`Chart.yaml`, `templates/`, and default `values.yaml`).
2. **Source 2 (`https://github.com`)** — Assigned the reference alias `ref: values` and holds our custom production overrides at `multiple_source_app/values-prod-podinfo.yaml`.
3. **Cross-Source Reference (`$values`)** — Source 1 mounts the external values file via `$values/multiple_source_app/values-prod-podinfo.yaml` during `helm template` rendering.

---

## Repository Structure

```text
argocd/
└── multiple_source_app/
    ├── values-prod-podinfo.yaml          # Production Helm overrides for podinfo
    ├── podinfo-app.yaml                  # Argo CD Multi-Source Application manifest
    └── README.md                         # Documentation & implementation guide
```

---

## Step-by-Step Implementation

### Step 1: Prepare the Custom Values Repository
Ensure your production overrides file (`values.yaml`) is committed and pushed inside `multiple_source_app/` in the `neerajkori/argocd` repository:

```bash
cd ~/argocd/multiple_source_app
git add values-prod-podinfo.yaml
git commit -m "Add production values.yaml for podinfo"
git push origin main
```

---

### Step 2: Create the Multi-Source Argo CD `Application` Manifest - as it present in the repo - `helm-podinfo-app-withmultipleSources.yml`

> **Important Notes:**
> * Do **not** add a `path:` field to Source 2 (`ref: values`) unless that directory also contains raw Kubernetes manifests you want Argo CD to deploy.
> * The file path referenced after `$values/` must be committed and pushed to the remote branch (`HEAD`) before syncing.

---

### Step 3: Deploy the Application to Argo CD
Apply the `Application` manifest to your cluster and trigger a sync:

```bash
kubectl apply -f podinfo-app.yaml
argocd app sync podinfo
```

*(If you recently pushed changes to `values.yaml` and Argo CD hasn't picked them up yet, run `argocd app get podinfo --hard-refresh` before syncing.)*

---

### Step 4: Verify the Deployment
1. **Check Argo CD application health and sync status:**
   ```bash
   argocd app get podinfo 
   ```
2. **Verify the running Pods and Service in Kubernetes:**
   ```bash
   kubectl get pods,svc -n default -l app.kubernetes.io/name=podinfo
   ```
3. **Test the live application endpoint:**
   ```bash
   kubectl port-forward svc/podinfo 9898:9898
   curl http://localhost:9898
   ```

---

## Cleanup
To remove the application and all deployed Kubernetes resources:

```bash
kubectl delete -f podinfo-app.yaml
```
