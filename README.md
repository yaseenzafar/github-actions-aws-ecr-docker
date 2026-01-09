## 📄 README.md (Ready to Paste)


# AWS ECR Docker Build & Push (OIDC)

This repository contains a CI pipeline that **builds a Docker image and pushes it to Amazon ECR** using **OIDC authentication** (no long-lived AWS credentials).

The workflow is implemented using **Gitea Actions**, but it can be easily adapted for **GitHub Actions** with minimal changes.

---

## 🚀 Features

- ✅ Secure AWS authentication using **OIDC (OpenID Connect)**
- ✅ No AWS access keys stored in secrets
- ✅ Uses **Docker Buildx** for efficient builds
- ✅ Automatically builds and pushes images on every push to `main`
- ✅ Works with **Gitea Actions** and **GitHub Actions**

---

## 🧱 Architecture Overview

```text
Git Push (main)
   ↓
Gitea / GitHub Actions Runner
   ↓
Assume AWS IAM Role via OIDC
   ↓
Authenticate to Amazon ECR
   ↓
Build Docker Image
   ↓
Push Image to ECR
````

---

## 📁 Repository Structure

```text
.
├── Dockerfile
├── .gitea/
│   └── workflows/
│       └── build-and-push.yml
└── README.md
```

> For GitHub Actions, move the workflow to:
> `.github/workflows/build-and-push.yml`

---

## 🔐 Prerequisites

### 1. AWS Requirements

* An **Amazon ECR repository**
* An **IAM Role** with:

  * `AmazonEC2ContainerRegistryPowerUser` (or equivalent)
  * Trust policy allowing OIDC from Gitea or GitHub

### 2. OIDC Trust Policy (Example)

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER_URL>"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringLike": {
      "<OIDC_PROVIDER_URL>:sub": "repo:*"
    }
  }
}
```

---

## 🔑 Required Secrets

### Gitea Secrets

| Secret Name          | Description                   |
| -------------------- | ----------------------------- |
| `AWS_ROLE_TO_ASSUME` | IAM Role ARN to assume        |
| `AWS_REGION`         | AWS region (e.g. `us-east-1`) |

### GitHub Secrets (same names)

| Secret Name          | Description  |
| -------------------- | ------------ |
| `AWS_ROLE_TO_ASSUME` | IAM Role ARN |
| `AWS_REGION`         | AWS region   |

---

## ⚙️ Workflow Explanation

### Trigger

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

* Runs automatically on push to `main`
* Can also be triggered manually

---

### Permissions (OIDC)

```yaml
permissions:
  id-token: write
  contents: read
```

Required for AWS OIDC authentication.

---

### AWS Authentication

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    aws-region: ${{ secrets.AWS_REGION }}
```

This step:

* Requests an OIDC token
* Assumes the IAM role
* Exports temporary AWS credentials

---

### ECR Login

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2
  id: login-ecr
```

* Automatically authenticates Docker with ECR
* Exposes the registry URL as:

  ```
  steps.login-ecr.outputs.registry
  ```

---

### Build & Push Image

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ${{ steps.login-ecr.outputs.registry }}/github-ecr-repo:${{ gitea.run_number }}
```

* Builds Docker image from `Dockerfile`
* Pushes it to Amazon ECR
* Tags the image using the CI run number

---

## 🔄 Gitea vs GitHub Differences

| Gitea               | GitHub               |
| ------------------- | -------------------- |
| `gitea.run_number`  | `github.run_number`  |
| `.gitea/workflows/` | `.github/workflows/` |
| Gitea OIDC Provider | GitHub OIDC Provider |

### GitHub Tag Example

```yaml
tags: |
  ${{ steps.login-ecr.outputs.registry }}/github-ecr-repo:${{ github.run_number }}
```

---

## 🧪 Example ECR Image Output

```text
416818527652.dkr.ecr.us-east-1.amazonaws.com/github-ecr-repo:42
```


