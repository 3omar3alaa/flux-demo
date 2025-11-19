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

# 5️⃣ Deploy NGINX Ingress (with admission webhook) via Flux

## 5.1 Files under `apps/ingress-nginx/`

Create the folder:

```bash
mkdir -p apps/ingress-nginx
```

### `apps/ingress-nginx/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
```

### `apps/ingress-nginx/helmrelease.yaml`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 10m
  url: https://kubernetes.github.io/ingress-nginx
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 5m
  timeout: 10m0s
  install:
    createNamespace: false
  chart:
    spec:
      chart: ingress-nginx
      version: "4.11.0"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    controller:
      ingressClassResource:
        name: nginx
        enabled: true
        default: true
      service:
        # On kind, LoadBalancer won't get an external IP. Use NodePort instead.
        type: NodePort
```

### `apps/ingress-nginx/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrelease.yaml
```

Commit & push:

```bash
git add apps/ingress-nginx
git commit -m "Add ingress-nginx HelmRelease"
git push
```

Flux will:

- Create namespace `ingress-nginx`
- Install the ingress-nginx Helm chart
- Configure admission webhooks using chart’s default jobs

Verify:

```bash
flux get helmreleases -n ingress-nginx
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get validatingwebhookconfigurations | grep nginx
```

You should see:

- `ingress-nginx-controller` pod running
- `ingress-nginx-controller-admission` service present
- `ingress-nginx-admission` validating webhook configuration

---

# 6️⃣ Deploy podinfo via HelmRelease

Create:

```bash
mkdir -p apps/podinfo
```

### `apps/podinfo/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo
```

### `apps/podinfo/helmrelease.yaml`

```yaml
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

### `apps/podinfo/ingress.yaml`

Ingress that uses nginx ingress class, no host (path `/`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo
  namespace: podinfo
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: podinfo
                port:
                  number: 80
```

### `apps/podinfo/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrelease.yaml
  - ingress.yaml
  # secret.enc.yaml will be added later
```

Commit & push:

```bash
git add apps/podinfo
git commit -m "Add podinfo app via HelmRelease"
git push
```

Verify:

```bash
flux get helmreleases -A
kubectl get pods -n podinfo
kubectl get svc -n podinfo
kubectl get ing -n podinfo
```

---

# 7️⃣ Install age and SOPS (local machine)

## age

```bash
cd ~
curl -LO https://github.com/FiloSottile/age/releases/download/v1.1.1/age-v1.1.1-linux-amd64.tar.gz
tar -xzf age-v1.1.1-linux-amd64.tar.gz
sudo mv age-v1.1.1-linux-amd64/age /usr/local/bin/
sudo mv age-v1.1.1-linux-amd64/age-keygen /usr/local/bin/
```

Verify:

```bash
age --version
age-keygen --version
```

## SOPS

```bash
curl -LO https://github.com/getsops/sops/releases/download/v3.8.0/sops-v3.8.0.linux.amd64
chmod +x sops-v3.8.0.linux.amd64
sudo mv sops-v3.8.0.linux.amd64 /usr/local/bin/sops
```

Verify:

```bash
sops --version
```

---

# 8️⃣ Generate an age key pair

Run:

```bash
age-keygen -o age.agekey
```

This file looks like:

```text
# created: 2025-11-15T00:00:00Z
# public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AGE-SECRET-KEY-1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

- `public key:` line → SAFE to commit in `.sops.yaml`
- `AGE-SECRET-KEY-...` → PRIVATE, **never commit to Git**

Keep `age.agekey` **locally**; you’ll use it to create a Secret in the cluster.

---

# 9️⃣ Configure SOPS rules

Create `.sops.yaml` at the repo root:

```yaml
creation_rules:
  - path_regex: ".*\\.enc\\.yaml$"
    encrypted_regex: "^(data|stringData)$"
    age:
      - "age1YOUR_PUBLIC_KEY_HERE"
```

Replace `age1YOUR_PUBLIC_KEY_HERE` with the public key printed from `age.agekey`.

Commit & push:

```bash
git add .sops.yaml
git commit -m "Add SOPS configuration"
git push
```

---

# 🔟 Encrypt a podinfo Secret using SOPS

Create an unencrypted secret **locally** (do not commit):

`podinfo-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: podinfo-secret
  namespace: podinfo
type: Opaque
stringData:
  USERNAME: admin
  PASSWORD: mysupersecret
```

Encrypt it using SOPS:

```bash
sops --encrypt podinfo-secret.yaml > apps/podinfo/secret.enc.yaml
rm podinfo-secret.yaml
```

Now `apps/podinfo/secret.enc.yaml` is an encrypted SOPS file.

Update `apps/podinfo/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrelease.yaml
  - ingress.yaml
  - secret.enc.yaml
```

Commit & push:

```bash
git add apps/podinfo/secret.enc.yaml apps/podinfo/kustomization.yaml
git commit -m "Add SOPS-encrypted podinfo secret"
git push
```

At this point, Git contains **only the encrypted secret**, never in plaintext.

---

# 1️⃣1️⃣ Add the age private key to the cluster

Flux needs the **private age key** to decrypt SOPS files.  
You use the local `age.agekey` file to create a Kubernetes Secret directly.

```bash
kubectl create secret generic sops-age \
  -n flux-system \
  --from-file=age.agekey=age.agekey \
  --dry-run=client -o yaml | kubectl apply -f -
```

Check:

```bash
kubectl get secret sops-age -n flux-system
```

You should see `sops-age` in the `flux-system` namespace.

**Note:** `age.agekey` itself is NEVER committed to Git.

---

# 1️⃣2️⃣ Tell Flux to decrypt with SOPS

Edit `clusters/my-cluster/flux-system/apps.yaml` and add the `decryption` block:

```yaml
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
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

Commit & push:

```bash
git add clusters/my-cluster/flux-system/apps.yaml
git commit -m "Enable SOPS decryption for apps Kustomization"
git push
```

Force reconciliation:

```bash
flux reconcile kustomization apps -n flux-system --with-source --force
```

---

# 1️⃣3️⃣ Verify that the secret was decrypted by Flux

Now, the decrypted Secret should exist in the cluster:

```bash
kubectl get secret -n podinfo podinfo-secret
```

Decode the `USERNAME`:

```bash
kubectl get secret -n podinfo podinfo-secret \
  -o jsonpath="{.data.USERNAME}" | base64 --decode
echo
```

Expected output:

```text
admin
```

If you see `ENC[AES256_GCM,...]`, it means Flux is not decrypting (check `kustomize-controller` logs).

---

# 1️⃣4️⃣ (Optional) Use the secret in podinfo environment

To prove the secret is actually usable by the app, you can wire it into the Helm values.

In `apps/podinfo/helmrelease.yaml`, extend `values:`:

```yaml
values:
  replicaCount: 2
  env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: podinfo-secret
          key: USERNAME
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: podinfo-secret
          key: PASSWORD
```

Commit & push:

```bash
git add apps/podinfo/helmrelease.yaml
git commit -m "Wire podinfo-secret into podinfo env"
git push
```

After reconcile, check inside a pod:

```bash
kubectl exec -it -n podinfo deploy/podinfo -- env | grep USERNAME
```

You should see `USERNAME=admin`.

---

# 1️⃣5️⃣ Access podinfo via ingress

Use port-forward from your machine to the ingress controller service:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Then open:

```text
http://localhost:8080
```

You should see the podinfo UI served through NGINX Ingress.

---

# 🧠 Conceptual Notes (Hybrid Explanation)

### What does Kustomize do here?

- `apps/kustomization.yaml` groups `podinfo` and `ingress-nginx`.
- `apps/podinfo/kustomization.yaml` combines the namespace, HelmRelease, ingress, and secret.
- `apps/ingress-nginx/kustomization.yaml` combines the namespace and HelmRelease for ingress-nginx.
- `clusters/my-cluster/flux-system/apps.yaml` is a **Flux Kustomization** that:
  - Points Flux at `./apps`
  - Tells Flux to use SOPS decryption
  - Ensures everything under `apps/` is applied as a unit

### What does Flux actually do?

- **source-controller**:
  - Clones your Git repo into the cluster
- **kustomize-controller**:
  - Reads `Kustomization` objects (like `apps`)
  - Optionally decrypts manifests using SOPS
  - Applies them to the cluster
  - Prunes resources that no longer exist in Git
- **helm-controller**:
  - Reads `HelmRelease` + `HelmRepository`
  - Does `helm install/upgrade` for you
  - Waits for releases to become ready

### Why use Helm instead of plain YAML?

- Real applications have complex manifests (Deployments, Services, Ingress, ConfigMaps, etc.)
- Helm provides:
  - Versioned releases
  - Reusable charts
  - Values-based configuration
  - Easy upgrades & rollbacks

Flux + HelmRelease = GitOps-native Helm.

### Why SOPS?

- You keep **encrypted secrets in Git** (safe for backups, PRs, etc.)
- Decryption happens **inside** the cluster, using the private key in `sops-age`
- Git never contains raw secrets

---

# 🔧 Troubleshooting Cheatsheet

### Force Flux to re-sync

```bash
flux reconcile source git flux-system --with-source
flux reconcile kustomization apps -n flux-system --with-source --force
flux reconcile helmrelease ingress-nginx -n ingress-nginx --with-source
```

### Check Flux logs

```bash
kubectl logs -n flux-system deploy/kustomize-controller
kubectl logs -n flux-system deploy/helm-controller
```

### Debug ingress/webhooks

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get jobs -n ingress-nginx
kubectl logs -n ingress-nginx job/ingress-nginx-admission-create
kubectl logs -n ingress-nginx job/ingress-nginx-admission-patch
kubectl get validatingwebhookconfigurations | grep nginx
```

### Confirm SOPS decryption

- If `kubectl get secret ... -o yaml` still has `ENC[AES256_GCM,...]` in `data/`:
  - Check that:
    - `sops-age` exists in `flux-system`
    - `apps` Kustomization has the `decryption` block
    - `kustomize-controller` logs show no SOPS errors

---

# 🔁 Rebuilding Everything from Scratch

One of the biggest advantages of GitOps:

> You can destroy the cluster and rebuild it from Git.

Steps:

```bash
# 1. Delete cluster
kind delete cluster --name flux-demo

# 2. Recreate cluster
kind create cluster --name flux-demo

# 3. Bootstrap Flux again
flux bootstrap github \
  --owner=<USERNAME> \
  --repository=<REPO> \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# 4. Recreate SOPS age key secret in the cluster (from local age.agekey)
kubectl create secret generic sops-age \
  -n flux-system \
  --from-file=age.agekey=age.agekey

# 5. Force reconciliation
flux reconcile kustomization apps -n flux-system --with-source --force
```

After that:

- ingress-nginx, podinfo, ingress, secrets, etc. will all be restored from Git.

---

# ✅ Summary

This project demonstrates:

- GitOps with Flux and Kustomize  
- Application deployment using HelmRelease  
- Ingress traffic management with nginx and admission webhooks  
- Secret management through SOPS + age (encrypted at rest in Git)  
- Complete cluster rebuild from source-controlled configuration  

Use this as a template for:

- Personal GitOps labs  
- Interview demos  
- Internal company POCs  
- A foundation for multi-environment setups (dev/stage/prod)  
