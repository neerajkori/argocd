# Multi-Cluster Argo CD ApplicationSet Setup with Minikube

This repository demonstrates how to configure a multi-cluster local Kubernetes setup using **Minikube** (Docker driver) and deploy applications across clusters with **Argo CD ApplicationSets** (`applications_sets/nginx-appset.yml`). 

---

## Architecture Overview

Two separate Minikube profiles communicate over a shared Docker bridge network (`192.168.49.0/24`):
* **`minikube`** (`192.168.49.2`): Acts as the management cluster hosting Argo CD and local workloads.
* **`stg`** (`192.168.49.12`): Acts as the target staging cluster registered to the Argo CD control plane.

---

## Step-by-Step Setup Guide

### 1. Provision Multi-Cluster Minikube Environment
Start the primary management cluster and the secondary staging cluster on the same Docker network with static IPs:
```bash
minikube start -p minikube --driver=docker
minikube start -p stg --driver=docker --network=minikube --static-ip=192.168.49.12
kubectl config use-context minikube
```

### 2. Install and Configure Argo CD
Deploy Argo CD into the management cluster and expose its API server locally:
```bash
kubectl create namespace argocd --context=minikube
kubectl apply -n argocd -f https://githubusercontent.com --context=minikube
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s --context=minikube
kubectl port-forward svc/argocd-server -n argocd 8080:443 --context=minikube &
```
Retrieve the initial admin password and log in via CLI:
```bash
ARGOCD_PWD=\$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" --context=minikube | base64 -d)
argocd login localhost:8080 --username admin --password "\$ARGOCD_PWD" --insecure
```

### 3. Register the Staging Cluster (`stg`)
Point the `stg` cluster configuration to its Docker bridge IP so the Argo CD controller can reach it, then register it:
```bash
kubectl config set-cluster stg --server=https://192.168.49.12:8443 --insecure-skip-tls-verify=true
argocd cluster add stg --name stg
minikube update-context -p stg
kubectl config use-context minikube
```

### 4. Deploy the ApplicationSet
Apply an `ApplicationSet` manifest (referencing your target repository, such as `podinfo` or your `nginx-appset.yml`) to the `argocd` namespace on `minikube`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: podinfo
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: minikube
        url: https://default.svc
      - cluster: stg
        url: https://192.168.49.12:8443
  template:
    metadata:
      name: '{{.cluster}}-podinfo'
    spec:
      project: default
      source:
        repoURL: https://github.com
        targetRevision: HEAD
        path: .
      destination:
        server: '{{.url}}'
        namespace: podinfo
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```
Apply the manifest:
```bash
kubectl apply -f applications_sets/nginx-appset.yml -n argocd --context=minikube
```

### 5. Verify Deployment
Check the synchronization status across your clusters and access the Argo CD dashboard at [https://localhost:8080](https://localhost:8080):
```bash
kubectl get appset,applications -n argocd --context=minikube
argocd cluster list
kubectl get all -n podinfo --context=minikube
kubectl get all -n podinfo --context=stg
```
