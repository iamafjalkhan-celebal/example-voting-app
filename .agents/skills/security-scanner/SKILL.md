---
name: security-scanner
description: Integrate security scanning into GitHub Actions — SAST (CodeQL, Semgrep), dependency auditing, secret detection, container scanning, SBOM generation, and license compliance. Non-blocking in PRs, blocking on main.
---

# Security Scanner

Expert at **integrating security scanning into GitHub Actions build pipelines**. Covers static analysis (SAST), dependency vulnerability scanning, secret detection, container image scanning, SBOM generation, and license compliance.

## When to Use This Skill

- User wants to add security scanning to CI
- Pipeline needs dependency vulnerability checks
- User asks about CodeQL, Semgrep, Trivy, or Snyk
- User wants secret detection in CI
- User needs SBOM generation or license compliance

---

## Scanning Strategy

### What to Scan and When

| Scan Type | Trigger | Blocking? | Tool |
|-----------|---------|-----------|------|
| Dependency audit | Every PR + push | High/critical = block | npm audit, pip-audit, govulncheck |
| SAST (static analysis) | Every PR + scheduled | Configurable | CodeQL, Semgrep |
| Secret detection | Every PR | Always block | gitleaks |
| Container scan | After docker build | High/critical = block | Trivy, Grype |
| License compliance | On dependency change | Configurable | license-checker, FOSSA |
| SBOM generation | On main branch build | Never block | Syft, CycloneDX |

### Severity Strategy

```
Critical → Block immediately, fail the build
High     → Block on main, warn on PRs
Medium   → Warn (informational annotation)
Low      → Ignore in CI (track in dashboard)
```

---

## Dependency Vulnerability Scanning

### Go — govulncheck

```yaml
jobs:
  security:
    name: Security — Dependencies
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Scan for vulnerabilities
        run: govulncheck ./...
```

### Node.js — npm audit

```yaml
- name: Audit dependencies
  run: |
    # --audit-level controls what severity fails the build
    npm audit --audit-level=high --omit=dev
```

For more control, use `npm-audit-action`:

```yaml
- name: Audit dependencies
  uses: oke-py/npm-audit-html@v2
  with:
    output-path: audit-report.html
    audit-level: high
```

### Python — pip-audit

```yaml
- name: Install pip-audit
  run: pip install pip-audit

- name: Audit dependencies
  run: |
    pip-audit \
      -r requirements.txt \
      --severity high \
      --format json \
      --output audit.json
    
    # Pretty summary
    pip-audit -r requirements.txt
```

### Java — OWASP Dependency Check

```yaml
- name: OWASP Dependency Check
  run: ./mvnw org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7

- name: Upload report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: dependency-check-report
    path: target/dependency-check-report.html
    retention-days: 14
```

### Rust — cargo audit

```yaml
- name: Audit dependencies
  uses: rustsec/audit-check@v2
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

---

## SAST — Static Application Security Testing

### CodeQL (GitHub Native)

CodeQL is GitHub's built-in SAST tool. It runs as a **separate workflow** and integrates with the Security tab.

```yaml
# .github/workflows/codeql.yml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    # Run weekly to catch new vulnerability patterns
    - cron: '0 6 * * 1'

permissions:
  security-events: write
  contents: read

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    timeout-minutes: 30

    strategy:
      fail-fast: false
      matrix:
        language: [go]  # or: javascript, python, java, csharp, cpp, ruby, swift

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          # Use extended queries for more thorough analysis
          queries: +security-extended

      # For compiled languages, CodeQL needs to observe the build
      - name: Build
        uses: github/codeql-action/autobuild@v3

      - name: Perform Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: /language:${{ matrix.language }}
```

**CodeQL language support**:

| Language | Auto-build? | Notes |
|----------|------------|-------|
| JavaScript/TypeScript | Yes | No build needed |
| Python | Yes | No build needed |
| Go | Yes | Observes `go build` |
| Java/Kotlin | Yes | Observes Maven/Gradle |
| C/C++ | No | Needs manual build step |
| C# | Yes | Observes `dotnet build` |
| Ruby | Yes | No build needed |

### Semgrep

Semgrep is a fast, flexible SAST tool with community rules:

```yaml
jobs:
  semgrep:
    name: Semgrep SAST
    runs-on: ubuntu-latest
    timeout-minutes: 15
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        run: semgrep scan --config=auto --sarif --output=semgrep.sarif .

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif
```

---

## Secret Detection

### gitleaks

```yaml
jobs:
  secrets:
    name: Secret Detection
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for scanning all commits

      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**gitleaks configuration** (`.gitleaks.toml`):
```toml
[allowlist]
  description = "Allowlisted patterns"
  paths = [
    '''(.*?)\.test\.(js|ts)$''',
    '''(.*?)fixtures(.*?)''',
  ]

# Block on these patterns
[[rules]]
  id = "aws-access-key"
  description = "AWS Access Key"
  regex = '''AKIA[0-9A-Z]{16}'''

[[rules]]
  id = "private-key"
  description = "Private Key"
  regex = '''-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----'''
```

### truffleHog

```yaml
- name: TruffleHog scan
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.pull_request.base.sha }}
    head: HEAD
```

---

## Container Image Scanning

Scan Docker images **after build, before push**.

### Trivy

```yaml
jobs:
  scan-image:
    name: Container Scan
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1  # Fail on critical/high

      - name: Upload Trivy SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

### Grype

```yaml
- name: Scan with Grype
  uses: anchore/scan-action@v4
  id: scan
  with:
    image: myapp:${{ github.sha }}
    fail-build: true
    severity-cutoff: high

- name: Upload SARIF
  if: always()
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: ${{ steps.scan.outputs.sarif }}
```

---

## SBOM Generation

Generate Software Bill of Materials on main branch builds:

### Syft

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json
    artifact-name: sbom

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.spdx.json
    retention-days: 90
```

### CycloneDX (for non-Docker projects)

```yaml
# Node.js
- run: npx @cyclonedx/cyclonedx-npm --output-file sbom.json

# Python
- run: pip install cyclonedx-bom && cyclonedx-py -r -o sbom.json

# Go
- run: |
    go install github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@latest
    cyclonedx-gomod mod -json -output sbom.json
```

---

## License Compliance

### license-checker (Node.js)

```yaml
- name: Check licenses
  run: |
    npx license-checker --production --onlyAllow \
      'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD;CC0-1.0' \
      --excludePrivatePackages
```

### go-licenses (Go)

```yaml
- name: Check licenses
  run: |
    go install github.com/google/go-licenses@latest
    go-licenses check ./... --allowed_licenses=Apache-2.0,MIT,BSD-2-Clause,BSD-3-Clause,ISC
```

---

## SARIF Integration

SARIF (Static Analysis Results Interchange Format) is GitHub's standard for security findings. Upload SARIF to get:
- Results in the **Security tab**
- Inline **PR annotations**
- **Alert tracking** across runs

```yaml
# Most security tools can output SARIF
# Upload with:
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: results.sarif
```

**Required permissions**:
```yaml
permissions:
  security-events: write  # Required for SARIF upload
  contents: read
```

---

## Workflow Template: Comprehensive Security Pipeline

```yaml
# .github/workflows/security.yml
name: Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly scheduled scan

permissions:
  contents: read
  security-events: write

jobs:
  # Fast — run on every PR
  dependency-audit:
    name: Dependency Audit
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

  # Fast — run on every PR
  secret-detection:
    name: Secret Detection
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Slower — run on PRs + scheduled
  codeql:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: go
          queries: +security-extended
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3

  # Image scan — only when Dockerfile changes
  container-scan:
    name: Container Scan
    runs-on: ubuntu-latest
    timeout-minutes: 15
    if: github.event_name != 'schedule'
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t app:scan .
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:scan
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: 1
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy.sarif

  # SBOM — only on main branch
  sbom:
    name: Generate SBOM
    runs-on: ubuntu-latest
    timeout-minutes: 10
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - run: |
          go install github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@latest
          cyclonedx-gomod mod -json -output sbom.json
      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.json
          retention-days: 90
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Skipping dependency audit to save CI time | Vulnerabilities ship to production | It takes <1 min — always run it |
| Blocking on ALL severity levels | Too noisy, developers ignore it | Block on critical/high, warn on medium |
| No scheduled scans | New CVEs found after code was written | Run weekly CodeQL scans |
| Secret detection without `fetch-depth: 0` | Only scans latest commit, misses history | Full clone for complete scanning |
| `continue-on-error: true` on security jobs | Vulnerabilities pass silently | Fail the build on critical findings |
| Running security scans after deployment | Too late to prevent exposure | Scan before build artifacts are used |
| No SARIF upload | Findings buried in logs | Upload SARIF for GitHub Security integration |
| Ignoring medium-severity findings forever | They accumulate and some escalate | Track in dashboard, triage periodically |

---

## Related Skills

When a task involves more than security scanning, load these:

- **Building Docker images to scan?** → `skill({ name: "docker-builder" })` for optimized Dockerfiles
- **Adding code quality checks alongside security?** → `skill({ name: "code-quality-gate" })` for linters and coverage
- **Designing the full pipeline?** → `skill({ name: "devops-pipeline-architect" })` for pipeline structure
- **Security scan failing unexpectedly?** → `skill({ name: "workflow-debugger" })` for diagnosing CI failures
- **Managing releases after security review?** → `skill({ name: "gh-cli" })` for release automation

