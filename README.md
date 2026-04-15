# DevSecOps Capstone Project: 3-Tier Task Management Application — Phase 2

This is the **Phase 2 — DevSecOps** branch. It builds on Phase 1 by adding security at every stage of the pipeline and cluster.

> **Prerequisite:** Complete Phase 1 (main branch) before starting here.

## What You're Securing

The same 3-tier task management app from Phase 1:
- **Frontend**: React UI
- **Backend**: Node.js/Express REST API
- **Database**: PostgreSQL

Your Phase 1 pipeline (Kubernetes, Helm, GitLab CI, ArgoCD, Observability) is the foundation. Phase 2 adds security on top of it.

## Quick Start

```bash
# Run the app locally
cd app
docker compose up
```

Then follow the guides in order starting from [docs/10-devsecops-intro.md](./docs/10-devsecops-intro.md).

## What You'll Implement

```
Developer Push
    │
    ▼
GitLab CI
    ├── Gitleaks       — secrets detection
    ├── Semgrep        — SAST (OWASP Top 10)
    ├── Trivy          — dependency + image scan
    ├── Build Image
    ├── Syft           — generate SBOM
    └── Cosign         — sign image
    │
    ▼
Kubernetes Cluster
    ├── Kyverno        — admission control policies
    ├── External Secrets Operator — secrets from Vault
    └── Falco          — runtime threat detection → Grafana
```

## Project Structure

```
.
├── app/                          # Application source code (provided, do not modify)
│   ├── frontend/
│   ├── backend/
│   └── database/
│
├── docs/                         # Phase 2 implementation guides
│   ├── 10-devsecops-intro.md     # Start here
│   ├── 11-sast-sca-scanning.md   # Gitleaks · Semgrep · Trivy
│   ├── 12-secrets-management.md  # ESO + Vault
│   ├── 13-manifest-security.md   # Checkov
│   ├── 14-admission-control.md   # Kyverno
│   ├── 15-supply-chain-security.md # Syft + Cosign
│   ├── 16-runtime-security.md    # Falco
│   └── devsecops-capstone-requirements.md
│
├── devsecops-training.html       # Phase 2 training landing page
└── .gitlab-ci.yml                # Your pipeline (you will add security stages)
```

## Implementation Requirements

### 1. Pipeline Security (20%)
- Gitleaks, Semgrep, Trivy scanning in GitLab CI
- Pipeline blocks on HIGH/CRITICAL findings

### 2. Secrets Management (20%)
- External Secrets Operator + HashiCorp Vault
- No credentials in Git or values.yaml

### 3. Manifest Security (20%)
- Checkov scanning Helm charts and K8s manifests
- All HIGH/CRITICAL findings fixed

### 4. Admission Control (20%)
- Kyverno with 4 core policies enforced
- Signed image verification at admission

### 5. Supply Chain + Runtime Security (20%)
- Syft SBOM + Cosign image signing in CI
- Falco runtime detection with Grafana dashboard

## Timeline

### Week 1: Pipeline & Cluster Security
- Days 1–2: SAST, SCA, secrets scanning in GitLab CI
- Days 3–4: External Secrets Operator + Vault
- Day 5: Checkov manifest scanning

### Week 2: Advanced Controls & Runtime
- Days 1–2: Kyverno admission control
- Days 3–4: Syft + Cosign + Falco
- Day 5: End-to-end testing and presentation

## Evaluation Criteria

### Basic (70–79%)
- At least 2 scanning jobs in CI
- ESO installed with at least 1 secret managed
- Kyverno installed with at least 2 policies
- Falco running with default rules

### Proficient (80–89%)
- All 4 scanning jobs in CI
- All DB credentials via ESO
- Checkov in CI, 0 HIGH/CRITICAL in Helm chart
- All 4 Kyverno policies enforced
- Falco alerts visible in Grafana

### Advanced (90–100%)
- All of the above plus:
- Images signed + Kyverno signature verification
- Custom Falco rules for your application
- SBOM attached to images in registry
- Grafana security dashboard with alert rules
- Full end-to-end demo

## Documentation

- [Training Guide](./devsecops-training.html) — Open in browser for Phase 2 overview and video resources
- [Implementation Guides](./docs/README.md) — Step-by-step guides 10–16
- [Capstone Requirements](./docs/devsecops-capstone-requirements.md) — Deliverables and evaluation

## Support

1. Check the troubleshooting section in each guide
2. Run `kubectl describe` and check pod logs for errors
3. Ask during lab sessions
