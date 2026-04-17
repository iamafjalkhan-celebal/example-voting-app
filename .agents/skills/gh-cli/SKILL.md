---
name: gh-cli
description: GitHub CLI (gh) operations — PR lifecycle, issue management, release automation, workflow runs, repo management, and API access. Use when the user needs to interact with GitHub from the command line.
---

# GitHub CLI (gh)

Expert reference for **GitHub CLI operations**. Covers the full lifecycle of PRs, issues, releases, workflow management, and direct API access.

## When to Use This Skill

- User needs to create, review, or merge PRs
- User wants to automate releases
- User needs to manage GitHub issues
- User asks about triggering or monitoring workflow runs
- User wants to interact with the GitHub API directly
- User asks "how do I do X on GitHub from the terminal?"

---

## Authentication

### Check Status

```bash
# Verify authentication
gh auth status

# Show which account is active
gh auth status --hostname github.com
```

### Login

```bash
# Interactive login (browser-based)
gh auth login

# Login with token
gh auth login --with-token < token.txt

# Login to GitHub Enterprise
gh auth login --hostname github.example.com

# Switch accounts
gh auth switch
```

---

## Pull Requests

### Create PR

```bash
# Interactive (prompts for title, body, reviewers)
gh pr create

# Non-interactive with full details
gh pr create \
  --title "feat(auth): add OAuth2 support" \
  --body "## Changes
- Add OAuth2 middleware
- Add token refresh logic
- Add integration tests

Closes #42" \
  --reviewer "team-lead,security-reviewer" \
  --label "feature,needs-review" \
  --milestone "v2.0" \
  --assignee "@me"

# Create draft PR
gh pr create --draft --title "WIP: database migration"

# Create PR with specific base branch
gh pr create --base develop --title "feat: new dashboard"

# Create PR from a fork
gh pr create --repo upstream-org/repo --head my-fork:feat/branch
```

### List & View PRs

```bash
# List open PRs
gh pr list

# List your PRs
gh pr list --author "@me"

# List PRs needing your review
gh pr list --search "review-requested:@me"

# List PRs by label
gh pr list --label "bug"

# List PRs in JSON (for scripting)
gh pr list --json number,title,author --jq '.[] | "\(.number): \(.title) by \(.author.login)"'

# View a specific PR
gh pr view 42

# View PR in browser
gh pr view 42 --web

# View PR diff
gh pr diff 42
```

### Review PR

```bash
# Approve
gh pr review 42 --approve

# Request changes
gh pr review 42 --request-changes --body "Please fix the SQL injection in line 23"

# Comment (neither approve nor reject)
gh pr review 42 --comment --body "Looks good overall, minor suggestion on line 45"
```

### Merge PR

```bash
# Merge (default strategy from repo settings)
gh pr merge 42

# Squash merge (single commit)
gh pr merge 42 --squash --subject "feat(auth): add OAuth2 support (#42)"

# Rebase merge
gh pr merge 42 --rebase

# Merge and delete branch
gh pr merge 42 --squash --delete-branch

# Auto-merge when checks pass
gh pr merge 42 --auto --squash --delete-branch
```

### Checkout PR Locally

```bash
# Checkout PR branch
gh pr checkout 42

# Checkout PR from a fork (read-only)
gh pr checkout 42 --force
```

### Other PR Operations

```bash
# Close PR without merging
gh pr close 42

# Reopen a closed PR
gh pr reopen 42

# Mark PR as ready (from draft)
gh pr ready 42

# Add reviewers
gh pr edit 42 --add-reviewer "person1,person2"

# Add labels
gh pr edit 42 --add-label "urgent,security"
```

---

## Issues

### Create Issue

```bash
# Interactive
gh issue create

# Non-interactive
gh issue create \
  --title "Bug: login fails with SSO" \
  --body "## Steps to Reproduce
1. Click Login with SSO
2. Complete SAML flow
3. Redirected back but session is empty

## Expected: Logged in
## Actual: Blank page" \
  --label "bug,high-priority" \
  --assignee "@me" \
  --milestone "v1.3"

# Create from a template
gh issue create --template "bug_report.md"
```

### List & View Issues

```bash
# List open issues
gh issue list

# List issues assigned to you
gh issue list --assignee "@me"

# List by label
gh issue list --label "bug" --label "high-priority"

# List closed issues
gh issue list --state closed --limit 20

# Search issues
gh issue list --search "auth timeout in:title"

# View specific issue
gh issue view 42

# View in browser
gh issue view 42 --web
```

### Manage Issues

```bash
# Close issue
gh issue close 42

# Close with comment
gh issue close 42 --comment "Fixed in #45"

# Reopen
gh issue reopen 42

# Add label
gh issue edit 42 --add-label "wontfix"

# Assign
gh issue edit 42 --add-assignee "developer1"

# Comment on issue
gh issue comment 42 --body "I can reproduce this on v1.2.3"

# Pin issue
gh issue pin 42

# Transfer issue to another repo
gh issue transfer 42 other-org/other-repo
```

---

## Releases

### Create Release

```bash
# Create release from a tag (interactive — opens editor for notes)
gh release create v1.2.0

# Create with auto-generated notes
gh release create v1.2.0 --generate-notes

# Create with title and notes
gh release create v1.2.0 \
  --title "v1.2.0 — OAuth2 Support" \
  --notes "## What's New
- OAuth2 authentication (#42)
- Dashboard redesign (#38)

## Bug Fixes
- Fix memory leak in WebSocket handler (#41)" \
  --target main

# Create pre-release
gh release create v2.0.0-beta.1 --prerelease --generate-notes

# Create draft release
gh release create v1.3.0 --draft --generate-notes

# Create release with binary assets
gh release create v1.2.0 \
  --generate-notes \
  ./dist/app-linux-amd64 \
  ./dist/app-darwin-arm64 \
  ./dist/app-windows-amd64.exe

# Create release from notes file
gh release create v1.2.0 --notes-file CHANGELOG.md
```

### List & View Releases

```bash
# List releases
gh release list

# List including drafts and pre-releases
gh release list --include-drafts --include-pre-releases

# View a specific release
gh release view v1.2.0

# View in browser
gh release view v1.2.0 --web
```

### Download & Delete

```bash
# Download all assets from a release
gh release download v1.2.0

# Download specific asset
gh release download v1.2.0 --pattern "*.tar.gz"

# Download to specific directory
gh release download v1.2.0 --dir ./downloads

# Delete a release
gh release delete v1.2.0 --yes

# Delete release and its tag
gh release delete v1.2.0 --yes --cleanup-tag
```

---

## Workflow Runs

### Trigger Workflows

```bash
# Trigger workflow_dispatch event
gh workflow run ci.yml

# Trigger with inputs
gh workflow run deploy.yml \
  --field environment=staging \
  --field image_tag=v1.2.0

# Trigger on a specific branch
gh workflow run ci.yml --ref feat/my-branch
```

### List & View Runs

```bash
# List recent workflow runs
gh run list

# List runs for a specific workflow
gh run list --workflow ci.yml

# List failed runs
gh run list --status failure

# List runs for a branch
gh run list --branch main

# View a specific run
gh run view 12345

# View run logs
gh run view 12345 --log

# View failed step logs only
gh run view 12345 --log-failed

# View in browser
gh run view 12345 --web
```

### Monitor & Control

```bash
# Watch a run in real-time (blocks until complete)
gh run watch 12345

# Watch the latest run
gh run watch

# Re-run failed jobs
gh run rerun 12345 --failed

# Re-run entire workflow
gh run rerun 12345

# Cancel a run
gh run cancel 12345

# Download artifacts from a run
gh run download 12345

# Download specific artifact
gh run download 12345 --name "test-results"
```

---

## Repository Operations

### Clone & Create

```bash
# Clone a repo
gh repo clone org/repo

# Clone and cd into it
gh repo clone org/repo -- --depth=1

# Create new repo
gh repo create my-app --public --description "My application"

# Create from template
gh repo create my-app --template org/template-repo --public

# Create and clone
gh repo create my-app --public --clone
```

### Fork & Sync

```bash
# Fork a repo
gh repo fork org/repo

# Fork and clone
gh repo fork org/repo --clone

# Sync fork with upstream
gh repo sync owner/repo

# Sync local fork
gh repo sync --source upstream/repo
```

### View & Edit

```bash
# View repo info
gh repo view

# View in browser
gh repo view --web

# Edit repo settings
gh repo edit --default-branch main
gh repo edit --enable-issues
gh repo edit --visibility public
```

---

## GitHub API Access

### REST API

```bash
# GET request
gh api repos/org/repo

# GET with query parameters
gh api repos/org/repo/pulls --method GET -f state=open -f per_page=5

# POST request (create an issue)
gh api repos/org/repo/issues --method POST \
  -f title="Bug report" \
  -f body="Description of the bug"

# PATCH request (update something)
gh api repos/org/repo/issues/42 --method PATCH \
  -f state=closed

# Paginate results
gh api repos/org/repo/commits --paginate --jq '.[].sha'
```

### GraphQL API

```bash
# Simple query
gh api graphql -f query='
  query {
    viewer {
      login
      repositories(first: 10) {
        nodes { name }
      }
    }
  }
'

# Query with variables
gh api graphql -f query='
  query($owner: String!, $repo: String!) {
    repository(owner: $owner, name: $repo) {
      pullRequests(first: 5, states: OPEN) {
        nodes {
          number
          title
          author { login }
        }
      }
    }
  }
' -f owner="org" -f repo="repo"
```

### JQ Filtering

```bash
# Extract specific fields
gh pr list --json number,title,author --jq '.[] | "\(.number) \(.title)"'

# Filter by condition
gh pr list --json number,title,labels --jq '.[] | select(.labels | any(.name == "bug"))'

# Count items
gh issue list --json number --jq 'length'
```

---

## Multi-Step Workflow Templates

### Create Release from PRs

```bash
# 1. List merged PRs since last release
LAST_TAG=$(gh release list --limit 1 --json tagName --jq '.[0].tagName')
gh pr list --state merged --search "merged:>=$(gh release view $LAST_TAG --json createdAt --jq '.createdAt')" --json number,title --jq '.[] | "- \(.title) (#\(.number))"'

# 2. Create release with auto-generated notes
gh release create v1.3.0 --generate-notes --target main
```

### PR Review Workflow

```bash
# 1. List PRs needing your review
gh pr list --search "review-requested:@me"

# 2. Checkout and review
gh pr checkout 42
# ... review code ...

# 3. Approve or request changes
gh pr review 42 --approve --body "LGTM — clean implementation"

# 4. Merge if you're the last reviewer
gh pr merge 42 --squash --delete-branch
```

### Deploy Workflow

```bash
# 1. Trigger deployment
gh workflow run deploy.yml --field environment=staging --field image_tag=v1.2.0

# 2. Watch the run
gh run watch

# 3. Check deployment status
gh run list --workflow deploy.yml --limit 1 --json status,conclusion --jq '.[0]'
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Using `gh pr merge` without `--squash` or `--rebase` | Messy git history with merge commits | Choose a consistent merge strategy |
| Not using `--delete-branch` with merge | Stale branches accumulate | Always add `--delete-branch` |
| Creating releases without `--generate-notes` | Manual notes are incomplete | Let GitHub auto-generate, then edit |
| Using `gh api` without `--jq` | Huge JSON output, hard to parse | Always filter with `--jq` |
| Not setting `--auto` for PR merges | Manually watching for CI to pass | Use `gh pr merge --auto` to merge when ready |
| Hardcoding PR numbers in scripts | Breaks when numbers change | Use `gh pr list --json` + `--jq` to find dynamically |
| Not using `gh run watch` for deployments | Deploying blind, no feedback | Watch runs to catch failures immediately |

---

## Related Skills

When a task involves more than CLI operations, load these:

- **Git branching or commit operations?** → `skill({ name: "git-commands" })` for branching strategies and conventional commits
- **Setting up CI/CD workflows?** → `skill({ name: "devops-pipeline-architect" })` for pipeline design
- **Creating release pipelines?** → `skill({ name: "build-optimizer" })` for conditional execution on tags
- **Security scanning before release?** → `skill({ name: "security-scanner" })` for pre-release security checks
