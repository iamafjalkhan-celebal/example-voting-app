---
name: code-quality-gate
description: Configure linting, formatting, static analysis, type checking, and test coverage as GitHub Actions pipeline gates. Covers all major languages with inline PR annotations and enforcement strategies.
---

# Code Quality Gate

Expert at **configuring quality checks as CI gates** in GitHub Actions — linters, formatters, static analyzers, type checkers, and coverage enforcement. Quality gates run fast, fail early, and annotate PRs with actionable feedback.

## When to Use This Skill

- User wants to add linting or formatting checks to CI
- Pipeline needs code quality enforcement
- User asks about coverage thresholds
- User wants inline PR annotations from linters
- User needs to set up static analysis

---

## Core Principle: Lint Before Build

Quality gates should be the **first job** in any pipeline — they're fast and catch issues before expensive build/test steps:

```
lint (2-5 min) ──→ test (10-15 min) ──→ build (5-10 min)
         ↑                                    
   Fail fast here                        
```

---

## Language-Specific Quality Gates

### Go

```yaml
jobs:
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      # Comprehensive linting via golangci-lint
      - name: Lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          # Produces GitHub annotations inline on PR diffs
          args: --timeout=5m

      # Format check — gofmt is non-negotiable in Go
      - name: Format check
        run: |
          unformatted=$(gofmt -l .)
          if [ -n "$unformatted" ]; then
            echo "::error::Files not formatted with gofmt:"
            echo "$unformatted"
            exit 1
          fi

      # Vet — catches common mistakes the compiler doesn't
      - name: Vet
        run: go vet ./...
```

**golangci-lint configuration** (`.golangci.yml`):
```yaml
linters:
  enable:
    - errcheck      # Unchecked errors
    - govet         # Vet analysis
    - staticcheck   # Advanced static analysis
    - unused        # Unused code
    - ineffassign   # Ineffectual assignments
    - gocritic      # Opinionated style checks
    - revive        # Replacement for golint
    - gosec         # Security issues

linters-settings:
  errcheck:
    check-type-assertions: true
  gocritic:
    enabled-tags:
      - diagnostic
      - style
      - performance

issues:
  max-issues-per-linter: 50
  max-same-issues: 10
```

### JavaScript / TypeScript

```yaml
jobs:
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm ci

      # Lint — ESLint with zero warnings policy
      - name: Lint
        run: npx eslint . --max-warnings=0 --format=stylish

      # Format check — Prettier (check mode, no auto-fix)
      - name: Format check
        run: npx prettier --check .

      # Type check — TypeScript compiler
      - name: Typecheck
        run: npx tsc --noEmit

      # Optional: check for unused exports
      - name: Find dead code
        run: npx ts-prune | grep -v '(used in module)' || true
```

### Python

```yaml
jobs:
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version-file: .python-version
          cache: pip

      - name: Install quality tools
        run: pip install ruff mypy

      # Lint — ruff is the fastest Python linter
      - name: Lint
        run: ruff check . --output-format=github

      # Format check — ruff format (replaces black + isort)
      - name: Format check
        run: ruff format --check .

      # Type check — mypy
      - name: Type check
        run: mypy . --ignore-missing-imports
```

**ruff configuration** (`ruff.toml`):
```toml
target-version = "py311"
line-length = 88

[lint]
select = [
  "E",    # pycodestyle errors
  "W",    # pycodestyle warnings
  "F",    # pyflakes
  "I",    # isort
  "UP",   # pyupgrade
  "B",    # flake8-bugbear
  "S",    # flake8-bandit (security)
  "SIM",  # flake8-simplify
]

[lint.per-file-ignores]
"tests/**" = ["S101"]  # Allow assert in tests
```

### Java (Maven)

```yaml
jobs:
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version-file: .java-version
          cache: maven

      # Checkstyle — style enforcement
      - name: Checkstyle
        run: ./mvnw checkstyle:check -q

      # SpotBugs — bug detection
      - name: SpotBugs
        run: ./mvnw spotbugs:check -q

      # PMD — code analysis
      - name: PMD
        run: ./mvnw pmd:check -q
```

### Rust

```yaml
jobs:
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - uses: Swatinem/rust-cache@v2

      # Format check
      - name: Format check
        run: cargo fmt --all -- --check

      # Clippy — the Rust linter
      - name: Clippy
        run: cargo clippy --all-targets --all-features -- -D warnings
```

---

## Coverage Enforcement

### Go Coverage

```yaml
- name: Run tests with coverage
  run: go test -race -coverprofile=coverage.out -covermode=atomic ./...

- name: Check coverage threshold
  run: |
    COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
    echo "Total coverage: ${COVERAGE}%"
    echo "## Coverage: ${COVERAGE}%" >> "$GITHUB_STEP_SUMMARY"
    if (( $(echo "$COVERAGE < 60" | bc -l) )); then
      echo "::error::Coverage ${COVERAGE}% is below 60% threshold"
      exit 1
    fi
```

### Node.js Coverage (Jest)

```yaml
- name: Run tests with coverage
  run: npx jest --ci --coverage --coverageReporters=text --coverageReporters=lcov
  # Jest enforces thresholds via jest.config:
  # coverageThreshold: { global: { branches: 70, functions: 70, lines: 80, statements: 80 } }

- name: Upload coverage to summary
  run: |
    echo "## Coverage Report" >> "$GITHUB_STEP_SUMMARY"
    npx jest --ci --coverage --coverageReporters=text 2>&1 | tail -20 >> "$GITHUB_STEP_SUMMARY"
```

### Python Coverage (pytest-cov)

```yaml
- name: Run tests with coverage
  run: |
    pytest \
      --cov=src \
      --cov-report=xml:coverage.xml \
      --cov-report=term \
      --cov-fail-under=60 \
      --junitxml=reports/junit.xml \
      -n auto  # Parallel test execution
```

---

## PR Annotations

### GitHub Problem Matchers

Register a problem matcher to get inline PR annotations from linter output:

```yaml
# .github/matchers/eslint.json
{
  "problemMatcher": [
    {
      "owner": "eslint",
      "pattern": [
        {
          "regexp": "^(.+):(\\d+):(\\d+):\\s+(error|warning)\\s+(.+)$",
          "file": 1,
          "line": 2,
          "column": 3,
          "severity": 4,
          "message": 5
        }
      ]
    }
  ]
}
```

```yaml
# In workflow:
- name: Register ESLint matcher
  run: echo "::add-matcher::.github/matchers/eslint.json"

- name: Lint
  run: npx eslint . --format=unix
```

### GitHub Actions Annotations

Many tools support GitHub's annotation format natively:
- `golangci-lint-action` → automatic annotations
- `ruff check --output-format=github` → automatic annotations
- `tsc` → use `problem-matchers` from `actions/setup-node`

---

## Test Result Reporting

### JUnit XML to GitHub Summary

```yaml
- name: Run tests
  run: pytest --junitxml=reports/junit.xml
  if: always()

- name: Publish test results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Test Results
    path: reports/junit.xml
    reporter: java-junit
```

### GitHub Step Summary

Write results directly to the job summary:

```yaml
- name: Test summary
  if: always()
  run: |
    echo "## Test Results 🧪" >> "$GITHUB_STEP_SUMMARY"
    echo "" >> "$GITHUB_STEP_SUMMARY"
    echo "| Metric | Value |" >> "$GITHUB_STEP_SUMMARY"
    echo "|--------|-------|" >> "$GITHUB_STEP_SUMMARY"
    echo "| Tests | 142 |" >> "$GITHUB_STEP_SUMMARY"
    echo "| Passed | 140 |" >> "$GITHUB_STEP_SUMMARY"
    echo "| Failed | 2 |" >> "$GITHUB_STEP_SUMMARY"
    echo "| Coverage | 78.3% |" >> "$GITHUB_STEP_SUMMARY"
```

---

## Workflow Template: Universal Quality Gate

```yaml
# .github/workflows/quality.yml
name: Quality Gate

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  checks: write  # For test result annotations

concurrency:
  group: quality-${{ github.ref }}
  cancel-in-progress: true

jobs:
  quality:
    name: Quality Gate
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      # === SETUP (language-specific, see dependency-manager skill) ===
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci

      # === LINT (fastest checks first) ===
      - name: Lint
        run: npx eslint . --max-warnings=0

      - name: Format
        run: npx prettier --check .

      - name: Typecheck
        run: npx tsc --noEmit

      # === TEST (with coverage) ===
      - name: Test
        run: npx jest --ci --coverage --coverageReporters=text-summary

      # === SUMMARY ===
      - name: Quality summary
        if: always()
        run: |
          echo "## Quality Gate Results" >> "$GITHUB_STEP_SUMMARY"
          echo "✅ Lint, Format, Typecheck, Tests" >> "$GITHUB_STEP_SUMMARY"
```

---

## Commit Message and PR Checks

### Commit Message Linting (Conventional Commits)

```yaml
jobs:
  commitlint:
    name: Commit Messages
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm install @commitlint/cli @commitlint/config-conventional

      - name: Validate commits
        run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to HEAD
```

### PR Title Check

```yaml
jobs:
  pr-title:
    name: PR Title
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    timeout-minutes: 2
    steps:
      - name: Check PR title format
        run: |
          TITLE="${{ github.event.pull_request.title }}"
          if ! echo "$TITLE" | grep -qE '^(feat|fix|docs|chore|refactor|test|ci|perf|style|build)(\(.+\))?: .+'; then
            echo "::error::PR title must follow Conventional Commits format"
            echo "Expected: type(scope): description"
            echo "Got: $TITLE"
            exit 1
          fi
```

---

## Enforcement Strategy

### Check vs Enforce

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Check** (non-blocking) | Report issues, pass anyway | Gradual adoption, new rules |
| **Enforce** (blocking) | Fail the build on issues | Established rules |

```yaml
# Check mode — report but don't fail
- name: New lint rules (informational)
  run: npx eslint . --rule '{"new-rule": "warn"}' || true
  continue-on-error: true

# Enforce mode — fail the build
- name: Core lint rules (enforced)
  run: npx eslint . --max-warnings=0
```

### Branch Protection

Configure GitHub branch protection to **require** quality gate jobs to pass:
1. Settings → Branches → Branch protection rules
2. "Require status checks to pass before merging"
3. Select your quality gate job names

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Auto-fixing in CI (`--fix`) | Hides issues from developers | Use `--check` mode; fix locally |
| Running linters after build | Slow feedback loop | Move linters to the first job |
| `continue-on-error: true` on enforced checks | Quality gate is meaningless | Fail the build on violations |
| No `--max-warnings=0` on ESLint | Warnings accumulate forever | Treat all warnings as errors in CI |
| Coverage threshold of 100% | Flaky, discourages meaningful tests | Use 60-80% as practical threshold |
| Ignoring type errors | TypeScript types catch real bugs | Run `tsc --noEmit` in CI |
| No PR annotations | Developers have to dig through logs | Use problem matchers or annotation formatters |
| Running tests without `-race` (Go) | Misses data race bugs | Always use `-race` flag |

---

## Related Skills

When a task involves more than code quality, load these:

- **Optimizing job speed?** → `skill({ name: "build-optimizer" })` for parallelization and matrix strategies
- **Adding security scanning?** → `skill({ name: "security-scanner" })` for SAST, dependency audit, and secret detection
- **Setting up dependencies/caching?** → `skill({ name: "dependency-manager" })` for setup actions
- **Designing the full pipeline?** → `skill({ name: "devops-pipeline-architect" })` for pipeline structure

