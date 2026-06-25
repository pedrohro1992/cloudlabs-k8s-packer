# TODO — GitHub Actions Pipeline for Packer AMI Build

## Phase 1 — AWS IAM OIDC Setup

### 1.1 Create the OIDC Identity Provider

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

Save the returned `Arn` — you'll need it for the trust policy.

### 1.2 Create the IAM Policy for Packer

Create `packer-build-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeImages",
        "ec2:CreateImage",
        "ec2:RegisterImage",
        "ec2:DeregisterImage",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:StopInstances",
        "ec2:CreateTags",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeAccountAttributes",
        "ec2:DescribeLaunchTemplates"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name PackerBuildPolicy \
  --policy-document file://packer-build-policy.json
```

### 1.3 Create the IAM Role with OIDC Trust

Replace `<YOUR_ACCOUNT_ID>` with your AWS account ID.

Create `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:pedrohro1992/cloudlabs-k8s-packer:*"
        },
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name GitHubActions-PackerBuild \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name GitHubActions-PackerBuild \
  --policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/PackerBuildPolicy
```

### 1.4 Add the Role ARN to GitHub Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Name | Value |
|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::<YOUR_ACCOUNT_ID>:role/GitHubActions-PackerBuild` |

---

## Phase 2 — GitHub Actions Workflow

Create `.github/workflows/build-ami.yml`:

```yaml
name: Build K8s Control Plane AMI

on:
  push:
    branches: [main]
    paths:
      - 'packer/**'
      - 'ansible/**'
  pull_request:
    branches: [main]
    paths:
      - 'packer/**'
      - 'ansible/**'
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup Packer
        uses: hashicorp/setup-packer@main
        with:
          version: "latest"

      - name: Initialize Packer
        run: packer init .
        working-directory: packer

      - name: Validate Packer Template
        run: packer validate .
        working-directory: packer

      - name: Build AMI
        run: packer build .
        working-directory: packer
```

---

## Phase 3 — Push & Verify

1. Commit and push everything to `main`
2. Go to the **Actions** tab in your GitHub repo
3. The workflow will trigger on push (if files under `packer/` or `ansible/` changed)
4. You can also manually trigger it via **workflow_dispatch**
5. Check the build logs — Packer will assume the OIDC role and build the AMI
