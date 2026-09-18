# Git & GitHub for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Branching, Committing, and Merging**

```bash
# Clone a repository
git clone https://github.com/org/repo.git

# Create and switch to a new feature branch
git checkout -b feature/new-pipeline

# Stage changes
git add .
# OR stage specific files
git add scripts/deploy.sh

# Commit with a descriptive message
git commit -m "feat(ci): add terraform deployment script"

# Push branch to remote
git push -u origin feature/new-pipeline

# Switch back to main and merge
git checkout main
git pull origin main
git merge feature/new-pipeline
```

---

## 2. Intermediate Examples

**Rebase, Cherry-Pick, and Stash**

```bash
# 1. Rebase (Updating feature branch with latest main without creating merge commits)
git checkout feature/new-api
git fetch origin
git rebase origin/main
# If conflicts occur, fix them, then:
git add .
git rebase --continue
# Force push is required if you had already pushed the feature branch
git push --force origin feature/new-api

# 2. Cherry-Pick (Copying a specific commit from one branch to another)
git checkout hotfix-branch
git cherry-pick a1b2c3d4 # The commit hash

# 3. Stash (Save uncommitted work temporarily to switch branches)
git stash save "WIP: writing tests"
git checkout main # Do some quick work
git checkout feature-branch
git stash pop # Bring the changes back
```

---

## 3. Advanced Examples

**Reset, Revert, and Git Hooks**

```bash
# 1. Resetting (Altering history - DANGEROUS on shared branches)
# Undo the last commit but keep the files staged
git reset --soft HEAD~1

# Undo the last commit and wipe out the changes completely
git reset --hard HEAD~1

# 2. Reverting (Safe - creates a new commit that undoes a previous one)
git revert a1b2c3d4

# 3. Git Hooks (Client-side automation)
# Create a pre-commit hook to prevent committing directly to main
cat << 'EOF' > .git/hooks/pre-commit
#!/bin/bash
branch="$(git rev-parse --abbrev-ref HEAD)"
if [ "$branch" = "main" ]; then
  echo "You can't commit directly to main branch"
  exit 1
fi
EOF
chmod +x .git/hooks/pre-commit
```

---

## 4. Interview Coding Exercises

### Problem 1: Conflict Resolution
**Task:** You are trying to merge `featureA` into `main`, but Git says: `Automatic merge failed; fix conflicts and then commit the result.` How do you resolve this via CLI?

**Solution:**
1. Open the files marked as conflicted. Look for conflict markers:
```text
<<<<<<< HEAD
code from main
=======
code from featureA
>>>>>>> featureA
```
2. Edit the file to keep the desired code, and remove the `<<<<`, `====`, `>>>>` markers.
3. Run `git add <file>` to mark it as resolved.
4. Run `git commit` to finalize the merge.

### Problem 2: Recovering a "deleted" commit
**Task:** You accidentally ran `git reset --hard HEAD~3` and lost 3 commits. You haven't pushed them yet. How do you get them back?

**Solution:**
```bash
# View all recent actions, even deleted ones
git reflog
# Find the hash of the commit before the reset (e.g., HEAD@{1} -> a1b2c3d)
# Reset back to that specific commit
git reset --hard a1b2c3d
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Cannot Push
**Scenario:** You run `git push origin main` and get: `Updates were rejected because the tip of your current branch is behind its remote counterpart.`
**What is wrong?**
**Answer:** Someone else pushed changes to `main` while you were working.
**Fix:** Run `git pull --rebase origin main` to apply their changes, place your commits on top, and then run `git push origin main`.

### Broken Configuration 2: Accidentally Committed a Secret
**Scenario:** You committed an AWS access key and pushed it to GitHub.
**Fix:** 
1. **Immediate action:** Revoke/rotate the key in AWS immediately. (Git history is immutable on the remote if anyone has fetched it).
2. To remove it from the repo history, use tools like `git filter-repo` or `BFG Repo-Cleaner`.
3. Force push the clean history.

---

## 6. Common Interview Questions

**Q: "What is the difference between `git merge` and `git rebase`?"**
*Answer:* `merge` takes two branches and joins them with a new "merge commit", preserving the exact history and chronological order. `rebase` takes your branch's commits, moves them to the tip of the target branch, and reapplies them. Rebase creates a perfectly linear, clean history but rewrites commit SHAs, which is dangerous on public/shared branches.

**Q: "What is a detached HEAD state?"**
*Answer:* It happens when you checkout a specific commit hash or tag instead of a branch name. You can view and edit files, but any new commits you make will be orphaned and lost when you switch branches unless you create a new branch from that detached state (`git checkout -b new-branch-name`).

**Q: "Explain GitHub branch protection rules."**
*Answer:* Branch protection rules enforce workflows on specific branches (like `main`). Common rules include: requiring pull request reviews before merging, requiring status checks (CI pipeline success) to pass before merging, preventing force pushes, and preventing branch deletion.

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `git commit --amend -m "new msg"` | Fix the message of the very last commit (if unpushed). |
| `git fetch --all --prune` | Fetch latest from all remotes and delete local branches tracking deleted remote branches. |
| `git tag -a v1.0.0 -m "Release"` | Create an annotated release tag. |
| `git push origin --tags` | Push tags to GitHub (normal push doesn't send tags). |
| `git log --oneline --graph` | View history in a clean, visual CLI tree. |
| `git config --global core.editor vim`| Set default Git editor. |
