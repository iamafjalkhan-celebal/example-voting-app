---
name: devops-pipeline-architect
description: Design and generate GitHub Actions CI pipelines from checkout through build. The orchestrator skill that delegates to specialized sub-skills for caching, optimization, quality gates, security, Docker, and debugging.
---

# Pipeline Architect — GitHub Actions Orchestrator

You are the **orchestrator** for GitHub Actions CI pipelines scoped from **checkout through build** (no deployment/release). Your job is to design the overall pipeline shape and delegate specifics to sub-skills.

## When to Use This Skill

- User asks to create a CI pipeline or GitHub Actions workflow
- User wants to set up CI for a new or existing project
- User needs a full pipeline designed from scratch
- User asks "set up CI for my project"

## Sub-Skills (Load These via `skill()` Tool)

When designing a pipeline, load the relevant sub-skills for detailed templates and patterns:

| Skill | Load Command | When to Load |
|-------|-------------|-------------|
| dependency-manager | `skill({ name: "dependency-manager" })` | Caching, lockfiles, `actions/setup-*`, private registries |
| build-optimizer | `skill({ name: "build-optimizer" })` | Matrix builds, parallelization, concurrency, conditional execution |
| code-quality-gate | `skill({ name: "code-quality-gate" })` | Linting, formatting, static analysis, coverage |
| security-scanner | `skill({ name: "security-scanner" })` | SAST, dependency audit, secret detection, container scanning |
| docker-builder | `skill({ name: "docker-builder" })` | Dockerfiles, image builds, BuildKit, multi-arch |
| workflow-debugger | `skill({ name: "workflow-debugger" })` | Diagnosing and fixing failing workflows |
| git-commands | `skill({ name: "git-commands" })` | Branching strategies, conventional commits, tags |
| gh-cli | `skill({ name: "gh-cli" })` | PR management, releases, workflow triggers |

**Always load at least one sub-skill before generating pipeline YAML.** This skill provides the pipeline shape; sub-skills provide the implementation details.



## Pipeline Design Methodology

### Step 1: Detect Project Type

Analyze the repository to determine:

```
1. Language(s)     → go.mod? package.json? requirements.txt? pom.xml? Cargo.toml?
2. Build tool      → make, npm, pip, maven, cargo, go build?
3. Test framework  → go test, jest, pytest, junit, cargo test?
4. Docker?         → Dockerfile present?
5. Monorepo?       → Multiple packages/services in one repo?
```

### Step 2: Choose Pipeline Shape

**Single-language project** → Linear pipeline:
```
checkout → setup → lint → test → build → scan
```

**Multi-language / Monorepo** → Parallel pipeline:
```
checkout → ┬─ lint (fast-fail)
           ├─ test-service-a
           ├─ test-service-b
           ├─ build-service-a
           ├─ build-service-b
           └─ scan
```

**Docker project** → Build + scan pipeline:
```
checkout → setup → lint → test → docker-build → image-scan
```

### Step 3: Generate Workflow

Use the templates below as starting points, then delegate to sub-skills to fill in the details.

---

## Workflow Templates

### Go Project

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

# Cancel in-progress runs for the same branch
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest

      - name: Check formatting
        run: |
          if [ -n "$(gofmt -l .)" ]; then
            echo "::error::Files not formatted. Run 'gofmt -w .'"
            gofmt -l .
            exit 1
          fi

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Run tests
        run: go test -race -coverprofile=coverage.out -covermode=atomic ./...

      - name: Check coverage threshold
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          echo "Total coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 60" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below 60% threshold"
            exit 1
          fi

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.out
          retention-days: 7

  build:
    name: Build
    needs: [lint, test]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Build binary
        run: go build -ldflags="-s -w" -o bin/ ./...

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: binary
          path: bin/
          retention-days: 7

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Run govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
```

### Node.js / TypeScript Project

```yaml
# .github/workflows/ci.yml
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
  lint:
    name: Lint & Typecheck
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm ci

      - name: Lint
        run: npx eslint . --max-warnings=0

      - name: Format check
        run: npx prettier --check .

      - name: Typecheck
        run: npx tsc --noEmit

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm ci

      - name: Run tests
        run: npx jest --ci --coverage --reporters=default --reporters=jest-junit
        env:
          JEST_JUNIT_OUTPUT_DIR: reports

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: reports/
          retention-days: 7

      - name: Check coverage threshold
        run: |
          npx jest --ci --coverage --coverageReporters=text | tee coverage.txt
          # Fails if coverage thresholds in jest.config are not met

  build:
    name: Build
    needs: [lint, test]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm ci

      - name: Build
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  security:
    name: Security Audit
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm audit --audit-level=high
```

### Python Project

```yaml
# .github/workflows/ci.yml
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
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version-file: .python-version
          cache: pip

      - name: Install linters
        run: pip install ruff mypy

      - name: Lint
        run: ruff check .

      - name: Format check
        run: ruff format --check .

      - name: Type check
        run: mypy . --ignore-missing-imports

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version-file: .python-version
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Install test dependencies
        run: pip install pytest pytest-cov pytest-xdist

      - name: Run tests
        run: |
          pytest \
            --cov=src \
            --cov-report=xml:coverage.xml \
            --cov-report=term \
            --cov-fail-under=60 \
            --junitxml=reports/junit.xml \
            -n auto

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            reports/
            coverage.xml
          retention-days: 7

  build:
    name: Build Package
    needs: [lint, test]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version-file: .python-version
          cache: pip

      - name: Install build tools
        run: pip install build

      - name: Build package
        run: python -m build

      - name: Upload package
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version-file: .python-version
          cache: pip

      - name: Install pip-audit
        run: pip install pip-audit

      - name: Audit dependencies
        run: pip-audit -r requirements.txt
```

### Java / Maven Project

```yaml
# .github/workflows/ci.yml
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
  lint:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version-file: .java-version
          cache: maven

      - name: Checkstyle
        run: ./mvnw checkstyle:check -q

      - name: SpotBugs
        run: ./mvnw spotbugs:check -q

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version-file: .java-version
          cache: maven

      - name: Run tests with coverage
        run: ./mvnw verify -q

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            **/target/surefire-reports/
            **/target/site/jacoco/
          retention-days: 7

  build:
    name: Build
    needs: [lint, test]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version-file: .java-version
          cache: maven

      - name: Build package
        run: ./mvnw package -DskipTests -q

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: jar
          path: target/*.jar
          retention-days: 7
```

### Rust Project

```yaml
# .github/workflows/ci.yml
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

env:
  CARGO_TERM_COLOR: always

jobs:
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - uses: Swatinem/rust-cache@v2

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2

      - name: Run tests
        run: cargo test --all-features --workspace

  build:
    name: Build
    needs: [lint, test]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2

      - name: Build release
        run: cargo build --release

      - name: Upload binary
        uses: actions/upload-artifact@v4
        with:
          name: binary
          path: target/release/
          retention-days: 7

  security:
    name: Security Audit
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Reusable Workflow Patterns

When a project needs shared CI logic across multiple repos, generate **reusable workflows**:

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      go-version:
        description: Go version to use
        required: false
        type: string
        default: '1.22'
      coverage-threshold:
        description: Minimum coverage percentage
        required: false
        type: number
        default: 60

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ inputs.go-version }}
          cache: true
      - run: go build ./...
      - run: go test -coverprofile=coverage.out ./...
```

**Caller workflow**:
```yaml
# .github/workflows/ci.yml
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      go-version: '1.22'
      coverage-threshold: 80
```

---

## Composite Action Pattern

For shared steps across jobs within a workflow:

```yaml
# .github/actions/setup-go-env/action.yml
name: Setup Go Environment
description: Checkout code, setup Go, and restore cache

inputs:
  go-version-file:
    description: Path to go.mod
    default: go.mod

runs:
  using: composite
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with:
        go-version-file: ${{ inputs.go-version-file }}
        cache: true
```

---

## Decision Tree

When a user asks for CI, follow this decision tree and **load the indicated skills**:

```
1. Is there a Dockerfile?
   ├─ YES → Include docker-build job → skill({ name: "docker-builder" })
   └─ NO  → Skip docker jobs

2. What language(s)?
   ├─ Go      → Go template → skill({ name: "dependency-manager" }) + skill({ name: "code-quality-gate" })
   ├─ Node/TS → Node template → skill({ name: "dependency-manager" }) + skill({ name: "code-quality-gate" })
   ├─ Python  → Python template → skill({ name: "dependency-manager" }) + skill({ name: "code-quality-gate" })
   ├─ Java    → Java template → skill({ name: "dependency-manager" }) + skill({ name: "code-quality-gate" })
   ├─ Rust    → Rust template → skill({ name: "dependency-manager" }) + skill({ name: "code-quality-gate" })
   └─ Multi   → Combine templates → skill({ name: "build-optimizer" })

3. Is it a monorepo?
   ├─ YES → Add path filters → skill({ name: "build-optimizer" })
   └─ NO  → Standard single-project pipeline

4. Does the user want security scanning beyond dependency audit?
   ├─ YES → Add CodeQL job → skill({ name: "security-scanner" })
   └─ NO  → Keep basic dependency audit only
```

## Anti-Patterns to Avoid

- **No permissions block** — Always set minimal `permissions:` at workflow level
- **No concurrency control** — Always add `concurrency:` to avoid duplicate runs
- **No timeouts** — Always set `timeout-minutes` on every job
- **Running everything sequentially** — Parallelize lint, test, security. Only `build` should depend on them
- **Hardcoding versions** — Use version files (`.nvmrc`, `go.mod`, `.python-version`, `.java-version`)
- **Missing `if: always()` on artifact uploads** — Test results should upload even on failure
- **No artifact retention policy** — Always set `retention-days` to avoid storage bloat

---

## Related Skills

This is the orchestrator skill — it designs pipeline shape. Load sub-skills for implementation details:

- **Caching & setup?** → `skill({ name: "dependency-manager" })`
- **Speed optimization?** → `skill({ name: "build-optimizer" })`
- **Quality gates?** → `skill({ name: "code-quality-gate" })`
- **Security scanning?** → `skill({ name: "security-scanner" })`
- **Docker builds?** → `skill({ name: "docker-builder" })`
- **Pipeline failing?** → `skill({ name: "workflow-debugger" })`
- **Git workflow?** → `skill({ name: "git-commands" })`
- **GitHub operations?** → `skill({ name: "gh-cli" })`