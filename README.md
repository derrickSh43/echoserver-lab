# 🚀 GitHub Actions CI/CD to GKE using Workload Identity Federation

This guide walks you through securely deploying to a GKE cluster using GitHub Actions and **Workload Identity Federation (WIF)** without needing to store long-lived service account keys. This is production-grade and reusable.

---

## 📁 Project Structure

```
.
├── .github
│   └── workflows
│       └── deploy-gke.yaml
├── k8s
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml (optional)
```

---

## 🧩 Placeholder Reference

| Placeholder               | Description                                                                                       |
| ------------------------- | ------------------------------------------------------------------------------------------------- |
| `<PROJECT_ID>`            | Your GCP project ID (e.g., `class65projects`)                                                     |
| `<PROJECT_NUMBER>`        | Your GCP project number (e.g., `603509934273`)                                                    |
| `<GITHUB_REPO>`           | Your GitHub repo in `owner/repo` format (e.g., `derrickSh43/echoserver-lab`)                      |
| `<WORKLOAD_POOL_NAME>`    | Name of your WIF pool (e.g., `gke-lab-pool`)                                                      |
| `<WIF_PROVIDER_NAME>`     | Name of your OIDC provider (e.g., `gke-lab-provider`)                                             |
| `<SERVICE_ACCOUNT_EMAIL>` | GCP service account used for deployment (e.g., `gke-lab-sa@<PROJECT_ID>.iam.gserviceaccount.com`) |

---

## 🛠️ 1. Create Workload Identity Pool + Provider

```bash
gcloud iam workload-identity-pools create <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --display-name="GitHub WIF Pool"

# Create OIDC provider
# NOTE: Replace <GITHUB_REPO> with exact casing (case-sensitive)
gcloud iam workload-identity-pools providers create-oidc <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-condition="attribute.repository == '<GITHUB_REPO>'"
```

---

## 🧾 2. Create and Bind a GCP Service Account

```bash
# Create service account
gcloud iam service-accounts create gke-lab-sa \
  --project="<PROJECT_ID>" \
  --display-name="GKE Deploy Service Account"

# Allow GitHub repo to impersonate this account
gcloud iam service-accounts add-iam-policy-binding <SERVICE_ACCOUNT_EMAIL> \
  --project="<PROJECT_ID>" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<WORKLOAD_POOL_NAME>/attribute.repository/<GITHUB_REPO>"
```

---

## 🔐 3. Add GitHub Repository Secrets

Go to `Settings → Secrets and variables → Actions`, and add:

| Name          | Value          |
| ------------- | -------------- |
| `GKE_PROJECT` | `<PROJECT_ID>` |

You can add more as needed (`GKE_CLUSTER`, etc.)

---

## 🤖 4. GitHub Actions Workflow Example (`.github/workflows/deploy-gke.yaml`)

```yaml
name: Deploy Echoserver to GKE
on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GKE_PROJECT }}
  GKE_CLUSTER: gke-central1-cluster
  GKE_ZONE: us-central1-a
  DEPLOYMENT_NAME: echoserver
  NAMESPACE: prod
  PROJECT_NUMBER: <PROJECT_NUMBER>

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<WORKLOAD_POOL_NAME>/providers/<WIF_PROVIDER_NAME>
          service_account: gke-lab-sa@${{ env.PROJECT_ID }}.iam.gserviceaccount.com

      - name: Set up gcloud CLI
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ env.PROJECT_ID }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}
          project_id: ${{ env.PROJECT_ID }}

      - name: Deploy to GKE
        run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
          kubectl rollout status deployment/${{ env.DEPLOYMENT_NAME }} -n ${{ env.NAMESPACE }}
```

---

## 🧹 5. Optional Cleanup

```bash
# Delete provider (if redoing or renaming)
gcloud iam workload-identity-pools providers delete <WIF_PROVIDER_NAME> \
  --project="<PROJECT_ID>" \
  --location="global" \
  --workload-identity-pool="<WORKLOAD_POOL_NAME>"

# Delete WIF pool
gcloud iam workload-identity-pools delete <WORKLOAD_POOL_NAME> \
  --project="<PROJECT_ID>" \
  --location="global"
```

---
