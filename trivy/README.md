# Trivy: Container Image Vulnerability Scanning in CI/CD

This folder demonstrates integrating [Trivy](https://github.com/aquasecurity/trivy) into a GitHub Actions pipeline as a security gate, with results surfaced in GitHub's native Security tab.

## What this demonstrates

- Scanning a container image for known CVEs as part of CI
- Failing the build when CRITICAL/HIGH vulnerabilities are found
- Publishing scan results to GitHub Code Scanning (Security tab) via SARIF
- A structured, expiring vulnerability exception mechanism (`.trivyignore.yaml`)

---

## Local setup

```bash
# macOS
brew install trivy

# WSL2 / Ubuntu
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy
```

Sanity check against a known-old public image before touching anything project-specific:

```bash
trivy image nginx:1.16
```

## The test target: `Dockerfile`

An intentionally outdated base image, so the scanner has real CVEs to find:

```dockerfile
FROM python:3.9-slim

COPY app.py /app/app.py
WORKDIR /app
CMD ["python", "app.py"]
```

```bash
docker build -t devops-demo:latest .
trivy image devops-demo:latest
```

---

## CI workflow: `.github/workflows/trivy-scan.yml`

```yaml
name: Trivy Container Scan
on:
  workflow_dispatch:
  push:
    branches: [main]
    paths:
      - 'trivy/**'
  pull_request:
    branches: [main]
    paths:
      - 'trivy/**'
jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t devops-demo:${{ github.sha }} ./trivy
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: devops-demo:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true
          exit-code: '1'
          trivyignores: 'trivy/.trivyignore.yaml'
      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
          category: 'trivy-container-scan'
```

### Design decisions in this workflow

**`paths: ['trivy/**']`**
This repo hosts multiple independent tool demos (ArgoCD, Trivy, etc.). Path filtering keeps each tool's workflow scoped to its own folder so unrelated changes don't trigger unrelated scans.

**`permissions` declared explicitly at job level**
`contents: read` and `security-events: write` are declared rather than relying on the repository's default token permissions. Without this, `upload-sarif` fails with a "Resource not accessible by integration" warning — the job silently runs without the ability to write to the Security tab, because GitHub falls back to whatever the repo's default `GITHUB_TOKEN` permission is.

**`format: sarif` instead of `table`**
SARIF (Static Analysis Results Interchange Format) is what GitHub's Code Scanning API consumes. `table` format is human-readable in logs but not ingestible by the Security tab.

**Single Trivy invocation doing both jobs**
`exit-code` (gate CI) and `format`/`output` (produce a report) are independent parameters — one invocation can both fail the build on CRITICAL/HIGH and produce the SARIF file for upload. An earlier draft of this workflow split this into two separate Trivy calls; that was unnecessary complexity once it became clear the parameters don't conflict.

**`if: always()` on the upload step**
Because `exit-code: '1'` intentionally fails the job when vulnerabilities are found, GitHub Actions would skip all subsequent steps by default. `if: always()` ensures the scan results still get uploaded to the Security tab even when the job fails — which is the point: you want to see *why* it failed, not just that it failed.

**`category` on `upload-sarif`**
Without a `category`, a second SARIF source uploaded later to the same repo would overwrite this one in the Security tab. Explicit categorization keeps multiple scan sources (e.g. a future IaC or dependency scan) visible side by side.

**`ignore-unfixed: true`**
Separates "there's a patch available and you haven't applied it" from "there is currently no fix upstream." Only the former should realistically block CI — otherwise builds get stuck on issues nobody can act on yet.

---

## Vulnerability exceptions: `.trivyignore.yaml`

```yaml
vulnerabilities: []
```

Currently empty by design. When a real exception is needed, entries follow this shape:

```yaml
vulnerabilities:
  - id: CVE-2023-12345
    statement: "Why this is temporarily accepted, e.g. vulnerable code path is not exercised by this service"
    expired_at: 2026-12-31T00:00:00Z
```

### Why `.trivyignore.yaml` over the legacy `.trivyignore` format

The plain-text `.trivyignore` format is just a list of CVE IDs with no context — six months later, nobody remembers why a given CVE was suppressed. The structured YAML format requires a `statement` (the reasoning) and an `expired_at` date, so exceptions self-expire and force periodic re-review rather than accumulating indefinitely.

---

## Trade-offs considered but not implemented here

These are documented rather than built into the demo, since the goal is to show the reasoning, not to over-engineer a portfolio piece:

- **Pinning the Action to a commit SHA instead of a version tag** — mitigates supply-chain risk if a tag is ever moved, at the cost of manual (or Dependabot-assisted) version bumps. Worth it for production pipelines handling sensitive data; unnecessary overhead for a demo repo.
- **Governance on the ignore file** — in a real team, changes to `.trivyignore.yaml` should require review (e.g. via `CODEOWNERS`) so that suppressing a finding isn't a unilateral action.
- **Scheduled re-scans** — the Trivy vulnerability DB updates daily. A `push`-triggered scan only proves an image was clean *at build time*; a daily `schedule: cron` scan of the currently deployed image tag would catch newly disclosed CVEs in an image that hasn't changed.
- **Severity threshold as a gradual rollout** — jumping straight to `MEDIUM,HIGH,CRITICAL` on a codebase with existing technical debt tends to produce a wall of failing builds and encourages teams to bypass the gate rather than fix it. Starting at `CRITICAL` and tightening over time is the more realistic rollout path.
- **Private repo limitation** — GitHub Code Scanning requires GitHub Advanced Security for private repositories. This repo is public specifically so the Security tab integration works without that licensing dependency.