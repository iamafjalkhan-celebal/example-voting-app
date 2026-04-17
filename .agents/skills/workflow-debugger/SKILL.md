---
name: workflow-debugger
description: Diagnose and fix failing GitHub Actions workflows. Covers common failure patterns, YAML syntax pitfalls, permission errors, caching issues, debugging techniques, and action-specific troubleshooting.
---

# Workflow Debugger

Expert at **diagnosing and fixing failing GitHub Actions workflows**. When a pipeline fails, this skill helps identify the root cause and provide a targeted fix.

## When to Use This Skill

- A GitHub Actions workflow is failing
- User sees cryptic error messages in CI
- Workflow behaves differently than expected
- User asks "why is my CI failing?"
- User needs to debug a specific action or step

---

## Debugging Methodology

### Step 1: Identify the Failure Layer

```
Workflow YAML parse error?     → YAML Syntax Issue
Job not starting?              → Trigger / Permissions Issue
Step failing?                  → Runtime Error
Intermittent failures?         → Flaky Test / External Dependency
Slow but not failing?          → Performance Issue (see build-optimizer skill)
```

### Step 2: Read the Error Message

GitHub Actions errors follow patterns. Match the error to the sections below.

### Step 3: Apply the Fix

Each section includes the **exact fix** for the identified issue.

---

## Common Failure Patterns

### 1. Permission Errors

**Error**: `Resource not accessible by integration` or `403 Forbidden`

**Cause**: Missing `permissions:` block. GitHub Actions defaults to **read-only** for `GITHUB_TOKEN` on public repos and forks.

**Fix**: Add explicit permissions:

```yaml
permissions:
  contents: read       # Checkout code
  pull-requests: write # Comment on PRs
  checks: write        # Create check runs
  security-events: write # Upload SARIF
  issues: write        # Create issues
  packages: write      # Push to GHCR
```

**Permission reference**:

| Action | Required Permission |
|--------|-------------------|
| `actions/checkout` | `contents: read` |
| Comment on PR | `pull-requests: write` |
| Upload SARIF (CodeQL) | `security-events: write` |
| Push to GHCR | `packages: write` |
| Create releases | `contents: write` |
| `dorny/test-reporter` | `checks: write` |
| Upload artifacts | None (built-in) |

### 2. Checkout Issues

**Error**: `fatal: could not read Username` or shallow clone issues

**Fixes**:

```yaml
# For private repos / submodules — use PAT token
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.PAT_TOKEN }}
    submodules: recursive

# For actions needing full git history (git log, changelog, gitleaks)
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full clone, not shallow

# For large repos with LFS
- uses: actions/checkout@v4
  with:
    lfs: true
```

### 3. Cache Misses

**Error**: Cache never hits, always `Cache not found`

**Debugging**:

```yaml
- uses: actions/cache@v4
  id: cache
  with:
    path: ~/.cache/something
    key: ${{ runner.os }}-tool-${{ hashFiles('**/lockfile') }}
    restore-keys: |
      ${{ runner.os }}-tool-

# Add diagnostic step
- name: Cache diagnostic
  run: |
    echo "Cache hit: ${{ steps.cache.outputs.cache-hit }}"
    echo "Cache key: ${{ runner.os }}-tool-${{ hashFiles('**/lockfile') }}"
    ls -la ~/.cache/something || echo "Cache directory does not exist"
```

**Common causes**:
- **Wrong `hashFiles()` glob** → File not found, hash is empty string, key always differs
- **Wrong path** → Caching a directory that doesn't exist
- **Cross-OS caching** → Linux cache can't be used on macOS
- **Cache eviction** → LRU eviction after 7 days of no access or when repo exceeds 10 GB
- **Branch isolation** → PR can read main's cache but not vice versa

**Fix**: Verify the glob pattern finds files:

```yaml
- name: Debug hashFiles
  run: |
    echo "Lock files found:"
    find . -name "package-lock.json" -o -name "go.sum" -o -name "requirements.txt"
```

### 4. Version Mismatches

**Error**: `SyntaxError: Unexpected token` or `Error: The process '/usr/bin/node' failed`

**Cause**: Node.js version mismatch between setup action and code expectations.

**Fix**: Pin versions explicitly:

```yaml
# ❌ BAD — "latest" can change unpredictably
- uses: actions/setup-node@v4
  with:
    node-version: latest

# ✅ GOOD — version file, single source of truth
- uses: actions/setup-node@v4
  with:
    node-version-file: .nvmrc
```

**Action version pinning**:

```yaml
# ❌ BAD — major version can include breaking changes
- uses: actions/checkout@v4

# ✅ BETTER — pin to SHA for critical workflows
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

### 5. Timeout Issues

**Error**: `The job running on runner ... has exceeded the maximum execution time`

**Causes**:
- Hung process (database connection, network call)
- Infinite loop in tests
- Very large build without cache

**Fix**: Set explicit timeouts at job AND step level:

```yaml
jobs:
  build:
    timeout-minutes: 15  # Job-level timeout
    steps:
      - name: Wait for database
        timeout-minutes: 2  # Step-level timeout
        run: |
          for i in $(seq 1 30); do
            pg_isready -h localhost && exit 0
            sleep 2
          done
          exit 1
```

### 6. Rate Limiting

**Error**: `toomanyrequests` or `429 Too Many Requests`

**Targets**: Docker Hub, npm registry, PyPI

**Fixes**:

```yaml
# Docker Hub — authenticate to get higher limits
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

# npm — use cache to reduce registry hits
- uses: actions/setup-node@v4
  with:
    cache: npm
    registry-url: https://registry.npmjs.org
```

### 7. Runner Environment Differences

**Error**: Works on `ubuntu-22.04` but fails on `ubuntu-latest` (which moved to `ubuntu-24.04`)

**Cause**: `ubuntu-latest` is a moving target. It periodically moves to the next LTS.

**Fix**: Pin the runner version for reproducibility:

```yaml
# ❌ BAD — will change when GitHub updates "latest"
runs-on: ubuntu-latest

# ✅ GOOD for reproducibility
runs-on: ubuntu-24.04

# ✅ ALSO GOOD — test on multiple versions
strategy:
  matrix:
    os: [ubuntu-22.04, ubuntu-24.04]
```

---

## YAML Syntax Pitfalls

### Expression Syntax

```yaml
# ❌ WRONG — bare expression
if: github.ref == 'refs/heads/main'

# ✅ CORRECT — both work, but ${{ }} is safer
if: ${{ github.ref == 'refs/heads/main' }}
if: github.ref == 'refs/heads/main'
# Note: bare expressions work in `if:` but NOT in most other fields

# ❌ WRONG — string comparison without quotes
if: ${{ github.event.action == push }}  # 'push' is treated as context variable

# ✅ CORRECT — quote string literals
if: ${{ github.event.action == 'push' }}
```

### Truthy/Falsy Values

```yaml
# These are all TRUTHY (the job runs):
if: true
if: 1
if: 'text'
if: '0'    # String '0' is TRUTHY!

# These are FALSY (the job is skipped):
if: false
if: 0
if: ''
if: null
```

### Multiline Strings

```yaml
# Literal block (preserves newlines) — useful for scripts
- run: |
    echo "line 1"
    echo "line 2"

# Folded block (newlines become spaces) — useful for long commands
- run: >
    echo "this is all
    one single line"

# Literal block with chomp (remove trailing newline)
- run: |-
    echo "no trailing newline"
```

### Environment Variable Scoping

```yaml
# Workflow-level env — available to ALL jobs and steps
env:
  GLOBAL_VAR: global

jobs:
  build:
    # Job-level env — available to all steps in THIS job
    env:
      JOB_VAR: job-level

    steps:
      - name: Step with env
        # Step-level env — available only to THIS step
        env:
          STEP_VAR: step-level
        run: |
          echo "$GLOBAL_VAR"  # ✅ accessible
          echo "$JOB_VAR"     # ✅ accessible
          echo "$STEP_VAR"    # ✅ accessible

      - name: Next step
        run: |
          echo "$GLOBAL_VAR"  # ✅ accessible
          echo "$JOB_VAR"     # ✅ accessible
          echo "$STEP_VAR"    # ❌ NOT accessible (step-level only)
```

### Setting Outputs

```yaml
# ❌ OLD WAY (deprecated) — using set-output
- run: echo "::set-output name=result::value"

# ✅ NEW WAY — using GITHUB_OUTPUT
- id: my-step
  run: echo "result=value" >> "$GITHUB_OUTPUT"

# Multiline output
- id: my-step
  run: |
    {
      echo "result<<EOF"
      echo "line 1"
      echo "line 2"
      echo "EOF"
    } >> "$GITHUB_OUTPUT"

# Using the output
- run: echo "${{ steps.my-step.outputs.result }}"
```

### Secret Masking

```yaml
# Secrets are automatically masked in logs
- run: echo "${{ secrets.MY_SECRET }}"
# Output: ***

# BUT! If you transform the secret, the transformed version is NOT masked
- run: echo "${{ secrets.MY_SECRET }}" | base64
# Output: the base64 value is visible!

# Fix: Register custom masks
- run: |
    DECODED=$(echo "${{ secrets.MY_SECRET }}" | base64)
    echo "::add-mask::$DECODED"
    echo "$DECODED"  # Now masked
```

---

## Debugging Techniques

### Enable Debug Logging

Set these as **repository secrets** for verbose logging:

| Secret | Effect |
|--------|--------|
| `ACTIONS_RUNNER_DEBUG` = `true` | Verbose runner-level logs |
| `ACTIONS_STEP_DEBUG` = `true` | Verbose step-level logs |

Or add on a per-run basis:
```yaml
env:
  ACTIONS_STEP_DEBUG: true
```

### Add Diagnostic Steps

```yaml
# System info
- name: System diagnostics
  if: failure()
  run: |
    echo "=== OS ==="
    cat /etc/os-release
    echo "=== Disk ==="
    df -h
    echo "=== Memory ==="
    free -h
    echo "=== Environment ==="
    env | sort
    echo "=== Docker ==="
    docker version || true
    echo "=== Installed tools ==="
    which go node python java rustc 2>/dev/null || true

# File system inspection
- name: Inspect workspace
  if: failure()
  run: |
    echo "=== Working directory ==="
    pwd
    echo "=== Files ==="
    ls -la
    echo "=== Git status ==="
    git status
    echo "=== Git log ==="
    git log --oneline -5
```

### Test Locally with `act`

[`act`](https://github.com/nektos/act) runs GitHub Actions locally:

```bash
# Install
brew install act  # macOS
# or: curl https://raw.githubusercontent.com/nektos/act/master/install.sh | bash

# Run the default event (push)
act

# Run a specific job
act -j build

# Run with secrets
act -s MY_SECRET=value

# List available workflows and jobs
act -l

# Dry run (validate without executing)
act -n
```

**Limitations of `act`**:
- Some GitHub-hosted runner tools may not be available
- Caching doesn't work the same way
- `GITHUB_TOKEN` is not automatically available
- Some actions may not work (especially ones that call GitHub APIs)

---

## Action-Specific Issues

### actions/checkout

| Issue | Fix |
|-------|-----|
| Shallow clone (missing history) | `fetch-depth: 0` |
| Missing submodules | `submodules: recursive` |
| LFS files are pointers | `lfs: true` |
| Private repo in multi-repo | Use `token:` with PAT |
| Detached HEAD | Checkout is always detached — use `${{ github.sha }}` |

### actions/cache

| Issue | Fix |
|-------|-----|
| Cache never hits | Debug `hashFiles()` glob — ensure files exist |
| Cache is stale | Change a component of the cache key |
| Cache too large | Cache only what's needed, not entire directories |
| PR can't use main's cache | Cache access is branch-scoped — PRs can read main's cache |
| Cache errors don't fail build | Cache actions are non-fatal by default |

### actions/setup-node / actions/setup-go / etc.

| Issue | Fix |
|-------|-----|
| Version not found | Check available versions in the action's docs |
| Cache not working | Ensure lockfile exists in the correct location |
| Wrong architecture | Specify `architecture:` input |
| Multiple versions needed | Use matrix strategy |

---

## Workflow Template: Self-Diagnosing Pipeline

```yaml
# .github/workflows/ci.yml — with built-in diagnostics
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Build
        run: go build ./...

      - name: Test
        run: go test -race ./...

      # === DIAGNOSTICS (only run on failure) ===
      - name: Diagnostics — Environment
        if: failure()
        run: |
          echo "## Failure Diagnostics" >> "$GITHUB_STEP_SUMMARY"
          echo "### Environment" >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
          go version >> "$GITHUB_STEP_SUMMARY"
          go env >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"

      - name: Diagnostics — Disk Space
        if: failure()
        run: |
          echo "### Disk Space" >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
          df -h >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"

      - name: Diagnostics — Dependencies
        if: failure()
        run: |
          echo "### Dependencies" >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
          go mod graph | head -50 >> "$GITHUB_STEP_SUMMARY"
          echo '```' >> "$GITHUB_STEP_SUMMARY"
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| `continue-on-error: true` to suppress failures | Hides real problems | Fix the root cause, remove `continue-on-error` |
| Ignoring annotations and warnings | Warnings become errors over time | Address warnings proactively |
| Hardcoding paths (`/home/runner/...`) | Breaks when runner environment changes | Use `$GITHUB_WORKSPACE`, `$HOME`, `$RUNNER_TEMP` |
| No `if: failure()` diagnostics | Hard to debug remotely | Add diagnostic steps that run on failure |
| Using deprecated action syntax | Will break when deprecated features are removed | Use `GITHUB_OUTPUT` not `set-output`, use `node20` not `node16` |
| Setting `ACTIONS_RUNNER_DEBUG` permanently | Logs become huge and slow | Enable only when debugging, then disable |
| No `timeout-minutes` | Hung jobs burn CI minutes for hours | Always set timeouts |
| Retrying without understanding the failure | Masks intermittent bugs | Investigate flaky failures, add retry only for known-transient issues |

---

## Related Skills

As the diagnostic skill, you may need to load any specialist skill depending on where the failure is:

- **Cache miss or dependency issue?** → `skill({ name: "dependency-manager" })` for cache key strategies
- **Timeout or slow build?** → `skill({ name: "build-optimizer" })` for parallelization and timeout tuning
- **Lint or test failure?** → `skill({ name: "code-quality-gate" })` for quality gate configuration
- **Security scan failure?** → `skill({ name: "security-scanner" })` for scanner configuration
- **Docker build failure?** → `skill({ name: "docker-builder" })` for Dockerfile and BuildKit issues
- **Pipeline design issue?** → `skill({ name: "devops-pipeline-architect" })` for workflow structure
- **Git-related CI issue?** → `skill({ name: "git-commands" })` for checkout, fetch-depth, and ref issues
- **GitHub Actions API issue?** → `skill({ name: "gh-cli" })` for `gh run` and `gh workflow` diagnostics

