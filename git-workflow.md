# Git Workflow and Release Management: Beginner to Expert

A comprehensive guide covering Git workflows, branching strategies, and release management practices for developers at all levels.

---

## Table of Contents

1. [Git Basics (Beginner)](#1-git-basics-beginner)
2. [Branching and Merging (Intermediate)](#2-branching-and-merging-intermediate)
3. [Collaboration Workflows (Intermediate)](#3-collaboration-workflows-intermediate)
4. [Advanced Git Techniques (Advanced)](#4-advanced-git-techniques-advanced)
5. [Release Management (Intermediate to Advanced)](#5-release-management-intermediate-to-advanced)
6. [CI/CD Integration (Advanced)](#6-cicd-integration-advanced)
7. [Best Practices and Conventions (Expert)](#7-best-practices-and-conventions-expert)
8. [Troubleshooting Common Issues](#8-troubleshooting-common-issues)

---

## 1. Git Basics (Beginner)

### 1.1 What is Git?

Git is a distributed version control system that tracks changes in your codebase over time. Unlike centralized systems, every developer has a complete copy of the repository history.

**Key Concepts:**
- **Repository (Repo)**: A directory containing your project files and Git's tracking data
- **Commit**: A snapshot of your project at a specific point in time
- **Branch**: A parallel version of your code
- **Remote**: A version of your repository hosted on a server (e.g., GitHub, GitLab)

### 1.2 Essential Git Commands

#### Initial Setup
```bash
# Configure your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Initialize a new repository
git init

# Clone an existing repository
git clone <repository-url>
```

#### Basic Workflow
```bash
# Check status of your working directory
git status

# Add files to staging area
git add <file>              # Add specific file
git add .                   # Add all changes
git add -p                  # Add interactively (choose hunks)

# Commit changes
git commit -m "Commit message"
git commit -am "Add and commit tracked files"

# View commit history
git log
git log --oneline           # Compact view
git log --graph --all       # Visual branch structure

# View changes
git diff                    # Unstaged changes
git diff --staged           # Staged changes
git diff HEAD               # All changes since last commit
```

#### Working with Remotes
```bash
# View remote repositories
git remote -v

# Add a remote
git remote add origin <url>

# Fetch changes from remote
git fetch origin

# Pull changes (fetch + merge)
git pull origin main

# Push changes to remote
git push origin main
git push -u origin main     # Set upstream for first push
```

### 1.3 Understanding the Three States

Git has three main states for your files:

1. **Working Directory**: Your actual files where you make changes
2. **Staging Area (Index)**: Files marked for the next commit
3. **Repository (.git directory)**: Committed snapshots

```
Working Directory → (git add) → Staging Area → (git commit) → Repository
```

### 1.4 The `.gitignore` File

Create a `.gitignore` file to exclude files from tracking:

```gitignore
# Dependencies
node_modules/
vendor/

# Environment files
.env
.env.local

# Build outputs
dist/
build/
*.log

# IDE files
.vscode/
.idea/
*.swp

# OS files
.DS_Store
Thumbs.db
```

---

## 2. Branching and Merging (Intermediate)

### 2.1 Branch Fundamentals

Branches allow you to develop features, fix bugs, or experiment without affecting the main codebase.

```bash
# Create a new branch
git branch feature-name

# Switch to a branch
git checkout feature-name

# Create and switch in one command
git checkout -b feature-name

# Modern alternative (Git 2.23+)
git switch feature-name
git switch -c feature-name

# List branches
git branch                  # Local branches
git branch -r               # Remote branches
git branch -a               # All branches

# Delete a branch
git branch -d feature-name  # Safe delete (merged only)
git branch -D feature-name  # Force delete
```

### 2.2 Merging Branches

#### Fast-Forward Merge
When no divergent commits exist on the target branch:

```bash
git checkout main
git merge feature-name
```

```
Before:
main:    A---B
              \
feature:       C---D

After:
main:    A---B---C---D
```

#### Three-Way Merge
When both branches have diverged:

```bash
git checkout main
git merge feature-name
```

```
Before:
main:    A---B---E
              \
feature:       C---D

After:
main:    A---B---E---M
              \     /
feature:       C---D
```

#### Merge with No Fast-Forward
Forces a merge commit even when fast-forward is possible:

```bash
git merge --no-ff feature-name
```

### 2.3 Rebasing

Rebase rewrites commit history by moving your branch to start from a different commit.

```bash
# Rebase current branch onto main
git checkout feature-name
git rebase main

# Or in one command
git rebase main feature-name
```

```
Before:
main:    A---B---E
              \
feature:       C---D

After:
main:    A---B---E
                  \
feature:           C'---D'
```

**When to Use Rebase:**
- ✅ Clean up local commits before pushing
- ✅ Keep a linear project history
- ✅ Update feature branch with latest main changes

**When NOT to Use Rebase:**
- ❌ On public/shared branches
- ❌ After pushing to remote (unless force-pushing is acceptable)

### 2.4 Merge vs Rebase Decision Guide

| Scenario | Recommendation |
|----------|---------------|
| Updating feature branch with main | Rebase (keeps history clean) |
| Integrating feature into main | Merge (preserves feature context) |
| Fixing mistakes in local commits | Rebase (interactive) |
| Collaborating on shared feature | Merge (safer) |

### 2.5 Handling Merge Conflicts

When Git can't automatically merge changes:

```bash
# After conflict occurs
git status                  # See conflicted files

# Edit conflicted files (look for conflict markers)
<<<<<<< HEAD
Your changes
=======
Incoming changes
>>>>>>> branch-name

# After resolving
git add <resolved-files>
git commit                  # For merge
git rebase --continue       # For rebase

# Abort if needed
git merge --abort
git rebase --abort
```

**Conflict Resolution Tools:**
```bash
# Use merge tool
git mergetool

# See both versions during conflict
git checkout --ours <file>      # Keep your version
git checkout --theirs <file>    # Keep incoming version
```

---

## 3. Collaboration Workflows (Intermediate)

### 3.1 Feature Branch Workflow

The simplest workflow for teams. Each feature gets its own branch.

**Process:**
1. Create feature branch from main
2. Develop and commit changes
3. Push to remote
4. Create pull request
5. Review and merge

```bash
# Developer A
git checkout main
git pull origin main
git checkout -b feature/user-authentication
# ... make changes ...
git add .
git commit -m "Add user authentication"
git push origin feature/user-authentication
# Create PR on GitHub/GitLab

# After approval and merge, cleanup
git checkout main
git pull origin main
git branch -d feature/user-authentication
```

**Advantages:**
- Simple and straightforward
- Isolates work in progress
- Easy to review and discuss changes

### 3.2 Gitflow Workflow

A robust workflow for projects with scheduled releases. Uses multiple long-lived branches.

**Branch Types:**

1. **`main`** (or `master`): Production-ready code
2. **`develop`**: Integration branch for features
3. **`feature/*`**: New features (branch from develop)
4. **`release/*`**: Release preparation (branch from develop)
5. **`hotfix/*`**: Urgent fixes (branch from main)

**Feature Development:**
```bash
# Start new feature
git checkout develop
git pull origin develop
git checkout -b feature/shopping-cart

# ... develop feature ...
git add .
git commit -m "Implement shopping cart"

# Finish feature
git checkout develop
git pull origin develop
git merge --no-ff feature/shopping-cart
git push origin develop
git branch -d feature/shopping-cart
```

**Release Process:**
```bash
# Start release
git checkout develop
git checkout -b release/1.2.0

# Bump version, update changelog, bug fixes only
# ... make changes ...
git commit -m "Bump version to 1.2.0"

# Finish release
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags

git checkout develop
git merge --no-ff release/1.2.0
git push origin develop

git branch -d release/1.2.0
```

**Hotfix Process:**
```bash
# Start hotfix
git checkout main
git checkout -b hotfix/1.2.1

# Fix critical bug
git add .
git commit -m "Fix critical security vulnerability"

# Finish hotfix
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix version 1.2.1"
git push origin main --tags

git checkout develop
git merge --no-ff hotfix/1.2.1
git push origin develop

git branch -d hotfix/1.2.1
```

**Gitflow Visualization:**
```
main:      •--------•-----------•---•  (tags: v1.0, v1.1, v1.2)
            \      / \         /   /
release:     \    /   \       /   /
              \  /     •-----•   /
               \/     /         /
develop:       •--•--•--•--•---•
                \  \ /  /  /
feature:         •--•  •--•
```

**Advantages:**
- Clear separation of concerns
- Supports multiple releases in parallel
- Emergency hotfixes don't interfere with development

**Disadvantages:**
- More complex
- Requires discipline
- Can be overkill for small projects

### 3.3 GitHub Flow

A simplified workflow optimized for continuous deployment.

**Rules:**
1. `main` branch is always deployable
2. Create descriptive branches from main
3. Commit and push regularly
4. Open pull request when ready
5. Deploy from pull request for testing
6. Merge after approval and successful deployment

```bash
# Create feature branch
git checkout main
git pull origin main
git checkout -b add-user-profile

# Develop and push frequently
git add .
git commit -m "Add user profile component"
git push origin add-user-profile

# Create PR, request review
# After approval, merge via UI or:
git checkout main
git merge add-user-profile
git push origin main

# Deploy main branch
# Delete feature branch
git branch -d add-user-profile
```

**Advantages:**
- Simple and lean
- Perfect for continuous deployment
- Fast feedback loop

**Best For:**
- Web applications with continuous deployment
- Small to medium teams
- Projects that deploy to production multiple times per day

### 3.4 GitLab Flow

Combines feature-driven development with issue tracking and environment branches.

**Structure:**
- **Feature branches** → `main` → **Environment branches** (`staging`, `production`)

```bash
# Feature development
git checkout -b 42-add-search-feature  # Include issue number

# ... develop ...
git push origin 42-add-search-feature
# Create merge request

# After merge to main, deploy to staging
git checkout staging
git merge main
git push origin staging

# After testing, deploy to production
git checkout production
git merge staging
git push origin production
```

**With Release Branches:**
```
feature → main → release/1.x → production
```

**Advantages:**
- Issue tracking integration
- Environment-based deployment
- Upstream first philosophy (fixes go to earliest branch)

### 3.5 Trunk-Based Development

Developers collaborate on a single branch (`main` or `trunk`) with very short-lived feature branches.

**Key Principles:**
- Commit directly to main or use very short-lived branches (< 1 day)
- Use feature flags for incomplete features
- Require comprehensive automated testing
- Deploy from main continuously

```bash
# Short-lived branch approach
git checkout -b quick-fix
# ... make small change ...
git commit -m "Fix validation bug"
git push origin quick-fix
# Create PR immediately, merge within hours

# Direct to main (advanced teams only)
git checkout main
git pull origin main
# ... make small change ...
git commit -m "Update error message"
git push origin main
```

**Feature Flags Example:**
```javascript
// Code with feature flag
if (featureFlags.isEnabled('new-checkout-flow')) {
  return <NewCheckout />;
} else {
  return <OldCheckout />;
}
```

**Advantages:**
- Reduces merge conflicts
- Enables continuous integration
- Faster feedback and deployment

**Requirements:**
- Strong automated testing
- Feature flag system
- High team discipline
- Good code review practices

### 3.6 Workflow Comparison

| Workflow | Complexity | Team Size | Release Frequency | CI/CD |
|----------|-----------|-----------|-------------------|-------|
| Feature Branch | Low | Any | Any | Optional |
| Gitflow | High | Medium-Large | Scheduled | Staged |
| GitHub Flow | Low | Small-Medium | Continuous | Required |
| GitLab Flow | Medium | Any | Continuous | Required |
| Trunk-Based | Medium | Any | Continuous | Required |

---

## 4. Advanced Git Techniques (Advanced)

### 4.1 Interactive Rebase

Clean up commit history before merging:

```bash
# Rebase last 3 commits
git rebase -i HEAD~3

# Rebase since branching from main
git rebase -i main
```

**Interactive Rebase Commands:**
```
pick 1fc6c95 First commit
squash 6b2481b Second commit (will be squashed into first)
reword dd1475d Third commit (edit message)
edit fa39187 Fourth commit (pause to amend)
drop 3a4a2f3 Fifth commit (remove)
```

**Common Use Cases:**

**Squash Multiple Commits:**
```bash
git rebase -i HEAD~3
# Change 'pick' to 'squash' for commits to combine
```

**Reorder Commits:**
```bash
git rebase -i HEAD~5
# Simply reorder the lines
```

**Edit a Past Commit:**
```bash
git rebase -i HEAD~3
# Change 'pick' to 'edit'
# Make changes
git add .
git commit --amend
git rebase --continue
```

### 4.2 Cherry-Picking

Apply specific commits from one branch to another:

```bash
# Apply a single commit
git cherry-pick <commit-hash>

# Apply multiple commits
git cherry-pick <hash1> <hash2>

# Apply a range of commits
git cherry-pick <start-hash>^..<end-hash>

# Cherry-pick without committing (stage changes only)
git cherry-pick -n <commit-hash>
```

**Use Cases:**
- Applying hotfixes to multiple branches
- Backporting features to older releases
- Selectively merging commits

**Example: Backporting a Bug Fix:**
```bash
# Bug fixed in develop (commit abc123)
git checkout release/1.5
git cherry-pick abc123
git push origin release/1.5
```

### 4.3 Git Reflog

Reflog records all changes to branch tips, even deleted commits:

```bash
# View reflog
git reflog

# View reflog for specific branch
git reflog show main

# Recover deleted commits or branches
git reflog
# Find the commit before deletion
git checkout -b recovered-branch <commit-hash>

# Undo a bad rebase
git reflog
git reset --hard <commit-before-rebase>
```

**Example Output:**
```
abc1234 HEAD@{0}: commit: Add new feature
def5678 HEAD@{1}: rebase: fast-forward
9ab0cde HEAD@{2}: checkout: moving from main to feature
```

### 4.4 Git Stash

Temporarily save uncommitted changes:

```bash
# Stash changes
git stash
git stash save "Work in progress on feature X"

# List stashes
git stash list

# Apply stash
git stash apply              # Keep stash in list
git stash pop                # Apply and remove from list
git stash apply stash@{2}    # Apply specific stash

# View stash contents
git stash show
git stash show -p            # Show diff

# Create branch from stash
git stash branch new-feature-branch

# Delete stashes
git stash drop stash@{0}
git stash clear              # Remove all stashes
```

**Advanced Stash Options:**
```bash
# Stash including untracked files
git stash -u

# Stash including ignored files
git stash -a

# Interactive stash (choose what to stash)
git stash -p
```

### 4.5 Git Bisect

Binary search to find which commit introduced a bug:

```bash
# Start bisect
git bisect start

# Mark current commit as bad
git bisect bad

# Mark a known good commit
git bisect good <commit-hash>

# Git will checkout middle commit
# Test and mark as good or bad
git bisect good   # Bug doesn't exist here
git bisect bad    # Bug exists here

# Repeat until found
# Git will tell you the first bad commit

# End bisect
git bisect reset
```

**Automated Bisect:**
```bash
# Run a script to test each commit
git bisect start HEAD <good-commit>
git bisect run npm test

# Git will automatically find the bad commit
```

### 4.6 Git Hooks

Automate tasks at specific Git events:

**Common Hooks:**
- `pre-commit`: Run before commit is created
- `commit-msg`: Validate commit message
- `pre-push`: Run before push
- `post-merge`: Run after merge

**Example: Pre-commit Hook** (`.git/hooks/pre-commit`):
```bash
#!/bin/sh
# Run linter before commit
npm run lint
if [ $? -ne 0 ]; then
    echo "Linting failed. Commit aborted."
    exit 1
fi
```

**Example: Commit Message Hook** (`.git/hooks/commit-msg`):
```bash
#!/bin/sh
# Enforce commit message format
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{10,}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "Invalid commit message format"
    echo "Format: type(scope): description"
    echo "Example: feat(auth): add user login"
    exit 1
fi
```

**Managing Hooks with Husky (Node.js):**
```bash
# Install husky
npm install --save-dev husky

# Initialize
npx husky install

# Add pre-commit hook
npx husky add .husky/pre-commit "npm test"
```

### 4.7 Git Submodules

Include other Git repositories within your repository:

```bash
# Add submodule
git submodule add <repository-url> <path>

# Clone repository with submodules
git clone --recursive <repository-url>

# Initialize submodules in existing clone
git submodule init
git submodule update

# Update submodules to latest
git submodule update --remote

# Execute command in all submodules
git submodule foreach 'git pull origin main'
```

**Alternative: Git Subtree** (simpler, no .gitmodules):
```bash
# Add subtree
git subtree add --prefix=vendor/library <repository-url> main --squash

# Update subtree
git subtree pull --prefix=vendor/library <repository-url> main --squash

# Push changes to subtree
git subtree push --prefix=vendor/library <repository-url> main
```

### 4.8 Git Worktree

Work on multiple branches simultaneously:

```bash
# List worktrees
git worktree list

# Create new worktree
git worktree add ../myproject-feature2 feature-2

# You now have two working directories:
# ./myproject (main branch)
# ./myproject-feature2 (feature-2 branch)

# Remove worktree
git worktree remove ../myproject-feature2

# Prune stale worktree references
git worktree prune
```

**Use Cases:**
- Run tests on one branch while developing on another
- Quick hotfixes without stashing current work
- Compare implementations side-by-side

### 4.9 Advanced Searching

Find code across history:

```bash
# Search for string in tracked files
git grep "searchterm"

# Search with line numbers
git grep -n "searchterm"

# Search in specific commit
git grep "searchterm" <commit-hash>

# Search commit messages
git log --grep="feature"

# Search commit content (code changes)
git log -S "functionName" --source --all

# Find when a file was deleted
git log --diff-filter=D --summary | grep <filename>

# Find commits that changed specific lines
git log -L :functionName:path/to/file.js
git log -L 100,150:path/to/file.js
```

### 4.10 Rewriting History

**Change Author Information:**
```bash
# Change author of last commit
git commit --amend --author="New Name <email@example.com>"

# Change author of multiple commits
git rebase -i HEAD~3
# Mark commits as 'edit'
git commit --amend --author="New Name <email@example.com>"
git rebase --continue
```

**Remove Sensitive Data:**
```bash
# Remove file from entire history
git filter-branch --tree-filter 'rm -f passwords.txt' HEAD

# Modern alternative using git-filter-repo
pip install git-filter-repo
git filter-repo --path passwords.txt --invert-paths

# Remove sensitive data using BFG Repo-Cleaner
java -jar bfg.jar --delete-files passwords.txt
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

---

## 5. Release Management (Intermediate to Advanced)

### 5.1 Semantic Versioning (SemVer)

Version format: **MAJOR.MINOR.PATCH** (e.g., 2.4.1)

**Increment:**
- **MAJOR**: Breaking changes (incompatible API changes)
- **MINOR**: New features (backward-compatible)
- **PATCH**: Bug fixes (backward-compatible)

**Pre-release versions:**
- `1.0.0-alpha` - Early testing
- `1.0.0-beta.1` - Feature complete, testing
- `1.0.0-rc.1` - Release candidate

**Examples:**
```
1.0.0     → First stable release
1.0.1     → Patch (bug fix)
1.1.0     → Minor (new feature)
2.0.0     → Major (breaking change)
2.0.0-rc.1 → Release candidate
```

**In package.json:**
```json
{
  "version": "1.2.3",
  "dependencies": {
    "express": "^4.18.0",    // Allow minor and patch updates
    "lodash": "~4.17.21",    // Allow patch updates only
    "react": "18.2.0"        // Exact version
  }
}
```

### 5.2 Git Tags

Tags mark specific points in history (usually releases):

**Lightweight Tags:**
```bash
git tag v1.0.0
```

**Annotated Tags (recommended for releases):**
```bash
# Create annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag a specific commit
git tag -a v1.0.0 <commit-hash> -m "Release version 1.0.0"

# List tags
git tag
git tag -l "v1.*"

# View tag information
git show v1.0.0

# Push tags to remote
git push origin v1.0.0
git push origin --tags        # Push all tags

# Delete tag
git tag -d v1.0.0            # Local
git push origin :refs/tags/v1.0.0  # Remote
git push origin --delete v1.0.0    # Remote (modern syntax)
```

**Checkout Tag:**
```bash
# Create branch from tag
git checkout -b version-1.0 v1.0.0
```

### 5.3 Release Strategies

#### 5.3.1 Rolling Release
Continuous deployment with no formal releases.

**Characteristics:**
- Deploy from main branch continuously
- Version based on timestamp or commit hash
- Common in SaaS applications

```bash
# Version based on timestamp
VERSION=$(date +%Y%m%d.%H%M%S)
docker build -t myapp:$VERSION .
```

#### 5.3.2 Scheduled Releases
Fixed release schedule (e.g., monthly, quarterly).

**Example: Monthly Release Cycle**
```bash
# Week 1-3: Development
git checkout develop
git pull origin develop
git checkout -b feature/new-feature

# Week 4: Code freeze and testing
git checkout -b release/2024.11
# Bug fixes only

# Week 4 end: Release
git checkout main
git merge release/2024.11
git tag -a v2024.11.0 -m "November 2024 Release"
git push origin main --tags
```

#### 5.3.3 Feature-Based Releases
Release when features are complete.

**Process:**
1. Develop features on separate branches
2. Merge when ready
3. Release after significant features complete

#### 5.3.4 LTS (Long-Term Support)
Maintain multiple versions simultaneously.

**Example Structure:**
```
main (v3.x)
release/2.x-lts (supported until 2025)
release/1.x-lts (supported until 2024)
```

**Backporting Fixes:**
```bash
# Fix in main
git checkout main
git commit -m "fix: security vulnerability"

# Backport to LTS branches
git checkout release/2.x-lts
git cherry-pick <commit-hash>
git push origin release/2.x-lts

git checkout release/1.x-lts
git cherry-pick <commit-hash>
git push origin release/1.x-lts
```

### 5.4 Changelog Management

#### Manual Changelog (CHANGELOG.md)
```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New feature for user profiles

### Changed
- Improved error handling in API

## [1.2.0] - 2024-10-15

### Added
- User authentication with OAuth2
- Password reset functionality
- Email verification

### Changed
- Updated dependencies
- Improved database query performance

### Fixed
- Fixed XSS vulnerability in comments
- Fixed memory leak in WebSocket connection

### Security
- Updated OpenSSL to patch CVE-2024-XXXX

## [1.1.0] - 2024-09-01

### Added
- Shopping cart functionality
- Payment gateway integration

### Deprecated
- Old API endpoints (will be removed in v2.0.0)

## [1.0.0] - 2024-08-01

### Added
- Initial stable release
- User management
- Product catalog
- Basic search functionality
```

#### Automated Changelog with Conventional Commits

**Using standard-version:**
```bash
# Install
npm install --save-dev standard-version

# Add to package.json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major"
  }
}

# Create release (automatically bumps version, updates CHANGELOG, creates tag)
npm run release
```

**Using semantic-release:**
```bash
# Install
npm install --save-dev semantic-release

# Configure .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/git",
    "@semantic-release/github"
  ]
}
```

### 5.5 Hotfix Process

Critical fixes that need immediate deployment:

**Gitflow-style Hotfix:**
```bash
# Create hotfix from production
git checkout main
git checkout -b hotfix/1.2.1

# Fix the issue
git add .
git commit -m "fix: critical security vulnerability"

# Merge to main and tag
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix: Security patch"
git push origin main --tags

# Merge back to develop
git checkout develop
git merge --no-ff hotfix/1.2.1
git push origin develop

# Cleanup
git branch -d hotfix/1.2.1
```

**GitHub Flow Hotfix:**
```bash
# Create hotfix branch
git checkout main
git pull origin main
git checkout -b hotfix-security-patch

# Fix and push
git commit -m "fix: patch security vulnerability"
git push origin hotfix-security-patch

# Create PR, get expedited review
# Merge and deploy immediately
# Tag after deployment
git checkout main
git pull origin main
git tag -a v1.2.1 -m "Emergency security patch"
git push origin --tags
```

### 5.6 Release Checklist Template

```markdown
## Pre-Release Checklist

- [ ] All features for milestone are merged
- [ ] All tests passing
- [ ] No critical bugs
- [ ] Documentation updated
- [ ] CHANGELOG.md updated
- [ ] Version bumped in package.json/pom.xml/etc.
- [ ] Migration scripts tested
- [ ] Security scan passed
- [ ] Performance tests completed

## Release Process

- [ ] Create release branch
- [ ] Final testing in staging environment
- [ ] Create tag with version number
- [ ] Build release artifacts
- [ ] Deploy to production
- [ ] Smoke tests in production
- [ ] Create GitHub/GitLab release with notes
- [ ] Announce release (email, Slack, blog)
- [ ] Monitor error tracking for issues

## Post-Release

- [ ] Merge release branch back to develop
- [ ] Close related issues/tickets
- [ ] Update project board
- [ ] Schedule retrospective
- [ ] Archive release documentation
```

### 5.7 Version Bumping Scripts

**Bash Script:**
```bash
#!/bin/bash
# bump-version.sh

CURRENT_VERSION=$(git describe --tags --abbrev=0 | sed 's/v//')
IFS='.' read -ra VERSION_PARTS <<< "$CURRENT_VERSION"

MAJOR=${VERSION_PARTS[0]}
MINOR=${VERSION_PARTS[1]}
PATCH=${VERSION_PARTS[2]}

case $1 in
  major)
    MAJOR=$((MAJOR + 1))
    MINOR=0
    PATCH=0
    ;;
  minor)
    MINOR=$((MINOR + 1))
    PATCH=0
    ;;
  patch)
    PATCH=$((PATCH + 1))
    ;;
  *)
    echo "Usage: $0 {major|minor|patch}"
    exit 1
    ;;
esac

NEW_VERSION="$MAJOR.$MINOR.$PATCH"
echo "Bumping version to $NEW_VERSION"

# Update package.json
sed -i "s/\"version\": \".*\"/\"version\": \"$NEW_VERSION\"/" package.json

# Commit and tag
git add package.json
git commit -m "chore: bump version to $NEW_VERSION"
git tag -a "v$NEW_VERSION" -m "Release version $NEW_VERSION"

echo "Version bumped to $NEW_VERSION"
echo "Don't forget to: git push origin main --tags"
```

**Node.js Script:**
```javascript
// bump-version.js
const fs = require('fs');
const { execSync } = require('child_process');

const bumpType = process.argv[2]; // major, minor, patch

if (!['major', 'minor', 'patch'].includes(bumpType)) {
  console.error('Usage: node bump-version.js [major|minor|patch]');
  process.exit(1);
}

const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf8'));
const [major, minor, patch] = packageJson.version.split('.').map(Number);

let newVersion;
switch (bumpType) {
  case 'major':
    newVersion = `${major + 1}.0.0`;
    break;
  case 'minor':
    newVersion = `${major}.${minor + 1}.0`;
    break;
  case 'patch':
    newVersion = `${major}.${minor}.${patch + 1}`;
    break;
}

packageJson.version = newVersion;
fs.writeFileSync('package.json', JSON.stringify(packageJson, null, 2) + '\n');

execSync(`git add package.json`);
execSync(`git commit -m "chore: bump version to ${newVersion}"`);
execSync(`git tag -a v${newVersion} -m "Release version ${newVersion}"`);

console.log(`✅ Version bumped to ${newVersion}`);
console.log(`   Don't forget to: git push origin main --tags`);
```

---

## 6. CI/CD Integration (Advanced)

### 6.1 Automated Release Pipeline

#### GitHub Actions Example

**.github/workflows/release.yml:**
```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Needed for conventional-changelog
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      - name: Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

#### GitLab CI Example

**.gitlab-ci.yml:**
```yaml
stages:
  - test
  - build
  - release

variables:
  GIT_DEPTH: 0

test:
  stage: test
  image: node:18
  script:
    - npm ci
    - npm test
  only:
    - merge_requests
    - main

build:
  stage: build
  image: node:18
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
  only:
    - main

release:
  stage: release
  image: node:18
  before_script:
    - npm ci
  script:
    - npx semantic-release
  only:
    - main
```

### 6.2 Automated Versioning with Conventional Commits

**Commit Message Format:**
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature (bumps MINOR version)
- `fix`: Bug fix (bumps PATCH version)
- `docs`: Documentation only
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `test`: Adding tests
- `chore`: Maintenance tasks
- `ci`: CI/CD changes

**Breaking Changes** (bumps MAJOR version):
```
feat(api): change authentication method

BREAKING CHANGE: The authentication method has changed from session-based to JWT tokens.
Clients must update their authentication logic.
```

**Examples:**
```bash
# Patch release (1.2.3 → 1.2.4)
git commit -m "fix(auth): resolve token expiration issue"

# Minor release (1.2.3 → 1.3.0)
git commit -m "feat(api): add user profile endpoints"

# Major release (1.2.3 → 2.0.0)
git commit -m "feat(api): redesign REST API

BREAKING CHANGE: All endpoints now require /v2/ prefix"
```

### 6.3 Automated Changelog Generation

**Using conventional-changelog:**
```bash
# Install
npm install --save-dev conventional-changelog-cli

# Generate changelog
npx conventional-changelog -p angular -i CHANGELOG.md -s

# Add to package.json
{
  "scripts": {
    "changelog": "conventional-changelog -p angular -i CHANGELOG.md -s"
  }
}
```

**Configuration (.versionrc.json):**
```json
{
  "types": [
    { "type": "feat", "section": "Features" },
    { "type": "fix", "section": "Bug Fixes" },
    { "type": "chore", "hidden": true },
    { "type": "docs", "section": "Documentation" },
    { "type": "style", "hidden": true },
    { "type": "refactor", "section": "Code Refactoring" },
    { "type": "perf", "section": "Performance Improvements" },
    { "type": "test", "hidden": true }
  ],
  "releaseCommitMessageFormat": "chore(release): {{currentTag}}",
  "skip": {
    "bump": false,
    "changelog": false,
    "commit": false,
    "tag": false
  }
}
```

### 6.4 Multi-Environment Deployment

**Branch-based Deployment Strategy:**
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - develop
      - staging
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Determine environment
        id: env
        run: |
          if [ "${{ github.ref }}" = "refs/heads/main" ]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [ "${{ github.ref }}" = "refs/heads/staging" ]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
          fi
      
      - name: Deploy to ${{ steps.env.outputs.environment }}
        run: |
          echo "Deploying to ${{ steps.env.outputs.environment }}"
          # Your deployment script here
```

**Tag-based Deployment:**
```yaml
name: Deploy Release

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Get version
        id: version
        run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Build Docker image
        run: |
          docker build -t myapp:${{ steps.version.outputs.version }} .
          docker tag myapp:${{ steps.version.outputs.version }} myapp:latest
      
      - name: Push to registry
        run: |
          docker push myapp:${{ steps.version.outputs.version }}
          docker push myapp:latest
      
      - name: Deploy to production
        run: |
          # Deployment commands
```

### 6.5 Rollback Strategies

**Quick Rollback Using Tags:**
```bash
# List recent tags
git tag -l --sort=-version:refname | head -5

# Deploy previous version
git checkout v1.2.0
./deploy.sh

# Or revert main branch
git checkout main
git revert <commit-hash>
git push origin main
```

**Docker Rollback:**
```bash
# List available versions
docker images myapp

# Deploy previous version
docker pull myapp:1.2.0
docker stop myapp-container
docker rm myapp-container
docker run -d --name myapp-container myapp:1.2.0
```

**Kubernetes Rollback:**
```bash
# Rollback deployment
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# View rollout history
kubectl rollout history deployment/myapp
```

### 6.6 Release Notifications

**Slack Notification Example:**
```yaml
# .github/workflows/release.yml
- name: Notify Slack
  if: success()
  uses: 8398a7/action-slack@v3
  with:
    status: custom
    custom_payload: |
      {
        text: "🚀 New release deployed!",
        attachments: [{
          color: 'good',
          text: `Version ${{ steps.version.outputs.version }} has been deployed to production.
          Release Notes: https://github.com/${{ github.repository }}/releases/tag/${{ github.ref }}`
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**Email Notification Script:**
```bash
#!/bin/bash
# notify-release.sh

VERSION=$1
CHANGELOG=$(git log --oneline --pretty=format:"- %s" v$VERSION^..v$VERSION)

mail -s "New Release: v$VERSION" team@example.com <<EOF
A new version has been released: v$VERSION

Changes:
$CHANGELOG

View release: https://github.com/yourorg/yourrepo/releases/tag/v$VERSION
EOF
```

---

## 7. Best Practices and Conventions (Expert)

### 7.1 Commit Message Conventions

**Conventional Commits Standard:**
```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Good Commit Messages:**
```
✅ feat(auth): add OAuth2 authentication
✅ fix(api): handle null response in user endpoint
✅ docs(readme): update installation instructions
✅ refactor(database): optimize query performance
✅ test(user): add unit tests for user service
```

**Bad Commit Messages:**
```
❌ fixed stuff
❌ updates
❌ WIP
❌ asdfasdf
❌ final changes (no really this time)
```

**Commit Message Template:**

Create `.gitmessage` file:
```
# <type>(<scope>): <subject> (Max 50 characters)

# <body> (Wrap at 72 characters)

# <footer>

# Type must be one of:
# feat:     New feature
# fix:      Bug fix
# docs:     Documentation
# style:    Formatting
# refactor: Code restructuring
# perf:     Performance improvement
# test:     Tests
# chore:    Maintenance
# ci:       CI/CD changes
#
# Example:
# feat(api): add user registration endpoint
#
# Implement POST /api/users endpoint with validation
# and email verification.
#
# Closes #123
```

Configure Git to use it:
```bash
git config --global commit.template ~/.gitmessage
```

### 7.2 Branch Naming Conventions

**Recommended Format:**
```
<type>/<ticket-number>-<short-description>
```

**Examples:**
```
✅ feature/AUTH-123-oauth-integration
✅ bugfix/STORE-456-cart-calculation
✅ hotfix/PROD-789-payment-gateway
✅ release/1.5.0
✅ chore/update-dependencies
```

**Patterns to Avoid:**
```
❌ new-feature
❌ johns-branch
❌ fix
❌ test-branch-2
```

**Branch Naming by Workflow:**

**Gitflow:**
```
feature/feature-name
release/1.5.0
hotfix/1.4.1
```

**GitHub Flow:**
```
add-user-authentication
fix-checkout-bug
update-readme
```

**With Issue Tracking:**
```
feature/JIRA-123-user-profile
fix/GH-456-memory-leak
docs/DOC-789-api-guide
```

### 7.3 Pull Request Best Practices

**PR Title Format:**
```
[TYPE] Short description (#issue-number)
```

**Examples:**
```
[FEATURE] Add user authentication (#123)
[FIX] Resolve memory leak in WebSocket connection (#456)
[REFACTOR] Restructure API endpoints (#789)
```

**PR Description Template:**
```markdown
## Description
Brief summary of changes

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Related Issues
Closes #123
Related to #456

## Changes Made
- Added user authentication with JWT
- Implemented password reset flow
- Updated API documentation

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing completed

### Test Instructions
1. Start the development server
2. Navigate to /login
3. Enter credentials: test@example.com / password123
4. Verify successful login

## Screenshots (if applicable)
[Add screenshots here]

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] Tests pass locally
- [ ] Dependent changes merged
```

**Code Review Guidelines:**

**For Authors:**
1. Keep PRs small (< 400 lines ideally)
2. Self-review before submitting
3. Provide context in description
4. Respond to feedback promptly
5. Don't force-push after review starts
6. Mark conversations as resolved

**For Reviewers:**
1. Review within 24 hours
2. Be constructive and specific
3. Ask questions rather than make demands
4. Praise good solutions
5. Test the changes locally if needed
6. Use suggestion feature for small fixes

**Review Comments Format:**
```markdown
**Issue:** The error handling here could be more robust

**Suggestion:**
```javascript
try {
  await fetchUser();
} catch (error) {
  logger.error('Failed to fetch user:', error);
  throw new UserFetchError('Unable to retrieve user data');
}
```

**Rationale:** This provides better error context and uses custom error types
```

### 7.4 Branch Protection Rules

**Recommended Settings:**

**For `main` branch:**
- ✅ Require pull request reviews (minimum 2)
- ✅ Require status checks to pass
- ✅ Require branches to be up to date
- ✅ Require conversation resolution
- ✅ Require signed commits (optional but recommended)
- ✅ Include administrators
- ✅ Restrict force pushes
- ✅ Restrict deletions

**For `develop` branch:**
- ✅ Require pull request reviews (minimum 1)
- ✅ Require status checks to pass
- ✅ Allow force pushes (for maintainers only)

**GitHub Settings:**
```yaml
# .github/settings.yml (using probot/settings)
branches:
  - name: main
    protection:
      required_pull_request_reviews:
        required_approving_review_count: 2
        dismiss_stale_reviews: true
        require_code_owner_reviews: true
      required_status_checks:
        strict: true
        contexts:
          - ci/tests
          - ci/lint
          - ci/security-scan
      enforce_admins: true
      restrictions: null
```

### 7.5 Code Owners (CODEOWNERS)

Define who must review specific parts of the codebase:

**.github/CODEOWNERS:**
```
# Global owners
* @org/core-team

# Frontend
/frontend/ @org/frontend-team
/frontend/components/ @org/ui-team

# Backend
/backend/ @org/backend-team
/backend/api/ @org/api-team

# Infrastructure
/infrastructure/ @org/devops-team
/.github/workflows/ @org/devops-team
/docker/ @org/devops-team

# Documentation
/docs/ @org/tech-writers
*.md @org/tech-writers

# Security-sensitive
/auth/ @org/security-team
/payment/ @org/security-team @org/compliance-team

# Database
/migrations/ @org/database-team
```

### 7.6 Security Best Practices

#### Signed Commits

**Setup GPG:**
```bash
# Generate GPG key
gpg --full-generate-key

# List keys
gpg --list-secret-keys --keyid-format LONG

# Get key ID (after sec rsa4096/)
# Example: 3AA5C34371567BD2

# Configure Git
git config --global user.signingkey 3AA5C34371567BD2
git config --global commit.gpgsign true

# Export public key for GitHub
gpg --armor --export 3AA5C34371567BD2
# Add to GitHub → Settings → SSH and GPG keys
```

**Sign Commits:**
```bash
# Automatically signed (if gpgsign=true)
git commit -m "feat: add new feature"

# Manual signing
git commit -S -m "feat: add new feature"

# Sign tags
git tag -s v1.0.0 -m "Release version 1.0.0"
```

#### Sensitive Data Protection

**.gitignore for secrets:**
```gitignore
# Environment variables
.env
.env.local
.env.*.local

# Credentials
credentials.json
secrets.yml
*.pem
*.key
*.p12

# IDE configurations (may contain paths)
.vscode/settings.json

# Database dumps
*.sql
*.dump
```

**Pre-commit Hook for Secret Detection:**
```bash
#!/bin/bash
# .git/hooks/pre-commit

# Check for common secret patterns
if git diff --cached | grep -E "(password|api_key|secret_key|private_key)" | grep -v "^-"; then
    echo "⚠️  Warning: Possible secret detected!"
    echo "Please review your changes before committing."
    exit 1
fi
```

**Use git-secrets:**
```bash
# Install
brew install git-secrets  # macOS
apt-get install git-secrets  # Linux

# Setup
git secrets --install
git secrets --register-aws

# Scan repository
git secrets --scan
git secrets --scan-history
```

#### Dependency Security

**Automated Dependency Scanning:**
```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
  schedule:
    - cron: '0 0 * * 1'  # Weekly

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Check for outdated packages
        run: npm outdated
```

### 7.7 Performance Optimization

#### Shallow Clones for CI/CD

```bash
# Shallow clone (faster)
git clone --depth 1 <repository-url>

# Shallow clone with specific branch
git clone --depth 1 --branch main <repository-url>
```

**In CI/CD:**
```yaml
- uses: actions/checkout@v3
  with:
    fetch-depth: 1  # Shallow clone
```

#### Git LFS for Large Files

```bash
# Install Git LFS
git lfs install

# Track large files
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "assets/**/*.bin"

# Add .gitattributes
git add .gitattributes

# Commit and push
git add .
git commit -m "Add large files with LFS"
git push origin main
```

**.gitattributes:**
```
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
```

#### Repository Maintenance

```bash
# Garbage collection
git gc --aggressive --prune=now

# Verify repository integrity
git fsck

# Remove unreachable objects
git prune

# Optimize repository
git repack -Ad
git gc --aggressive

# Show repository size
git count-objects -vH
```

### 7.8 Monorepo Management

**Strategies for Large Repositories:**

#### Git Sparse-Checkout

```bash
# Enable sparse-checkout
git sparse-checkout init --cone

# Specify directories to checkout
git sparse-checkout set frontend/ shared/

# List current sparse-checkout
git sparse-checkout list
```

#### Git Worktree for Monorepos

```bash
# Work on multiple packages simultaneously
git worktree add ../myproject-backend backend/
git worktree add ../myproject-frontend frontend/
```

#### Submodules for Microservices

```bash
# Add microservices as submodules
git submodule add https://github.com/org/auth-service services/auth
git submodule add https://github.com/org/payment-service services/payment

# Update all submodules
git submodule update --remote --recursive
```

---

## 8. Troubleshooting Common Issues

### 8.1 Undoing Changes

**Undo uncommitted changes:**
```bash
# Discard changes in working directory
git checkout -- <file>
git restore <file>  # Modern alternative

# Discard all uncommitted changes
git reset --hard HEAD

# Unstage files
git reset HEAD <file>
git restore --staged <file>  # Modern alternative
```

**Undo commits:**
```bash
# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, keep changes unstaged
git reset HEAD~1
git reset --mixed HEAD~1  # Same as above

# Undo last commit, discard changes
git reset --hard HEAD~1

# Undo last commit with new commit (safe for public branches)
git revert HEAD
```

**Fix last commit:**
```bash
# Amend commit message
git commit --amend -m "New message"

# Add files to last commit
git add forgotten-file.js
git commit --amend --no-edit

# Change author of last commit
git commit --amend --author="Name <email@example.com>"
```

### 8.2 Recovering Lost Commits

```bash
# Find lost commits
git reflog

# Recover deleted branch
git reflog | grep branch-name
git checkout -b recovered-branch <commit-hash>

# Recover after hard reset
git reflog
git reset --hard <commit-before-reset>

# Recover deleted commits
git fsck --lost-found
```

### 8.3 Resolving "Detached HEAD"

```bash
# You're in detached HEAD state
git status
# HEAD detached at abc1234

# Option 1: Create branch from current state
git checkout -b new-branch-name

# Option 2: Return to branch
git checkout main

# Option 3: Discard changes and return
git checkout main
# If you have commits, save them first:
git branch temp-branch
git checkout main
```

### 8.4 Fixing Merge Conflicts

```bash
# During merge conflict
git status  # See conflicted files

# Edit files to resolve conflicts
# Remove conflict markers: <<<<<<<, =======, >>>>>>>

# After resolving
git add <resolved-files>
git commit  # For merge
git rebase --continue  # For rebase

# Abort merge/rebase
git merge --abort
git rebase --abort

# Use theirs/ours strategy
git checkout --ours <file>    # Keep your version
git checkout --theirs <file>  # Keep their version
git add <file>
```

**Merge Tools:**
```bash
# Configure merge tool
git config --global merge.tool vimdiff
git config --global merge.tool meld
git config --global merge.tool vscode

# Use merge tool
git mergetool
```

### 8.5 Large File Issues

**Remove large file from history:**
```bash
# Using git filter-branch (old method)
git filter-branch --tree-filter 'rm -f large-file.zip' HEAD

# Using git-filter-repo (recommended)
pip install git-filter-repo
git filter-repo --path large-file.zip --invert-paths

# Using BFG Repo-Cleaner (fastest)
java -jar bfg.jar --delete-files large-file.zip
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force push after cleaning
git push origin --force --all
git push origin --force --tags
```

### 8.6 Permission Denied (SSH)

```bash
# Test SSH connection
ssh -T git@github.com

# Generate new SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"

# Add to SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Add public key to GitHub/GitLab
cat ~/.ssh/id_ed25519.pub
# Copy and paste into GitHub → Settings → SSH Keys

# Use SSH URL instead of HTTPS
git remote set-url origin git@github.com:username/repo.git
```

### 8.7 Diverged Branches

```bash
# Error: Your branch and 'origin/main' have diverged

# Option 1: Rebase (rewrites history)
git pull --rebase origin main

# Option 2: Merge
git pull origin main

# Option 3: Force your version (dangerous!)
git push --force origin main
# Or safer:
git push --force-with-lease origin main
```

### 8.8 Repository Corruption

```bash
# Check repository integrity
git fsck --full

# Recover corrupted repository
cd .git
rm -f index
git reset HEAD

# Clone fresh copy if severely corrupted
cd ..
mv old-repo old-repo-backup
git clone <repository-url>
```

### 8.9 Accidentally Committed to Wrong Branch

```bash
# Committed to main instead of feature branch

# Method 1: Move commit to new branch
git branch feature-branch  # Create branch at current commit
git reset --hard HEAD~1    # Remove commit from current branch
git checkout feature-branch

# Method 2: Cherry-pick to correct branch
git checkout feature-branch
git cherry-pick main       # Copy commit
git checkout main
git reset --hard HEAD~1    # Remove from main
```

### 8.10 Pull Request Conflicts

```bash
# Update feature branch with main
git checkout feature-branch
git fetch origin
git rebase origin/main

# Or merge
git merge origin/main

# Resolve conflicts
# ... edit files ...
git add .
git rebase --continue  # For rebase
# Or
git commit             # For merge

# Force push after rebase (if already pushed)
git push --force-with-lease origin feature-branch
```

---

## Summary: Choosing the Right Workflow

### Quick Decision Tree

```
Do you deploy continuously?
├─ Yes
│  ├─ Small team (<10) → GitHub Flow
│  └─ Large team → Trunk-Based Development
└─ No
   ├─ Scheduled releases → Gitflow
   └─ Flexible releases → Feature Branch Workflow
```

### Workflow Recommendations by Team Size

| Team Size | Recommended Workflow |
|-----------|---------------------|
| Solo | Feature Branch or simple branching |
| 2-5 | GitHub Flow |
| 5-15 | GitHub Flow or GitLab Flow |
| 15-50 | GitLab Flow or Gitflow |
| 50+ | Trunk-Based or Monorepo strategy |

### Release Strategy by Project Type

| Project Type | Release Strategy | Versioning |
|--------------|-----------------|------------|
| SaaS Web App | Continuous | Timestamp |
| Mobile App | Scheduled (2-4 weeks) | SemVer |
| Library/SDK | Feature-based | SemVer |
| Enterprise Software | Scheduled + LTS | SemVer with LTS |
| Open Source | Feature-based | SemVer |

---

## Additional Resources

### Learning Resources
- [Pro Git Book](https://git-scm.com/book/en/v2) - Comprehensive Git documentation
- [Learn Git Branching](https://learngitbranching.js.org/) - Interactive tutorial
- [Git Workflows Compared](https://www.atlassian.com/git/tutorials/comparing-workflows)

### Tools
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Release](https://semantic-release.gitbook.io/)
- [Git Flow](https://github.com/nvie/gitflow)
- [Husky](https://typicode.github.io/husky/) - Git hooks
- [Commitlint](https://commitlint.js.org/) - Lint commit messages

### VS Code Extensions
- GitLens
- Git Graph
- Git History
- Conventional Commits
- GitLab Workflow

---

## Glossary

- **Branch**: A parallel version of a repository
- **Commit**: A snapshot of changes
- **Merge**: Combining branches
- **Rebase**: Reapplying commits on top of another branch
- **Tag**: A reference to a specific commit (usually a release)
- **Remote**: A repository hosted on a server
- **Origin**: Default name for remote repository
- **HEAD**: Pointer to current commit
- **Detached HEAD**: HEAD pointing to commit instead of branch
- **Staging Area**: Intermediate area before committing
- **Cherry-pick**: Applying specific commits to another branch
- **Stash**: Temporarily saving uncommitted changes
- **Reflog**: Log of all reference updates
- **Bisect**: Binary search to find bug-introducing commit

---

**Last Updated:** October 2024  
**Version:** 1.0.0

---

*This guide is maintained as a living document. Contributions and suggestions are welcome!*

