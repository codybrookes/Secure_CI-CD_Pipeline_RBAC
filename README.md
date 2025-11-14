# Secure CI/CD Pipeline with RBAC (GitHub Actions → AWS via OIDC)

## Overview
This repository demonstrates a secure, credential-less CI/CD pipeline that enforces Role-Based Access Control (RBAC) across pipeline stages (Build, Test, Deploy) using GitHub Actions' OIDC integration with AWS IAM. Key features:

- **Three stage-specific IAM roles** with least-privilege permissions:
  - `BuildRole` — fetch source + write artifacts (no deploy).
  - `TestRole` — read artifact + run tests (no deploy).
  - `DeployRole` — deploy only (no IAM changes).
- **GitHub OIDC** (`token.actions.githubusercontent.com`) for short-lived credentials (no stored AWS secrets).
- **Trivy** vulnerability scanning gating pipeline on HIGH/CRITICAL vulnerabilities.
- **Monitoring/Audit** via CloudTrail and SNS alerts.

---

## Architecture (textual)
1. GitHub Actions triggers on push / PR.
2. Each job (build/test/deploy) uses `aws-actions/configure-aws-credentials` to assume its dedicated role via OIDC.
3. Build job produces an artifact (e.g., ECR image or S3 artifact).
4. Test job retrieves artifact and runs unit/integration tests + Trivy scan.
5. Deploy job (only runs if previous succeed) deploys to target (S3/ECS/EKS/Lambda).
6. CloudTrail logs and SNS notify on suspicious or failed pipeline events.

## High Level Arch Chart 
Developer  ->  GitHub Repo (source) 
                └─ .github/workflows/main.yml  (CI pipeline)
                     │
                     ├─ Build job (assume BuildRole via OIDC)
                     │     ├─ fetch source (S3/CodeCommit/Git)
                     │     ├─ build artifact (app/docker)
                     │     ├─ push artifact -> ECR / upload -> S3
                     │     └─ write logs -> CloudWatch
                     │
                     ├─ Test job (assume TestRole via OIDC)
                     │     ├─ download artifact
                     │     ├─ run tests
                     │     └─ run Trivy (fail on HIGH/CRITICAL)
                     │
                     └─ Deploy job (assume DeployRole via OIDC)
                           ├─ deploy artifact -> ECS / Lambda / S3 / EKS
                           └─ post-deploy smoke checks
AWS Side:
  - IAM OIDC Provider (token.actions.githubusercontent.com)
  - IAM Roles: BuildRole, TestRole, DeployRole (trust OIDC with repo/branch conditions)
  - CloudTrail logs (tracks AssumeRoleWithWebIdentity + API calls)
  - SNS Topic (alerts), CloudWatch Event rules / Alarms
  - ECR / S3 / ECS / Lambda resources
---

## Prerequisites
- GitHub repository in `ORG/REPO`.
- AWS account `ACCOUNT_ID`.
- GitHub Actions enabled.
- S3 bucket / ECR repo / other target resources created beforehand.
- IAM permissions to create roles, policies, and OIDC provider.

---

## 1) Create IAM OIDC Provider (trust GitHub)
Use the console or CLI. This allows GitHub Actions to request tokens assumed by IAM roles.

**AWS Console**: IAM → Identity providers → Add provider  
- Provider type: `OpenID Connect`  
- Provider URL: `https://token.actions.githubusercontent.com`  
- Audience: `sts.amazonaws.com`

*(If using CLI or CloudFormation, create the provider with the above URL and thumbprints.)*

---

## 2) Example role trust policy (for each role)
Below is a sample trust policy that restricts access to a single repo and (optionally) to a specific branch/environment. Replace `ORG/REPO` and `ACCOUNT_ID` accordingly.

**Trust policy (assume-role) — tailored to `BuildRole` / `TestRole` / `DeployRole`**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
