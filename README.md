# Secure_CI-CD_Pipeline_RBAC
Secure my CI/CD Pipeline to my digital portfolio with Role-Based Access Controls

## Overview
This project demonstrates how to build a **Secure CI/CD pipeline** using **GitHub Actions** and **AWS CodePipeline** with **Role-Based Access Controls (RBAC)**. The pipeline enforces **least privilege IAM roles**, integrates **security scanning**, and uses **credential-less deployments** for enhanced security.

---

## Features
- **IAM Role Segmentation:** Separate roles for Build, Test, and Deploy stages.
- **Credential-less Authentication:** GitHub OIDC integration with AWS.
- **Security Scanning:** Automated vulnerability detection using Trivy.
- **Logging & Alerts:** CloudTrail for IAM activity and SNS notifications for pipeline failures.

---

## Architecture


---

## Tech Stack
- **AWS Services:** IAM, CodePipeline, Secrets Manager, CloudTrail, SNS
- **CI/CD:** GitHub Actions
- **Security Tools:** Trivy
- **Languages:** Python, YAML


