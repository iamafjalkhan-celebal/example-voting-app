---
name: build-optimizer
description: Optimize GitHub Actions build speed through matrix strategies, job parallelization, concurrency control, conditional execution, and monorepo change detection. Makes pipelines faster without sacrificing reliability.
---

# Build Optimizer

Expert at **making GitHub Actions pipelines faster and more efficient**. Optimize through parallelization, matrix builds, conditional execution, and intelligent resource allocation.

## When to Use This Skill

- Pipeline takes too long
- User wants to add matrix builds (multiple OS/versions)
- Monorepo needs to build only changed packages
- Jobs run sequentially when they could run in parallel
- Duplicate runs waste CI minutes

---

## Job Parallelization

### Design the Dependency Graph

Jobs without `needs:` run in **parallel by default**. Use `needs:` only for true dependencies.

```yaml
jobs:
  # These 3 jobs run in PARALLEL — no needs: dependency
  lint:
    runs-on: ubuntu-latest
    steps: ...

  test:
    runs-on: ubuntu-latest
    steps: ...

  security:
    runs-on: ubuntu-latest
    steps: ...

  # This job waits for all 3 to pass
  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    steps: ...
```

**Visualization**:
```
lint ──────┐
test ──────┼──→ build
security ──┘
```

### Passing Data Between Jobs

Use artifacts to share build outputs between jobs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: go build -o bin/app ./...

      - uses: actions/upload-artifact@v4
        with:
          name: app-binary
          path: bin/app
          retention-days: 1  # Short-lived, just for this run

  integration-test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-binary

      - run: chmod +x app && ./app --health-check
```

### Job Outputs for Lightweight Data

For small values (strings, booleans), use job outputs instead of artifacts:

```yaml
jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      should-build: ${{ steps.check.outputs.changed }}
    steps:
      - id: check
        run: echo "changed=true" >> "$GITHUB_OUTPUT"

  build:
    needs: detect
    if: needs.detect.outputs.should-build == 'true'
    runs-on: ubuntu-latest
    steps: ...
```

---

## Matrix Builds

### Basic Matrix

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [18, 20, 22]
      # Don't cancel all if one fails — useful for compatibility testing
      fail-fast: false
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test
```

This creates **9 jobs** (3 OS × 3 versions).

### Matrix with Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    go-version: ['1.21', '1.22']
    # Exclude specific combinations
    exclude:
      - os: macos-latest
        go-version: '1.21'
    # Add extra combinations with additional variables
    include:
      - os: ubuntu-latest
        go-version: '1.22'
        coverage: true  # Extra variable for this combo only
```

### When to Use `fail-fast`

| Scenario | `fail-fast` | Reason |
|----------|------------|--------|
| PR validation | `true` (default) | Fast feedback, stop early |
| Compatibility testing | `false` | Need to know ALL failures |
| Release builds | `false` | Must verify every platform |

---

## Concurrency Control

### Cancel Duplicate Runs

When a PR gets new pushes, cancel the in-progress run:

```yaml
concurrency:
  # Same group = same PR or same branch
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

### Protect Main Branch Runs

Don't cancel runs on `main` — they may be creating release artifacts:

```yaml
concurrency:
  group: ci-${{ github.ref }}
  # Only cancel PR runs, not main branch runs
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}
```

### Per-Job Concurrency

Different concurrency groups for different jobs:

```yaml
jobs:
  test:
    concurrency:
      group: test-${{ github.ref }}
      cancel-in-progress: true

  build:
    concurrency:
      group: build-${{ github.ref }}
      cancel-in-progress: false  # Never cancel builds
```

---

## Conditional Execution

### Skip Jobs Based on Context

```yaml
jobs:
  # Only run on PRs, not pushes
  pr-check:
    if: github.event_name == 'pull_request'

  # Only run on main branch pushes
  publish:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

  # Skip if commit message contains [skip ci]
  build:
    if: "!contains(github.event.head_commit.message, '[skip ci]')"

  # Only run if specific label is present on PR
  e2e:
    if: contains(github.event.pull_request.labels.*.name, 'run-e2e')
```

### Path-Based Triggering

At the **workflow** level — run entire workflow only when certain files change:

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
      - '.github/workflows/ci.yml'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

### Change Detection for Monorepos

Use `dorny/paths-filter` for **per-job** path filtering:

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
      shared: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'services/api/**'
              - 'shared/**'
            web:
              - 'services/web/**'
              - 'shared/**'
            shared:
              - 'shared/**'

  build-api:
    needs: changes
    if: needs.changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building API..."

  build-web:
    needs: changes
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building Web..."
```

---

## Timeout Configuration

**Always set timeouts.** Default is 360 minutes (6 hours!) — a hung job wastes CI minutes silently.

```yaml
jobs:
  lint:
    timeout-minutes: 10    # Linting should be fast

  test:
    timeout-minutes: 15    # Tests need more time

  build:
    timeout-minutes: 10    # Builds should be predictable

  # Per-step timeouts for specific long steps
  integration:
    timeout-minutes: 30
    steps:
      - name: Start services
        timeout-minutes: 5    # Fail fast if services don't start
        run: docker compose up -d --wait

      - name: Run integration tests
        timeout-minutes: 20
        run: npm run test:integration
```

### Recommended Timeouts by Job Type

| Job Type | Timeout | Rationale |
|----------|---------|-----------|
| Lint / Format | 5-10 min | Should be near-instant |
| Unit tests | 10-15 min | If longer, tests need optimization |
| Integration tests | 15-30 min | Services startup adds time |
| Build | 10-15 min | If longer, build needs optimization |
| Security scan | 5-15 min | Depends on codebase size |
| Docker build | 10-20 min | Layer caching helps |

---

## Runner Selection

### GitHub-Hosted Runners

| Runner | vCPUs | RAM | Disk | Cost |
|--------|-------|-----|------|------|
| `ubuntu-latest` | 4 | 16 GB | 14 GB SSD | Standard |
| `ubuntu-latest-4-cores` | 4 | 16 GB | 14 GB SSD | Standard |
| `ubuntu-latest-8-cores` | 8 | 32 GB | 28 GB SSD | 2× |
| `ubuntu-latest-16-cores` | 16 | 64 GB | 56 GB SSD | 4× |
| `macos-latest` | 4 | 14 GB | 14 GB SSD | 10× Linux cost |
| `windows-latest` | 4 | 16 GB | 14 GB SSD | 2× Linux cost |

**Guidelines**:
- Use `ubuntu-latest` for everything unless you specifically need macOS/Windows
- Use larger runners only for builds that are CPU/memory bound and where time savings justify cost
- Pin specific versions (`ubuntu-24.04`) for reproducibility in release builds

---

## Incremental Build Strategies

### Compiler Cache (ccache / sccache)

For C/C++/Rust projects:

```yaml
- name: Setup sccache
  uses: mozilla-actions/sccache-action@v0.0.6

- name: Build with sccache
  run: cargo build --release
  env:
    SCCACHE_GHA_ENABLED: true
    RUSTC_WRAPPER: sccache
```

### Turborepo / Nx (JavaScript Monorepos)

```yaml
- name: Build affected packages
  run: npx turbo run build --filter='...[HEAD^1]'
  # or with Nx:
  # run: npx nx affected --target=build --base=HEAD~1
```

### Gradle Build Cache

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version-file: .java-version
    cache: gradle

- name: Build with cache
  run: ./gradlew build --build-cache
```

---

## Workflow Template: Optimized Monorepo CI

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
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  # Step 1: Detect what changed
  changes:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'services/api/**'
              - 'shared/**'
            web:
              - 'services/web/**'
              - 'shared/**'

  # Step 2: Lint everything (fast, always runs)
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - run: echo "Run linters here"

  # Step 3: Build only what changed (parallel)
  build-api:
    needs: [changes, lint]
    if: needs.changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - run: echo "Build API..."

  build-web:
    needs: [changes, lint]
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - run: echo "Build Web..."

  # Step 4: Gate — all builds must pass
  ci-ok:
    needs: [build-api, build-web]
    if: always()
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - name: Check results
        run: |
          if [[ "${{ contains(needs.*.result, 'failure') }}" == "true" ]]; then
            echo "::error::One or more jobs failed"
            exit 1
          fi
          echo "All checks passed ✅"
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Sequential jobs with no data dependency | Wastes time | Remove `needs:` to parallelize |
| `fail-fast: true` on release builds | Hides failures on other platforms | Set `fail-fast: false` |
| No `timeout-minutes` | Hung jobs burn CI minutes for hours | Always set timeouts |
| No `concurrency` group | Duplicate runs waste resources | Add `concurrency:` block |
| Rebuilding unchanged code in monorepo | Wastes time and resources | Use path filters or `dorny/paths-filter` |
| Using macOS/Windows runners unnecessarily | 10×/2× more expensive | Use `ubuntu-latest` unless platform-specific |
| Single large job instead of parallel smaller jobs | Slow and hard to debug | Split into focused parallel jobs |
| Not using `cancel-in-progress` on PR workflows | Old runs keep running | Cancel with `cancel-in-progress: true` |

---

## Related Skills

When a task involves more than build optimization, load these:

- **Caching dependencies?** → `skill({ name: "dependency-manager" })` for setup actions and cache strategies
- **Adding quality gates?** → `skill({ name: "code-quality-gate" })` for lint/test/coverage jobs
- **Designing the full pipeline?** → `skill({ name: "devops-pipeline-architect" })` for pipeline structure
- **Monorepo change detection not working?** → `skill({ name: "workflow-debugger" })` for debugging path filters

