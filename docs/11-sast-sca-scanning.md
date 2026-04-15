# SAST, SCA & Secrets Scanning in GitLab CI

In this guide you'll add three types of security scanning to your existing GitLab CI pipeline. These run automatically on every push and block deployments when critical issues are found.

## What Are We Scanning For?

```mermaid
graph TD
    A[Your Code Push] --> B{Security Scan}

    B --> C[SAST<br/>Static Application Security Testing]
    B --> D[SCA<br/>Software Composition Analysis]
    B --> E[Secrets Detection]
    B --> F[Container Image Scan]

    C --> C1[Finds vulnerabilities<br/>in your code logic<br/>Tool: Semgrep]
    D --> D1[Finds vulnerable<br/>dependencies/packages<br/>Tool: Trivy]
    E --> E1[Finds hardcoded<br/>passwords, API keys, tokens<br/>Tool: Gitleaks]
    F --> F1[Finds CVEs in<br/>base images and packages<br/>Tool: Trivy]

    style C fill:#fcc419,color:#000
    style D fill:#ff922b,color:#fff
    style E fill:#ff6b6b,color:#fff
    style F fill:#e64980,color:#fff
```

## Tools We'll Use

| Tool | What It Does | When It Runs |
|---|---|---|
| **Gitleaks** | Scans for hardcoded secrets (API keys, passwords, tokens) | Every push |
| **Semgrep** | Static code analysis — finds security bugs in your code | Every push |
| **Trivy** | Scans dependencies (SCA) AND container images for CVEs | Build stage |

All three are free, open-source, and have native GitLab CI support.

## Step 1: Add a Security Stage to Your Pipeline

Open your `.gitlab-ci.yml` and add a `security` stage before `build`:

```yaml
stages:
  - security    # ADD THIS
  - test
  - build
  - deploy
```

## Step 2: Add Gitleaks — Secrets Detection

This job fails immediately if it finds any hardcoded credentials. It runs on every push.

```yaml
secrets-scan:
  stage: security
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source . --verbose --redact
  allow_failure: false   # BLOCKS the pipeline on findings
```

**What Gitleaks catches:**
- AWS access keys and secret keys
- Database passwords in code
- API tokens committed by accident
- Private SSH keys or certificates
- `.env` files accidentally committed

> **Tip:** If Gitleaks flags a false positive (e.g., a test fixture), add a `.gitleaksignore` file to suppress specific findings.

## Step 3: Add Semgrep — SAST

Semgrep scans your source code for security vulnerabilities without running it.

```yaml
sast-scan:
  stage: security
  image: returntocorp/semgrep:latest
  script:
    - semgrep scan
        --config "p/owasp-top-ten"
        --config "p/nodejs"
        --config "p/react"
        --error
        --json
        --output semgrep-results.json
        .
  artifacts:
    when: always
    paths:
      - semgrep-results.json
    reports:
      sast: semgrep-results.json
    expire_in: 1 week
  allow_failure: false   # BLOCKS on critical findings
```

**What Semgrep catches:**
- SQL injection vulnerabilities
- Cross-site scripting (XSS)
- Insecure use of `eval()`
- Hardcoded secrets (second layer after Gitleaks)
- Prototype pollution in Node.js
- Dangerous React patterns

## Step 4: Add Trivy — Dependency and Image Scanning

Trivy does two jobs: scans your `package.json` dependencies (SCA) and scans your built container images.

### 4a. Dependency scan (runs before build)

```yaml
dependency-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy fs
        --exit-code 1
        --severity HIGH,CRITICAL
        --scanners vuln
        --format table
        .
  allow_failure: false
```

### 4b. Image scan (runs after build)

Add this job to your `build` stage, after pushing your image:

```yaml
image-scan:
  stage: build
  image: aquasec/trivy:latest
  needs:
    - build-backend    # run after your image is pushed
    - build-frontend
  script:
    - trivy image
        --exit-code 1
        --severity HIGH,CRITICAL
        --format table
        $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA
    - trivy image
        --exit-code 1
        --severity HIGH,CRITICAL
        --format table
        $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
  allow_failure: false
```

## Full Security Stage Example

Here's how your pipeline stages should look after adding all three scanners:

```yaml
stages:
  - security
  - test
  - build
  - deploy

# --- SECURITY STAGE ---

secrets-scan:
  stage: security
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source . --verbose --redact
  allow_failure: false

sast-scan:
  stage: security
  image: returntocorp/semgrep:latest
  script:
    - semgrep scan
        --config "p/owasp-top-ten"
        --config "p/nodejs"
        --config "p/react"
        --error
        .
  allow_failure: false

dependency-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy fs
        --exit-code 1
        --severity HIGH,CRITICAL
        --scanners vuln
        .
  allow_failure: false

# --- BUILD STAGE (existing jobs + image scan) ---

build-backend:
  stage: build
  # ... your existing build job ...

build-frontend:
  stage: build
  # ... your existing build job ...

image-scan:
  stage: build
  image: aquasec/trivy:latest
  needs: [build-backend, build-frontend]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL
        $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA
    - trivy image --exit-code 1 --severity HIGH,CRITICAL
        $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
  allow_failure: false
```

## Testing Your Scanning Pipeline

### Test Gitleaks (intentionally trigger it)

```bash
# Create a test file with a fake secret — should be caught
echo 'const API_KEY = "AKIAIOSFODNN7EXAMPLE"' > test-secret.js
git add test-secret.js
git commit -m "test secret detection"
git push
# Pipeline should FAIL at secrets-scan job
# Delete the file and push again to clear it
```

### Test Trivy (check for CVEs)

```bash
# Run locally before pushing
docker run --rm aquasec/trivy:latest image node:16-alpine
# Shows CVEs in the base image
# Use node:20-alpine to reduce findings
```

### Test Semgrep (run locally)

```bash
# Install semgrep
pip install semgrep

# Run against your backend
semgrep scan --config "p/nodejs" app/backend/src/
```

## Reading Scan Results

When a scan fails, GitLab shows you the output in the job log:

```
[HIGH] CVE-2023-12345 — lodash 4.17.20
  Fix: upgrade to lodash 4.17.21

[CRITICAL] Hardcoded AWS key found
  File: app/backend/src/config.js:12
  Rule: generic.aws-rule.aws-access-token
```

**How to respond:**
- **CVE in dependency** → update the package version in `package.json`
- **Hardcoded secret** → remove it, rotate the credential, use environment variables
- **SAST finding** → fix the code pattern flagged by Semgrep
- **False positive** → add to `.semgrepignore` or `.gitleaksignore` with a comment explaining why

## Common Issues

### Trivy reports too many LOW findings
```yaml
# Change severity threshold
- trivy image --exit-code 1 --severity CRITICAL .  # Only fail on CRITICAL
```

### Semgrep takes too long
```yaml
# Limit to specific directories
- semgrep scan --config "p/nodejs" app/backend/src/
```

### Gitleaks flags a test fixture
```bash
# Create .gitleaksignore
echo "test/fixtures/sample-data.js" >> .gitleaksignore
```

## Cheat Sheet

```bash
# Gitleaks — run locally
docker run --rm -v $(pwd):/repo zricethezav/gitleaks:latest detect --source /repo

# Semgrep — run locally
semgrep scan --config "p/owasp-top-ten" .

# Trivy — scan filesystem
trivy fs --severity HIGH,CRITICAL .

# Trivy — scan image
trivy image --severity HIGH,CRITICAL myimage:latest
```

## Next Steps

With your pipeline now scanning for vulnerabilities, the next step is making sure secrets never end up in your manifests or environment variables in the first place. Move on to [Guide 12 — Secrets Management](12-secrets-management.md).
