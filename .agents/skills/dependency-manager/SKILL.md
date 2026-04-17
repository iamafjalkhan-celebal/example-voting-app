---
name: dependency-manager
description: Optimize dependency caching, lockfile management, and environment setup in GitHub Actions. Covers actions/cache, actions/setup-*, private registries, and monorepo dependency strategies.
---

# Dependency Manager

Expert at **dependency caching, resolution, and environment setup** for GitHub Actions pipelines. Your goal is to make dependency installation as fast and reliable as possible.

## When to Use This Skill

- User asks about caching in GitHub Actions
- Pipeline has slow dependency installation
- User needs to configure private registry access
- User asks about `actions/cache` or `actions/setup-*`
- Monorepo needs per-package dependency management

---

## Setup Actions — Best Practices

### Go

```yaml
- uses: actions/setup-go@v5
  with:
    # PREFER: version from go.mod (single source of truth)
    go-version-file: go.mod
    # Built-in cache — no separate actions/cache needed
    cache: true
```

**Key points**:
- `actions/setup-go@v5` has **built-in caching** — do NOT add a separate `actions/cache` step for Go modules
- Always use `go-version-file: go.mod` instead of hardcoding `go-version: '1.22'`
- Cache key is automatically derived from `go.sum`

### Node.js

```yaml
- uses: actions/setup-node@v4
  with:
    # PREFER: version file (single source of truth)
    node-version-file: .nvmrc  # or .node-version or package.json#engines
    # Built-in cache — specify package manager
    cache: npm  # or yarn, pnpm
```

**Key points**:
- Built-in cache caches the **global package cache**, not `node_modules`
- Always use `npm ci` (not `npm install`) in CI — it's faster and deterministic
- For pnpm: set `cache: pnpm` and ensure `pnpm-lock.yaml` exists
- For yarn: set `cache: yarn`

**npm ci vs npm install**:
| | `npm ci` | `npm install` |
|---|---------|--------------|
| Speed | Faster (skips resolution) | Slower |
| Deterministic | Yes (uses lockfile exactly) | No (may update lockfile) |
| CI use | ✅ Always | ❌ Never |

### Python

```yaml
- uses: actions/setup-python@v5
  with:
    python-version-file: .python-version
    cache: pip  # or pipenv, poetry
```

**Key points**:
- Built-in cache caches the **pip download cache**, not the virtual environment
- Cache key is derived from `requirements.txt` (or `Pipfile.lock`, `poetry.lock`)
- Consider caching the virtual environment separately for larger projects (see Advanced section)

### Java

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin  # or corretto, zulu, microsoft
    java-version-file: .java-version
    cache: maven  # or gradle, sbt
```

**Key points**:
- Caches `~/.m2/repository` for Maven, `~/.gradle/caches` for Gradle
- Always use Maven Wrapper (`./mvnw`) so the CI matches local dev
- For multi-module Maven projects, all modules share the same cache

### Rust

```yaml
# Rust uses a community action — no official actions/setup-rust
- uses: dtolnay/rust-toolchain@stable
  with:
    components: clippy, rustfmt

# Cache separately — no built-in cache in toolchain action
- uses: Swatinem/rust-cache@v2
```

**Key points**:
- `Swatinem/rust-cache@v2` caches `~/.cargo` and `target/` intelligently
- It automatically handles cache key generation from `Cargo.lock`
- Prunes old cache entries to stay within GitHub's cache size limits

---

## Manual Cache Configuration

When setup actions don't cover your needs, use `actions/cache` directly:

### Cache Key Strategy

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/some-tool
    # Primary key — exact match required
    key: ${{ runner.os }}-tool-${{ hashFiles('**/lockfile') }}
    # Fallback keys — prefix match, may be stale
    restore-keys: |
      ${{ runner.os }}-tool-
```

**Key composition rules**:

| Component | Purpose | Example |
|-----------|---------|---------|
| `runner.os` | OS-specific binaries | `Linux`, `macOS`, `Windows` |
| `hashFiles()` | Content-based invalidation | `hashFiles('**/go.sum')` |
| branch name | Branch isolation (optional) | `${{ github.ref_name }}` |
| static string | Tool identifier | `node-modules`, `go-build` |

**Cache key patterns by ecosystem**:

```yaml
# Go
key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

# Node
key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# Python
key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}

# Maven
key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

# Rust
key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
```

### Restore Key Fallbacks

Restore keys allow partial cache hits. Order matters — most specific first:

```yaml
restore-keys: |
  ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
  ${{ runner.os }}-node-
```

This means: try exact lockfile match first, then fall back to any cache for this OS + tool.

---

## Advanced: Caching Virtual Environments (Python)

For large Python projects, cache the entire virtual environment:

```yaml
- name: Cache virtual environment
  uses: actions/cache@v4
  id: venv-cache
  with:
    path: .venv
    key: ${{ runner.os }}-venv-${{ hashFiles('**/requirements*.txt') }}

- name: Install dependencies
  if: steps.venv-cache.outputs.cache-hit != 'true'
  run: |
    python -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
```

**Why**: Pip's download cache only saves download time. Caching the venv also skips wheel building and installation.

---

## Private Registry Authentication

### npm (GitHub Packages / private registry)

```yaml
- uses: actions/setup-node@v4
  with:
    node-version-file: .nvmrc
    cache: npm
    registry-url: https://npm.pkg.github.com
    scope: '@your-org'

- run: npm ci
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### pip (private PyPI)

```yaml
- name: Install from private index
  run: pip install -r requirements.txt --extra-index-url https://pypi.your-org.com/simple/
  env:
    PIP_EXTRA_INDEX_URL: https://${{ secrets.PYPI_TOKEN }}@pypi.your-org.com/simple/
```

### Go (GOPRIVATE)

```yaml
- name: Configure private Go modules
  run: git config --global url."https://${{ secrets.GH_TOKEN }}@github.com/".insteadOf "https://github.com/"
  env:
    GOPRIVATE: github.com/your-org/*
```

### Maven (private repository)

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version-file: .java-version
    cache: maven
    server-id: private-repo
    server-username: ${{ secrets.MAVEN_USERNAME }}
    server-password: ${{ secrets.MAVEN_PASSWORD }}
```

---

## Monorepo Dependency Strategies

### Shared lockfile (hoisted)

```yaml
# If the monorepo has a root lockfile
- uses: actions/setup-node@v4
  with:
    node-version-file: .nvmrc
    cache: npm

- run: npm ci  # Installs everything from root
```

### Per-package lockfiles

```yaml
# Cache each package separately
- uses: actions/cache@v4
  with:
    path: |
      packages/api/node_modules
      packages/web/node_modules
    key: ${{ runner.os }}-modules-${{ hashFiles('packages/*/package-lock.json') }}
```

### Selective install (only changed packages)

Use with the build-optimizer skill's change detection to install only what's needed. Load it with `skill({ name: "build-optimizer" })`.

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Adding `actions/cache` when setup action has built-in cache | Duplicate caching, wasted time | Use the setup action's `cache` input |
| Caching `node_modules` directly | Platform-specific, can corrupt | Cache npm's global cache via `cache: npm` |
| Using `npm install` in CI | Non-deterministic, may update lockfile | Use `npm ci` |
| Caching build outputs (`dist/`, `target/`) in dependency cache | Different invalidation lifecycle | Use `actions/upload-artifact` for build outputs |
| No `restore-keys` fallback | Cache miss = full download every time | Add prefix-based fallbacks |
| Caching too broadly (`~/.cache`) | Cache bloat, slow restore times | Cache only specific tool directories |
| Hardcoding tool versions | Drift between CI and local dev | Use version files (`.nvmrc`, `go.mod`, etc.) |
| Committing lockfile changes from CI | Defeats deterministic builds | CI should only read lockfiles, never write |

---

## Workflow Template: Optimized Dependency Setup

```yaml
# Reusable composite action for dependency setup
# .github/actions/setup-deps/action.yml
name: Setup Dependencies
description: Install and cache project dependencies

inputs:
  language:
    description: 'go, node, python, java, rust'
    required: true

runs:
  using: composite
  steps:
    # Go
    - if: inputs.language == 'go'
      uses: actions/setup-go@v5
      with:
        go-version-file: go.mod
        cache: true

    # Node
    - if: inputs.language == 'node'
      uses: actions/setup-node@v4
      with:
        node-version-file: .nvmrc
        cache: npm
    - if: inputs.language == 'node'
      run: npm ci
      shell: bash

    # Python
    - if: inputs.language == 'python'
      uses: actions/setup-python@v5
      with:
        python-version-file: .python-version
        cache: pip
    - if: inputs.language == 'python'
      run: pip install -r requirements.txt
      shell: bash

    # Java
    - if: inputs.language == 'java'
      uses: actions/setup-java@v4
      with:
        distribution: temurin
        java-version-file: .java-version
        cache: maven

    # Rust
    - if: inputs.language == 'rust'
      uses: dtolnay/rust-toolchain@stable
    - if: inputs.language == 'rust'
      uses: Swatinem/rust-cache@v2
```

---

## GitHub Actions Cache Limits

| Limit | Value |
|-------|-------|
| Max cache entry size | 10 GB |
| Total cache per repo | 10 GB |
| Cache eviction | LRU, entries unused for 7+ days |
| Max cache entries | No hard limit, but total size applies |

When approaching limits, prioritize caching:
1. Dependencies (slowest to reinstall)
2. Build tools (if not provided by runner)
3. Build caches (only if significant time savings)

---

## Related Skills

When a task involves more than dependency management, load these:

- **Optimizing build speed?** → `skill({ name: "build-optimizer" })` for parallelization and matrix strategies
- **Setting up Docker builds?** → `skill({ name: "docker-builder" })` for Dockerfile layer caching
- **Designing the full pipeline?** → `skill({ name: "devops-pipeline-architect" })` for pipeline structure
- **Private registry auth issues?** → `skill({ name: "workflow-debugger" })` for diagnosing auth failures

