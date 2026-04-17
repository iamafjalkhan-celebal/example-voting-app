---
name: git-commands
description: Git operations reference — branching strategies, conventional commits, rebasing, recovery, tagging, and anti-patterns. Use when the user needs git commands for version control workflows.
---

# Git Commands

Expert reference for **git operations** in DevOps workflows. Covers branching, committing, history navigation, recovery, and collaboration patterns.

## When to Use This Skill

- User needs to create branches, merge, or rebase
- User asks about commit conventions or history
- User needs to recover from a mistake (wrong branch, bad merge, lost commits)
- User wants to manage tags or releases via git
- User asks "how do I undo..." anything in git

---

## Branching Strategies

### Feature Branch Workflow

```bash
# Create feature branch from main
git checkout main
git pull origin main
git checkout -b feat/add-auth

# Work, commit, push
git add .
git commit -m "feat(auth): add JWT middleware"
git push -u origin feat/add-auth

# After PR is merged, clean up
git checkout main
git pull origin main
git branch -d feat/add-auth
git push origin --delete feat/add-auth
```

### Release Branch Workflow

```bash
# Create release branch
git checkout main
git checkout -b release/v1.2.0

# Stabilize — only bug fixes allowed
git commit -m "fix(api): handle null response"

# Merge to main and tag
git checkout main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin main --tags

# Clean up
git branch -d release/v1.2.0
```

### Hotfix Workflow

```bash
# Branch from the tag, not main
git checkout -b hotfix/v1.2.1 v1.2.0

# Fix the issue
git commit -m "fix(critical): patch SQL injection vulnerability"

# Merge to main AND develop
git checkout main
git merge --no-ff hotfix/v1.2.1
git tag -a v1.2.1 -m "Hotfix v1.2.1"

git checkout develop
git merge --no-ff hotfix/v1.2.1

git push origin main develop --tags
git branch -d hotfix/v1.2.1
```

---

## Conventional Commits

### Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | Purpose | Example |
|------|---------|---------|
| `feat` | New feature | `feat(api): add user search endpoint` |
| `fix` | Bug fix | `fix(auth): handle expired tokens` |
| `docs` | Documentation | `docs(readme): update setup instructions` |
| `style` | Formatting (no logic change) | `style: fix indentation in config` |
| `refactor` | Code restructure (no behavior change) | `refactor(db): extract query builder` |
| `perf` | Performance improvement | `perf(api): add response caching` |
| `test` | Tests | `test(auth): add JWT validation tests` |
| `build` | Build system / deps | `build(deps): upgrade express to v5` |
| `ci` | CI config | `ci: add CodeQL security scan` |
| `chore` | Maintenance | `chore: update .gitignore` |

### Breaking Changes

```bash
# Footer notation
git commit -m "feat(api)!: change auth endpoint to /v2/auth

BREAKING CHANGE: The /auth endpoint is removed. Use /v2/auth instead."
```

### Multi-line Commits

```bash
git commit -m "feat(dashboard): add real-time metrics

- Add WebSocket connection for live data
- Implement chart rendering with D3
- Add auto-refresh toggle

Closes #142"
```

---

## History & Inspection

### Log Formats

```bash
# Compact one-line log
git log --oneline -20

# Graph view (branch topology)
git log --oneline --graph --all --decorate -30

# Log with file changes
git log --stat -10

# Log for a specific file
git log --follow -p -- src/server.js

# Search commit messages
git log --grep="fix" --oneline

# Commits by author in date range
git log --author="khan" --since="2 weeks ago" --oneline

# Commits between two refs
git log v1.0.0..v1.1.0 --oneline
```

### Diff

```bash
# Unstaged changes
git diff

# Staged changes
git diff --staged

# Between branches
git diff main..feat/new-feature

# Between commits
git diff abc123..def456

# Specific file
git diff main -- src/config.js

# Stats only (no content)
git diff --stat main..HEAD

# Name-only (just filenames)
git diff --name-only main..HEAD
```

### Blame & Show

```bash
# Who changed each line
git blame src/server.js

# Blame a specific line range
git blame -L 10,20 src/server.js

# Show a specific commit
git show abc123

# Show a file at a specific commit
git show abc123:src/server.js
```

---

## Staging & Stashing

### Selective Staging

```bash
# Stage specific files
git add src/server.js src/config.js

# Interactive staging (choose hunks)
git add -p

# Stage all tracked files (not new files)
git add -u

# Stage everything
git add -A
```

### Stash

```bash
# Stash current changes
git stash

# Stash with a message
git stash push -m "WIP: auth refactor"

# Stash including untracked files
git stash push -u -m "WIP: includes new files"

# List stashes
git stash list

# Apply most recent stash (keep in stash list)
git stash apply

# Apply and remove from stash list
git stash pop

# Apply a specific stash
git stash apply stash@{2}

# Drop a stash
git stash drop stash@{0}

# Clear all stashes
git stash clear
```

---

## Remote Operations

### Push

```bash
# Push and set upstream
git push -u origin feat/my-branch

# Push tags
git push origin --tags

# Push a specific tag
git push origin v1.0.0

# Force push (ONLY for your own feature branches)
git push --force-with-lease origin feat/my-branch

# Delete remote branch
git push origin --delete feat/old-branch
```

### Pull & Fetch

```bash
# Fetch all remotes (no merge)
git fetch --all --prune

# Pull with rebase (cleaner history)
git pull --rebase origin main

# Pull with merge (default)
git pull origin main
```

### `--force-with-lease` vs `--force`

| Flag | Behavior | Safety |
|------|----------|--------|
| `--force` | Overwrites remote unconditionally | ❌ Dangerous — destroys others' work |
| `--force-with-lease` | Overwrites only if remote hasn't changed since your last fetch | ✅ Safe — aborts if someone else pushed |

**Rule: Never use `--force`. Always use `--force-with-lease`.**

---

## Recovery Playbook

### Undo Last Commit (Keep Changes)

```bash
# Soft reset — uncommit but keep changes staged
git reset --soft HEAD~1

# Mixed reset — uncommit and unstage, keep in working tree
git reset HEAD~1
```

### Undo Last Commit (Discard Changes)

```bash
# Hard reset — destroy the commit and changes
git reset --hard HEAD~1
```

### Revert a Pushed Commit (Safe for Shared Branches)

```bash
# Create a new commit that undoes the changes
git revert abc123

# Revert a merge commit (specify parent)
git revert -m 1 abc123
```

### Recover Lost Commits

```bash
# Find the lost commit in reflog
git reflog

# Restore to that point
git reset --hard HEAD@{3}

# Or cherry-pick just that commit
git cherry-pick abc123
```

### Fix Wrong Branch

```bash
# Committed to main instead of feature branch — move it
git checkout -b feat/correct-branch    # branch from current (with the commit)
git checkout main
git reset --hard HEAD~1                # remove from main
```

### Amend Last Commit

```bash
# Fix the message
git commit --amend -m "fix(auth): correct token expiry check"

# Add forgotten files to last commit
git add forgotten-file.js
git commit --amend --no-edit
```

---

## Rebase vs Merge

### Decision Tree

```
Is this a shared/protected branch (main, develop)?
├─ YES → Use MERGE (preserves history, safe for teams)
│        git merge --no-ff feat/my-branch
└─ NO  → Is the branch behind main?
         ├─ YES → Use REBASE (clean linear history)
         │        git rebase main
         └─ NO  → Just merge or fast-forward
```

### Interactive Rebase

```bash
# Rebase last 3 commits (squash, reorder, edit)
git rebase -i HEAD~3

# Rebase onto main
git rebase -i main
```

**Interactive rebase commands:**

| Command | Effect |
|---------|--------|
| `pick` | Keep commit as-is |
| `reword` | Keep commit, edit message |
| `squash` | Merge into previous commit, combine messages |
| `fixup` | Merge into previous commit, discard message |
| `drop` | Delete the commit |
| `edit` | Pause to amend the commit |

---

## Tags

```bash
# Create annotated tag (recommended)
git tag -a v1.0.0 -m "Release v1.0.0"

# Create tag from a specific commit
git tag -a v1.0.0 abc123 -m "Release v1.0.0"

# List tags
git tag -l

# List tags matching pattern
git tag -l "v1.*"

# Push all tags
git push origin --tags

# Push a specific tag
git push origin v1.0.0

# Delete local tag
git tag -d v1.0.0

# Delete remote tag
git push origin --delete v1.0.0
```

---

## Configuration

```bash
# Set identity
git config user.name "Your Name"
git config user.email "you@example.com"

# Global config
git config --global core.autocrlf input    # Linux/Mac
git config --global pull.rebase true       # Default to rebase on pull
git config --global init.defaultBranch main

# Useful aliases
git config --global alias.st "status -sb"
git config --global alias.lg "log --oneline --graph --all --decorate -20"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD"
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| `git push --force` on shared branches | Destroys others' work | Use `--force-with-lease` on feature branches only |
| Committing directly to main | No review, no CI gate | Use feature branches + PRs |
| Giant commits ("fix everything") | Impossible to review or revert | Small, focused commits with conventional prefixes |
| No commit message convention | Unreadable history, no auto-changelog | Use conventional commits |
| `git add .` without checking | Commits unintended files | Use `git add -p` or specific files |
| Rebasing shared branches | Rewrites history others depend on | Only rebase your own feature branches |
| Not pruning after merge | Stale branches accumulate | `git fetch --prune` and delete merged branches |
| Using `reset --hard` without reflog backup | Permanent data loss | Check `git reflog` first |

---

## Related Skills

When a task involves more than just git commands, load these:

- **Creating a release?** → `skill({ name: "gh-cli" })` for `gh release create`
- **Setting up CI triggers for branches?** → `skill({ name: "devops-pipeline-architect" })` for workflow `on:` configuration
- **Managing deployment branches?** → `skill({ name: "build-optimizer" })` for conditional execution patterns
