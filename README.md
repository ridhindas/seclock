# 🔒 Seclock: Legally-Aware Digital Inheritance & Emergency Access Vault

![Python](https://img.shields.io/badge/Python-3.12+-blue.svg)
![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688.svg)
![Docker](https://img.shields.io/badge/Docker-Enabled-2496ED.svg)
![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20ECR-FF9900.svg)
![CI/CD](https://img.shields.io/badge/CI/CD-Jenkins%20%7C%20DevSecOps-2B4F71.svg)

**Seclock** is a highly secure, production-ready backend system designed to manage digital inheritance and emergency access to sensitive assets. It ensures that digital assets (financial credentials, legal documents, memory vaults) are only decrypted and released when strict, multi-factor cryptographic and legal conditions are met.

---

## 🌟 Key Features

- **Shamir’s Secret Sharing (k-of-n)**: The master decryption key is split into `n` shares. A configurable threshold `k` (e.g., 3-of-5) is required to reconstruct the key, preventing single-point-of-failure or rogue nominee access.
- **Multi-Channel Liveness Checks**: Automated check-ins via App, SMS, and IVR. Includes OCR-based verification of Indian "Jeevan Pramaan" (Digital Life Certificates) to prove the owner is alive.
- **Tamper-Evident Audit Ledger**: Every action (setup, check-in, share submission, veto) is cryptographically hashed (SHA-256) into an immutable, append-only ledger.
- **Emergency Owner Veto**: The vault owner can instantly halt any ongoing decryption claim and reset the liveness timer if a false alarm occurs.
- **Tiered Asset Disclosure**: Assets are categorized into Tier 1 (Financial), Tier 2 (Credentials), and Tier 3 (Memory), allowing staged disclosure based on claim validation.

---

## 🏗️ DevSecOps CI/CD Architecture

This project features a fully automated, enterprise-grade DevSecOps pipeline built on AWS. It enforces "Shift-Left" security, ensuring no vulnerable code or container reaches production.

```mermaid
graph TD
    A[Developer] -->|git push| B(GitHub Repository)
    B -->|Webhook| C[Jenkins Pipeline]
    
    subgraph DevSecOps Stages
    C --> D[1. Gitleaks: Secret Scanning]
    D --> E[2. SAST/SCA: Bandit & pip-audit]
    E --> F[3. E2E Testing: pytest]
    F --> G[4. Docker Build: Multi-stage, Non-root]
    G --> H[5. Trivy: Container Vulnerability Scan]
    end
    
    H -->|Pass| I[Amazon ECR]
    H -->|Fail| J[Pipeline Aborted]
    
    I --> K[6. EKS Deployment via IAM OIDC]
    K --> L[Amazon EKS Cluster]
    L --> M[AWS Network Load Balancer]
    M --> N[End User / Nominee]
