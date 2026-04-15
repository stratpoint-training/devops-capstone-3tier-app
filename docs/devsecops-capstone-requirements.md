# DevSecOps Capstone Project — Phase 2

## Overview

This is Phase 2 of your training. You've already built a working DevOps pipeline in Phase 1. Now you will secure it — adding security controls at every stage of the pipeline and across the cluster.

**Prerequisite:** Phase 1 capstone must be complete and working before starting Phase 2.

## What You're Building

You will implement a complete **DevSecOps pipeline** on top of your Phase 1 infrastructure:

```
Developer Push
    │
    ▼
GitLab CI
    ├── Secrets Scan (Gitleaks)
    ├── SAST (Semgrep)
    ├── Dependency Scan (Trivy)
    ├── Build Image
    ├── Image Scan (Trivy)
    ├── Generate SBOM (Syft)
    └── Sign Image (Cosign)
    │
    ▼
Kubernetes Cluster
    ├── Admission Control (Kyverno)
    │   ├── Require non-root
    │   ├── Require resource limits
    │   ├── Disallow latest tag
    │   └── Verify image signatures
    ├── Secrets from Vault (External Secrets Operator)
    └── Runtime Security (Falco → Grafana)
```

## Project Requirements

### 1. Pipeline Security (20%)

Extend your existing GitLab CI pipeline with:

- **Secrets scanning** — Gitleaks runs on every push, blocks on findings
- **SAST** — Semgrep with OWASP Top 10 ruleset, blocks on critical findings
- **Dependency scanning** — Trivy scans `package.json` before build
- **Image scanning** — Trivy scans built images, blocks on HIGH/CRITICAL CVEs
- **Pipeline order** — security stage runs before build stage

Deliverables:
- Updated `.gitlab-ci.yml` with all four scanning jobs
- Screenshot of a pipeline run showing all security stages passing
- Screenshot of a pipeline that failed on a security finding (then fixed)

### 2. Secrets Management (20%)

Replace all hardcoded credentials with External Secrets Operator:

- **ESO installed** in cluster
- **Vault installed** (dev mode for local) with app secrets stored
- **SecretStore** configured and connected to Vault
- **ExternalSecret** created for database credentials
- **Helm chart updated** — no credentials in `values.yaml`, references the managed secret instead
- No secrets in Git history (verified with Gitleaks)

Deliverables:
- `secretstore.yaml` and `externalsecret-db.yaml` committed to repo
- Screenshot of `kubectl describe externalsecret` showing `SecretSynced`
- Screenshot of backend pod running with secrets injected from ESO
- Updated `values.yaml` showing no hardcoded credentials

### 3. Manifest Security (20%)

Scan all Kubernetes manifests and Helm charts with Checkov:

- **Local scan** — run Checkov against your Helm chart, fix all HIGH/CRITICAL findings
- **CI integration** — Checkov runs as a pipeline stage after build, blocks on HIGH/CRITICAL
- **Fixes implemented** in Helm chart templates:
  - Non-root security context on all containers
  - CPU and memory limits on all containers
  - No `latest` image tags
  - Secrets from SecretKeyRef (not plain env vars)
  - `allowPrivilegeEscalation: false`

Deliverables:
- Checkov scan output showing 0 HIGH/CRITICAL findings
- Updated Helm chart templates with security context applied
- `manifest-scan` job in `.gitlab-ci.yml`

### 4. Admission Control (20%)

Install Kyverno and enforce security policies cluster-wide:

- **Kyverno installed** and running
- **Minimum 4 policies** implemented and enforced:
  1. Require non-root containers
  2. Require CPU and memory limits
  3. Disallow `latest` image tag
  4. Verify image signatures (Cosign)
- **ArgoCD still syncs** successfully after policies are applied
- **Audit report** shows 0 violations for your application

Deliverables:
- All policy YAML files committed to `policies/kyverno/` directory
- Screenshot of `kubectl get clusterpolicy` showing all policies READY
- Screenshot of a `kubectl run` attempt being blocked by a policy
- Screenshot of ArgoCD sync succeeding with policies active

### 5. Supply Chain Security + Runtime Security (20%)

Implement SBOM generation, image signing, and runtime monitoring:

**Supply Chain (Syft + Cosign):**
- SBOM generated for every image build (CycloneDX format)
- Images signed with Cosign keyless signing in CI
- SBOMs attached to images in the registry
- Kyverno policy verifies image signatures before deployment

**Runtime Security (Falco):**
- Falco installed and running
- At least 2 custom rules written for your application
- Falco alerts forwarded to Loki (from Phase 1 observability stack)
- Grafana dashboard showing security events
- Alert rule configured for CRITICAL events

Deliverables:
- `generate-sbom` and `sign-images` jobs in `.gitlab-ci.yml`
- SBOM artifact (`.json` file) from a recent pipeline run
- `policy-verify-image-signature.yaml` Kyverno policy
- `custom-rules.yaml` Falco rules file
- Screenshot of Grafana security dashboard showing Falco events
- Screenshot of Falco detecting a shell spawn in a container

---

## Two-Week Timeline

### Week 1: Pipeline and Cluster Security

**Days 1–2: Pipeline Security**
- Add Gitleaks, Semgrep, Trivy to GitLab CI
- Test each scanner — intentionally trigger a finding, then fix it
- Verify pipeline blocks on HIGH/CRITICAL findings

**Days 3–4: Secrets Management**
- Install ESO and Vault
- Migrate database credentials from `values.yaml` to ExternalSecret
- Verify app still works end-to-end after migration

**Day 5: Manifest Security**
- Run Checkov locally, fix all HIGH/CRITICAL findings in Helm chart
- Add Checkov to GitLab CI
- Commit and verify pipeline passes

### Week 2: Advanced Controls and Runtime

**Days 1–2: Admission Control**
- Install Kyverno
- Apply all four policies in Audit mode first
- Fix any violations in your Helm chart
- Switch to Enforce mode
- Verify ArgoCD sync still works

**Days 3–4: Supply Chain + Runtime Security**
- Add Syft SBOM generation to CI
- Add Cosign image signing to CI
- Add Kyverno signature verification policy
- Install Falco, write custom rules
- Connect Falco to Loki, build Grafana dashboard

**Day 5: End-to-End Testing and Presentation**
- Full pipeline run with all security stages
- Simulate an attack (shell in container) — verify Falco detects it
- Final documentation
- Presentation prep

---

## Evaluation Criteria

### Basic Implementation (70–79%)
- Pipeline has at least 2 security scanning jobs
- ESO installed and at least one secret managed externally
- Checkov run locally with some findings fixed
- Kyverno installed with at least 2 policies
- Falco installed and default rules running

### Proficient Implementation (80–89%)
- All 4 scanning jobs in CI (secrets, SAST, dependencies, images)
- All database credentials managed via ESO
- Checkov in CI pipeline, 0 HIGH/CRITICAL findings in Helm chart
- All 4 Kyverno policies active and enforced
- Images signed with Cosign, signature verification policy active
- Falco alerts visible in Grafana

### Advanced Implementation (90–100%)
- All of the above, plus:
- Pipeline fails on a real finding — fix documented with before/after
- Custom Falco rules for your specific application
- SBOM attached to images in registry
- Grafana security dashboard with alert rules
- End-to-end demo: commit → scan → build → sign → deploy → detect
- Comprehensive documentation of each security control and why it matters

---

## Optional Bonus Points

1. **DAST** (+5%) — Add OWASP ZAP scan against your running staging environment
2. **Network Policies** (+5%) — Implement Kubernetes NetworkPolicy to restrict pod-to-pod traffic
3. **CIS Benchmark** (+5%) — Run kube-bench against your cluster and remediate findings

---

## Deliverables

### GitLab Repository
- Updated `.gitlab-ci.yml` with all security stages
- `policies/kyverno/` directory with all Kyverno policies
- `policies/falco/custom-rules.yaml` with custom Falco rules
- `secretstore.yaml` and `externalsecret-db.yaml` manifests
- Updated Helm chart with security contexts applied
- No secrets in Git history

### Documentation
- Security architecture diagram showing all controls and where they run
- Threat model: what each control protects against
- Incident response runbook: what to do when Falco fires a CRITICAL alert

### Presentation (30 minutes)
- **Technical demo (20 min):**
  - Live pipeline run with all security stages
  - Show a scan finding being caught and fixed
  - Deploy app — show Kyverno enforcing policies
  - Trigger a Falco alert — show it in Grafana
- **Q&A (10 min):**
  - Be prepared to explain what each tool does and why it's in the pipeline
  - Know the difference between P0 and P1 controls
  - Know what happens if each control is removed

---

## Reference Documentation

- [External Secrets Operator](https://external-secrets.io/latest/)
- [Kyverno](https://kyverno.io/docs/)
- [Falco](https://falco.org/docs/)
- [Cosign](https://docs.sigstore.dev/cosign/overview/)
- [Syft](https://github.com/anchore/syft)
- [Checkov](https://www.checkov.io/1.Welcome/Quick%20Start.html)
- [Semgrep Rules](https://semgrep.dev/r)
- [Trivy](https://trivy.dev)
- [Gitleaks](https://github.com/gitleaks/gitleaks)

---

Remember:
- Phase 2 builds on Phase 1 — your cluster and pipelines must be working first
- Security controls layer on top of each other — implement in order
- A failed scan is success — it means you caught something
- Document why each control exists, not just how to install it
