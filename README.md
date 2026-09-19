# Azure AKS + ACR + GitHub Actions CI/CD Pipeline

A containerized app deployed to Azure Kubernetes Service (AKS), pulling from a private Azure Container Registry (ACR), with a fully automated GitHub Actions CI/CD pipeline that builds and redeploys on every push to `main`.

Built as a hands-on learning project to translate existing AWS/EKS DevOps experience to Azure.

---

## Architecture

```
Developer push to GitHub (main branch)
        │
        ▼
GitHub Actions workflow triggers
        │
        ├─► Azure login (service principal via AZURE_CREDENTIALS secret)
        │
        ├─► docker build → tag with commit SHA
        │
        ├─► docker push → Azure Container Registry (ACR)
        │
        ├─► az aks get-credentials → fetch kubeconfig
        │
        └─► kubectl set image → rolling update on AKS
                    │
                    ▼
        AKS Cluster (1 node, Standard_B2as_v2)
                    │
                    ▼
        LoadBalancer Service → Public IP → App reachable on the internet
```

**Resources used** (all inside a single resource group for easy cleanup):
- Resource Group: `rg-aks-practice`
- Azure Container Registry: `acrpracticedevang` (Basic SKU)
- AKS Cluster: `aks-practice` (1 node, free control-plane tier, `Standard_B2as_v2`)
- Service Principal: `gh-actions-aks-practice` (scoped to the resource group only, Contributor role)

---

## Prerequisites

- Azure CLI (`az`) installed and logged in (`az login`)
- Docker installed and running locally
- `kubectl` installed
- A GitHub repository for the project
- An Azure free account/subscription

---

## Build Steps (in order, with every command used)

### 1. Resource group + CLI defaults

```bash
az group create --name rg-aks-practice --location centralindia
az configure --defaults group=rg-aks-practice location=centralindia
```

### 2. Register required resource providers

New Azure subscriptions don't come with every provider pre-registered — this has no AWS equivalent and will silently block later steps if skipped.

```bash
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute

# check status before moving on:
az provider show --namespace Microsoft.ContainerRegistry --query registrationState -o tsv
```

### 3. Create ACR

```bash
az acr create --name acrpracticedevang --sku Basic
```

### 4. Build and push the image

> **Note:** `az acr build` (ACR Tasks / remote build) is blocked on free-trial-credit subscriptions with a `TasksOperationsNotAllowed` error. Build locally with Docker instead.

```bash
az acr login --name acrpracticedevang
docker build -t acrpracticedevang.azurecr.io/demoapp:v1 .
docker push acrpracticedevang.azurecr.io/demoapp:v1

# verify it landed:
az acr repository list --name acrpracticedevang -o table
az acr repository show-tags --name acrpracticedevang --repository demoapp -o table
```

### 5. Create AKS and attach ACR

> **Note:** Free-trial subscriptions restrict which VM sizes are available per region. `Standard_B2s` failed; `Standard_B2as_v2` worked. Run `az vm list-sizes --location <region> -o table` if you hit the same wall.

```bash
az aks create \
  --name aks-practice \
  --node-count 1 \
  --tier free \
  --node-vm-size Standard_B2as_v2 \
  --generate-ssh-keys \
  --attach-acr acrpracticedevang
```

Attaching ACR at creation time sets up pull permissions automatically — no IAM-role/IRSA-equivalent dance needed, unlike EKS + ECR.

### 6. Connect kubectl

```bash
az aks get-credentials --name aks-practice --overwrite-existing
kubectl get nodes
```

`--overwrite-existing` is needed any time you're recreating a cluster with a name that already exists in your local kubeconfig (e.g. after a teardown/rebuild).

### 7. Kubernetes manifests

**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: acrpracticedevang.azurecr.io/demoapp:v1
          ports:
            - containerPort: 80
```

**`service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demoapp-svc
spec:
  type: LoadBalancer
  selector:
    app: demoapp
  ports:
    - port: 80
      targetPort: 80
```

### 8. Deploy

```bash
kubectl apply -f .
kubectl get pods -w          # wait for Running
kubectl get svc demoapp-svc  # wait for EXTERNAL-IP to populate (1-2 min)
```

---

## CI/CD Pipeline (GitHub Actions)

### Service principal setup

```bash
# On Windows Git Bash, leading "/subscriptions/..." paths get mangled into
# Windows filesystem paths — prefix with MSYS_NO_PATHCONV=1 to avoid this.
MSYS_NO_PATHCONV=1 az ad sp create-for-rbac \
  --name gh-actions-aks-practice \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/rg-aks-practice \
  --sdk-auth
```

> Security note: never paste the credential output anywhere outside a GitHub secret field (not chat logs, not commits, not Slack). If it's ever exposed, rotate immediately: `az ad sp credential reset --id <client-id>`.

### GitHub repository secrets

| Secret | Value |
|---|---|
| `AZURE_CREDENTIALS` | Full JSON output from `create-for-rbac --sdk-auth` |
| `ACR_NAME` | `acrpracticedevang` |
| `AKS_CLUSTER` | `aks-practice` |
| `RESOURCE_GROUP` | `rg-aks-practice` |

### `.github/workflows/deploy.yml`

```yaml
name: Build and Deploy to AKS

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Build and push image to ACR
        run: |
          az acr login --name ${{ secrets.ACR_NAME }}
          docker build -t ${{ secrets.ACR_NAME }}.azurecr.io/demoapp:${{ github.sha }} .
          docker push ${{ secrets.ACR_NAME }}.azurecr.io/demoapp:${{ github.sha }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group ${{ secrets.RESOURCE_GROUP }} \
            --name ${{ secrets.AKS_CLUSTER }} \
            --overwrite-existing

      - name: Deploy to AKS
        run: |
          kubectl set image deployment/demoapp demoapp=${{ secrets.ACR_NAME }}.azurecr.io/demoapp:${{ github.sha }}
          kubectl rollout status deployment/demoapp
```

**Design choices worth noting:**
- Image tagged with `${{ github.sha }}` (the commit hash) rather than a static `v1` — guarantees every deploy uses a unique, traceable tag and lets Kubernetes correctly detect that the image actually changed.
- `kubectl set image` instead of re-applying the full manifest — the standard "just bump the image" pattern for continuous deployment.
- `kubectl rollout status` makes the job fail loudly if the new pod doesn't come up healthy, instead of reporting a false success.

---

## Issues Hit & Fixes (real debugging log)

| Issue | Fix |
|---|---|
| `MissingSubscriptionRegistration` on `az acr create` | Register the `Microsoft.ContainerRegistry` provider (and `ContainerService`/`Network`/`Compute` ahead of AKS) |
| `az acr build` → `TasksOperationsNotAllowed` | ACR Tasks (remote build) is blocked on free-trial subscriptions — build locally with `docker build`/`docker push` instead |
| `az aks create` → VM size `Standard_B2s` not allowed | Free-trial subscriptions restrict available VM sizes per region — used `Standard_B2as_v2` instead |
| `az aks get-credentials` → "different object already exists" | Added `--overwrite-existing` (needed when recreating a cluster with a previously-used name) |
| Git Bash mangling `/subscriptions/...` into a Windows path | Prefixed the command with `MSYS_NO_PATHCONV=1` |
| Credentials pasted into chat during setup | Rotated with `az ad sp credential reset` before continuing |

---

## Cleanup

Everything lives in one resource group, so teardown is one command:

```bash
az group delete --name rg-aks-practice --yes --no-wait

# confirm it's fully gone:
az group exists --name rg-aks-practice

# separately clean up the Azure AD service principal (lives at tenant level, not inside the resource group):
az ad sp list --display-name gh-actions-aks-practice --query "[].appId" -o tsv
az ad sp delete --id <the-id-it-returned>
```

---

## What This Demonstrates

- Provisioning Azure infrastructure via CLI (not portal-click-ops)
- Private container registry workflow (ACR) with managed-identity-based pull auth
- Managed Kubernetes (AKS) — cluster creation, scaling, and app deployment
- End-to-end CI/CD: GitHub Actions building, pushing, and deploying automatically on push
- Real troubleshooting against actual subscription/platform constraints, not a scripted happy path
