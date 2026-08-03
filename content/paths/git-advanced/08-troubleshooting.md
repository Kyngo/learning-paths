---
title: "Troubleshooting"
weight: 8
---

# Git Troubleshooting

Every Git user eventually encounters situations that seem catastrophic — lost commits, corrupted history, massive repositories, impossible merge conflicts. This chapter provides systematic solutions for the most common problems.

---

## Common Problems and Solutions

### Undo Last Commit (Keep Changes)

```bash
# Undo commit but keep changes staged
git reset --soft HEAD~1

# Undo commit and unstage changes
git reset --mixed HEAD~1    # default behavior of git reset HEAD~1

# Undo commit and DISCARD changes (destructive!)
git reset --hard HEAD~1
```

### Recover Deleted Branch

```bash
# Find the commit the branch pointed to
git reflog | grep "branch-name"
# or
git reflog --all | grep "branch-name"

# Recreate the branch
git branch recovered-branch abc1234
```

### Accidentally Committed to Wrong Branch

```bash
# Move last commit to correct branch
git branch correct-branch    # create branch at current commit
git reset --hard HEAD~1      # remove commit from current branch
git checkout correct-branch  # switch to correct branch
```

### Amend Last Commit

```bash
# Change message only
git commit --amend -m "corrected message"

# Add forgotten files to last commit
git add forgotten-file.txt
git commit --amend --no-edit
```

> **Warning:** Never amend commits that have been pushed to a shared branch.

### Revert a Pushed Commit

```bash
# Create a new commit that undoes the changes (safe for shared branches)
git revert abc1234

# Revert a merge commit (specify which parent to keep)
git revert -m 1 merge-commit-sha
```

### Find Which Commit Introduced a Bug

```bash
# Binary search through history
git bisect start
git bisect bad                  # current commit is bad
git bisect good v1.0.0          # this tag was good

# Git checks out middle commit — test it, then:
git bisect good   # or
git bisect bad

# Repeat until Git identifies the first bad commit
git bisect reset  # return to original state

# Automated bisect with a test script
git bisect start HEAD v1.0.0
git bisect run ./test-script.sh
```

### Recover from Detached HEAD

```bash
# You made commits in detached HEAD and want to keep them
git branch save-my-work    # create branch at current position
git checkout main          # return to main
git merge save-my-work     # merge if desired
```

---

## Merge Conflict Resolution Strategies

### Understanding Conflict Markers

```
<<<<<<< HEAD (your changes)
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}
======= (separator)
function calculateTotal(items, taxRate) {
  return items.reduce((sum, item) => sum + item.price * (1 + taxRate), 0);
}
>>>>>>> feature/add-tax (their changes)
```

### Resolution Approaches

| Strategy | When to Use |
|----------|-------------|
| **Accept ours** | Your version is definitely correct |
| **Accept theirs** | Their version is definitely correct |
| **Manual merge** | Both sides have valid changes to combine |
| **Re-implement** | Both versions are wrong; write fresh |

```bash
# Accept all of ours (for a specific file)
git checkout --ours path/to/file.txt

# Accept all of theirs
git checkout --theirs path/to/file.txt

# Use a merge tool
git mergetool

# Abort the merge entirely
git merge --abort
```

### Reducing Merge Conflicts

| Practice | Effect |
|----------|--------|
| Short-lived branches | Less time to diverge |
| Small, focused PRs | Fewer files touched per change |
| Frequent rebasing | Stay up to date with main |
| Clear ownership (CODEOWNERS) | Fewer people editing same files |
| Modular code | Changes isolated to modules |

### Rerere — Reuse Recorded Resolution

```bash
# Enable rerere (reuse recorded resolution)
git config --global rerere.enabled true

# Git remembers how you resolved conflicts
# Next time the same conflict appears, it auto-resolves
git rerere status    # show recorded resolutions
git rerere diff      # show what was recorded
```

---

## Large File Handling (Git LFS)

Git is designed for text files. Binary files (images, videos, datasets, models) bloat the repository because every version is stored in full.

### Setup Git LFS

```bash
# Install
brew install git-lfs   # macOS
git lfs install        # initialize in your user config

# Track file patterns
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "datasets/**"

# This creates/updates .gitattributes
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.zip filter=lfs diff=lfs merge=lfs -text
```

### How It Works

```
Regular Git:          Git LFS:
┌─────────────┐       ┌─────────────┐
│ .git/objects │       │ .git/objects │ (pointer files only)
│  file v1    │       │  pointer v1 │──→ LFS Server: actual file v1
│  file v2    │       │  pointer v2 │──→ LFS Server: actual file v2
│  file v3    │       │  pointer v3 │──→ LFS Server: actual file v3
└─────────────┘       └─────────────┘
    300 MB                 3 KB            300 MB (on LFS server)
```

### Common Operations

```bash
# Check what's tracked
git lfs ls-files

# See LFS storage usage
git lfs env

# Fetch LFS files (if not auto-fetched)
git lfs pull

# Migrate existing files to LFS (rewrites history!)
git lfs migrate import --include="*.psd" --everything
```

### When to Use LFS

| File Type | Use LFS? | Reason |
|-----------|----------|--------|
| Source code | No | Git handles text efficiently |
| Images (PNG, PSD) | Yes | Binary, many versions |
| Video/audio | Yes | Large binary files |
| ML models | Yes | Large, versioned binaries |
| Compiled artifacts | No (use artifact storage) | Don't belong in Git at all |
| Dependencies (node_modules) | No (use .gitignore) | Managed by package managers |

---

## Repository Cleanup (git-filter-repo)

When you need to rewrite history — remove secrets, large files, or restructure:

### Remove a File from All History

```bash
# Install
pip install git-filter-repo

# Remove a file from entire history
git filter-repo --invert-paths --path secrets.env

# Remove a directory
git filter-repo --invert-paths --path vendor/
```

### Remove Large Files

```bash
# Find large objects
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sort -k3 -n -r | head -20

# Remove files larger than 50MB from history
git filter-repo --strip-blobs-bigger-than 50M
```

### Migrate to LFS Retroactively

```bash
# Convert existing large files in history to LFS pointers
git lfs migrate import --include="*.bin" --everything

# Force push (required after history rewrite)
git push --force --all
git push --force --tags
```

> **Warning:** History rewrites require all team members to re-clone or reset. Coordinate carefully.

---

## Performance Optimization

### Diagnosing Slow Git

```bash
# Measure command timing
GIT_TRACE=1 git status
GIT_TRACE_PERFORMANCE=1 git log --oneline -10

# Check repo size and object count
git count-objects -vH

# Check for large pack files
du -sh .git/objects/pack/
```

### Optimization Techniques

| Problem | Solution | Command |
|---------|----------|---------|
| Many loose objects | Pack objects | `git gc` |
| Large pack files | Repack aggressively | `git repack -a -d --depth=250 --window=250` |
| Slow status | Update index | `git update-index --untracked-cache` |
| Large working tree | Filesystem monitor | `git config core.fsmonitor true` |
| Slow clone | Shallow clone | `git clone --depth 1` |
| Only need one branch | Single-branch clone | `git clone --single-branch --branch main` |
| Partial checkout needed | Sparse checkout | `git sparse-checkout set src/` |

### Commit Graph

```bash
# Generate commit-graph file for faster log/merge-base operations
git commit-graph write --reachable

# Enable auto-generation
git config --global fetch.writeCommitGraph true
```

---

## Garbage Collection and Maintenance

### What `git gc` Does

1. Packs loose objects into pack files
2. Removes unreachable objects (after grace period)
3. Compresses pack files
4. Updates auxiliary data structures

```bash
# Standard GC (safe, removes objects older than 2 weeks)
git gc

# Aggressive GC (slower but better compression)
git gc --aggressive

# Prune unreachable objects immediately (dangerous!)
git gc --prune=now
```

### Scheduled Maintenance (Git 2.30+)

```bash
# Enable background maintenance
git maintenance start

# This schedules:
# - Hourly:  prefetch (fetch from remotes)
# - Hourly:  commit-graph update
# - Daily:   loose-objects cleanup
# - Weekly:  incremental repack
```

### Reflog Expiration

```bash
# Reflogs keep deleted/amended commits recoverable
git reflog expire --expire=90.days.ago --all

# Check reflog for a branch
git reflog show main

# Reflogs are your safety net — don't expire them too aggressively
```

### Health Check

```bash
# Verify repository integrity
git fsck --full

# Check for corruption
git fsck --connectivity-only  # faster, checks reachability only
```

---

## Emergency Recovery

### Repository Corruption

```bash
# Fetch missing objects from remote
git fetch --all

# If pack file is corrupt
mv .git/objects/pack/corrupt.pack /tmp/
git unpack-objects < /tmp/corrupt.pack  # may recover some objects
git fetch origin  # re-fetch what's missing

# Nuclear option: re-clone
git clone <remote-url> fresh-clone
```

### Lost Stash

```bash
# Stashes are just commits — find them in reflog
git fsck --unreachable | grep commit
git show <sha>   # inspect each to find your stash

# Or check stash reflog
git reflog show stash
```

---

## Quick Reference

| Situation | Command |
|-----------|---------|
| Undo last commit (keep changes) | `git reset --soft HEAD~1` |
| Recover deleted branch | `git reflog` → `git branch name sha` |
| Find bug-introducing commit | `git bisect start/good/bad` |
| Resolve all conflicts as ours | `git checkout --ours .` |
| Remove file from history | `git filter-repo --invert-paths --path file` |
| Track large files | `git lfs track "*.bin"` |
| Speed up large repo | `git maintenance start` |
| Verify repo integrity | `git fsck --full` |
| See what's using space | `git count-objects -vH` |

---

## Key Takeaways

1. **Reflog is your safety net** — almost nothing in Git is truly lost if you act within the reflog expiry window.
2. Use `git bisect` to find bugs efficiently — it's logarithmic search through history.
3. **Git LFS** is essential for binary files — never commit large binaries directly.
4. **git-filter-repo** replaces the deprecated `filter-branch` — use it for history rewrites.
5. Enable `git maintenance` for large repositories — background optimization prevents slowdowns.
6. Merge conflicts are reduced by short-lived branches, small PRs, and clear code ownership.
7. Enable `rerere` to automatically resolve recurring conflicts — invaluable during long rebases.
8. Prevention beats recovery: `.gitignore` secrets, use LFS from the start, keep branches short.
