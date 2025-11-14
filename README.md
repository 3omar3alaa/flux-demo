# 🚀 Flux CD GitOps Demo (with Kustomize + Helm)

This project is a **fully functional GitOps environment** using:

-   **kind** -- lightweight Kubernetes cluster\
-   **Flux CD** -- GitOps operator\
-   **Kustomize** -- declarative configuration engine\
-   **Helm** -- deploy the `podinfo` application via `HelmRelease`

The project demonstrates GitOps end-to-end:

> You make changes in Git → Flux detects the change → Flux deploys
> automatically → Cluster stays in sync with Git.

------------------------------------------------------------------------

## 📦 Project Structure

    flux-demo/
    ├── apps/
    │   └── podinfo/
    │       ├── namespace.yaml
    │       ├── helmrelease.yaml
    │       └── kustomization.yaml
    └── clusters/
        └── my-cluster/
            └── flux-system/
                ├── gotk-components.yaml
                ├── gotk-sync.yaml
                ├── apps.yaml
                └── kustomization.yaml

------------------------------------------------------------------------

## 1️⃣ Prerequisites

Install the following:

``` bash
# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64 \
  && chmod +x ./kind \
  && sudo mv ./kind /usr/local/bin/kind

# flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# kubectl, git
sudo apt install -y kubectl git
```

------------------------------------------------------------------------

## 2️⃣ Create Kubernetes Cluster

``` bash
kind create cluster --name flux-demo
kubectl get nodes
```

------------------------------------------------------------------------

## 3️⃣ Bootstrap Flux CD to Your GitHub Repo

``` bash
flux bootstrap github \
  --owner=<GITHUB_USERNAME> \
  --repository=flux-demo \
  --branch=main \
  --path=clusters/my-cluster \
  --personal
```

Flux installs itself and creates GitOps wiring in your repo.

------------------------------------------------------------------------

## 4️⃣ Create the Application Structure

### apps/kustomization.yaml

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - podinfo
```

### apps/podinfo/kustomization.yaml

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrelease.yaml
```

### apps/podinfo/namespace.yaml

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo
```

### apps/podinfo/helmrelease.yaml

``` yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 10m
  url: https://stefanprodan.github.io/podinfo
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  interval: 5m
  chart:
    spec:
      chart: podinfo
      version: "6.7.0"
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  values:
    replicaCount: 2
```

------------------------------------------------------------------------

## 5️⃣ Create the Flux Kustomization for Apps

### clusters/my-cluster/flux-system/apps.yaml

``` yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps
  wait: true
```

Add this file to:

### clusters/my-cluster/flux-system/kustomization.yaml

``` yaml
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
  - apps.yaml
```

------------------------------------------------------------------------

## 6️⃣ Commit & Push

``` bash
git add .
git commit -m "Add podinfo GitOps app"
git push
```

Flux will detect changes automatically.

------------------------------------------------------------------------

## 7️⃣ Verify Deployment

``` bash
flux get kustomizations -n flux-system
flux get sources helm -A
flux get helmreleases -A

kubectl get pods -n podinfo
kubectl get svc -n podinfo
```

Port forward to access:

``` bash
kubectl port-forward -n podinfo svc/podinfo 8080:80
```

Open:

👉 http://localhost:8080

------------------------------------------------------------------------

## 8️⃣ Make a GitOps Change

Example: change replica count to 3:

``` yaml
values:
  replicaCount: 3
```

Commit & push:

``` bash
git commit -am "Scale podinfo"
git push
```

Flux will automatically apply the change.

------------------------------------------------------------------------

## 9️⃣ Delete Cluster

``` bash
kind delete cluster --name flux-demo
```

------------------------------------------------------------------------

