# Git Fundamentals

## What is it?

Git is a **version control system** — a tool that tracks every change you make to your files and lets you revisit any previous version. Think of it like an unlimited undo history, but shared across every developer on your team and baked into every AI tool you'll use.

When you save a file in a word processor, you overwrite the previous version. Git never overwrites. Every "commit" you make is a snapshot of your project at that exact moment, permanently stored and retrievable.

The core vocabulary you need:

- **Repository (repo):** A folder that Git is tracking. Contains your project files plus a hidden `.git` folder where all the history lives.
- **Commit:** A named snapshot of your files at a point in time. The fundamental unit of history.
- **Branch:** A separate line of development. It diverges from the main line, lets you work independently, and later merges back in.
- **HEAD:** Your current location. The commit you're standing on right now.

## Why it matters for vibe coding

Vibe coding — using AI to generate code on your behalf — creates specific, predictable failure modes that Git directly addresses.

**Without Git, you are defenseless against these AI failure modes:**

- **AI overwrites work it didn't know existed.** You ask AI to "refactor" a function. It doesn't see the other file where that function was already implemented differently. Without version control, both versions compete for the same file. Git lets you see exactly what changed and recover cleanly.
- **AI generates plausible but broken code.** You push AI output directly to production. Something breaks. Without Git history, you have no way to pinpoint which AI output caused it. With Git, `git bisect` narrows it to a specific commit.
- **AI suggests `git push --force`.** This overwrites the remote history and destroys work. Beginners don't know why this is catastrophic. Git history is your safety net — force-push is rarely the answer.
- **AI doesn't understand your `.gitignore`.** It confidently adds generated files, cache folders, and secrets to version control. Without understanding Git's staging model, you can't fix it quickly.
- **AI hallucinates branch names and commit hashes.** It says "just check out the `feat-magic` branch" when no such branch exists. Knowing how branches actually work lets you detect these hallucinations.

## The 20% you need to know

### The three areas

Git organizes every file into one of three states:

1. **Working directory:** The files you see and edit right now.
2. **Staging area (index):** A holding zone for changes you want to commit. Think of it as a pre-flight checklist.
3. **Repository (.git):** The committed history. The permanent record.

The flow is always: edit files in working directory -> `add` to staging -> `commit` to repository.

### Core commands you'll use daily

**Starting and saving work:**

```bash
git init                        # Turn any folder into a Git repo
git add filename.txt            # Stage a specific file
git add .                       # Stage all changed files
git commit -m "_descriptive message here"
git status                      # See what's changed, staged, or untracked
git log --oneline               # View commit history (one line per commit)
```

**Branching and navigation:**

```bash
git branch                      # List all branches (current branch marked with *)
git branch feature-name         # Create a new branch
git checkout feature-name       # Switch to that branch
git checkout -b feature-name    # Create AND switch in one step
git merge feature-name          # Merge feature-name into your current branch
```

**Syncing with remote (GitHub, GitLab, etc.):**

```bash
git remote -v                   # See what remote repos are connected
git push                        # Upload your commits to the remote
git pull                        # Download remote changes and merge them
git clone URL                   # Copy a remote repo to your local machine
```

### Pull requests explained

A Pull Request (PR) is not a Git command — it's a feature of platforms like GitHub and GitLab that wraps Git's `merge` functionality in a code review workflow.

The typical flow:
1. Push your branch to the remote.
2. Open a PR on GitHub/GitLab.
3. Team members review, comment, request changes.
4. You push more commits to the same branch — the PR updates automatically.
5. After approval, someone merges the branch into `main`.

PRs are the checkpointing mechanism for AI workflow. Before every major AI session, create a branch and commit your current state. When AI produces output, commit it before the next AI call. You now have recoverable checkpoints.

### The .gitignore file

This file tells Git which files to never track. Every project needs one.

```
node_modules/
__pycache__/
.env
*.log
.DS_Store
dist/
build/
```

Common mistake: adding `.gitignore` after files are already tracked. Git doesn't retroactively ignore files already in the repository. You must first untrack them.

### Recovering from mistakes: the safety net

```bash
git diff                         # Show unstaged changes (what you've edited but not yet added)
git diff --staged                # Show staged changes (what you've added but not yet committed)
git restore filename.txt         # Discard unstaged changes to a file (undo edits)
git restore --staged filename.txt # Unstage a file (remove from staging area, keep edits)
git checkout -- filename.txt     # Old syntax for the same as above (still works, git restore is preferred)
```

These commands are **non-destructive for committed history**. They only affect your working directory and staging area.

### Stash: the temporary shelf

When you need to switch branches but have uncommitted work you don't want to commit yet:

```bash
git stash              # Save uncommitted changes temporarily
git stash pop          # Bring them back
```

### Rebase vs. merge

Two ways to combine branches:

- **Merge:** Creates a new commit that joins two histories. Preserves the full branch topology. Safer, more history.
- **Rebase:** Replays your commits on top of another branch as if you started there. Produces a linear history. More powerful but can rewrite shared history if applied incorrectly.

Default advice: use merge for combining branches into `main`. Use rebase for keeping your feature branch up to date with `main` during development.

Never rebase commits that have been pushed and shared with others.

### Cherry-pick and bisect: surgical tools

```bash
git cherry-pick abc1234   # Apply the commit with hash abc1234 to your current branch
git bisect start          # Start a binary search to find which commit introduced a bug
git bisect bad            # Mark the current commit as bad
git bisect good abc1234   # Mark a known-good commit
# Git checks out commits automatically — test each one and mark good or bad
git bisect reset          # End the bisect session
```

`git bisect` is invaluable when AI introduces a regression. Binary search cuts a week of debugging into ~10 tests.

## Hands-on exercise

**Goal:** Create a Git repository, make commits, create a branch, merge it, and use `git bisect` to find a "bug." This takes 15 minutes.

### Setup

Open a terminal in any folder (not an existing Git repo). Run:

```bash
mkdir git-practice && cd git-practice
git init
echo "Project started" > readme.txt
git add readme.txt
git commit -m "Initial commit"
```

### Part 1: Branch and merge

```bash
git checkout -b add-feature        # Create and switch to a new branch
echo "Feature A" > feature.txt
git add feature.txt
git commit -m "Add feature A"
git checkout main                  # Switch back to main
git merge add-feature              # Merge the feature branch into main
```

Verify the merge: `git log --oneline` should show both commits.

### Part 2: Use git bisect to find a bug

This simulates a real scenario: you have a series of commits and need to find which one introduced a problem.

```bash
# Set up the scenario
git checkout -b bug-demo
echo "Line 1: Works" > status.txt
git add status.txt && git commit -m "Commit 1: Initial"

echo "Line 2: Works" >> status.txt
git add status.txt && git commit -m "Commit 2: Add line 2"

echo "Line 3: BROKEN" >> status.txt
git add status.txt && git commit -m "Commit 3: Add line 3"

echo "Line 4: Still broken" >> status.txt
git add status.txt && git commit -m "Commit 4: Add line 4"

# Mark the current (broken) state as bad
git bisect start
git bisect bad

# Mark the first commit as good
git log --oneline   # Find the hash of "Commit 1: Initial" and use it below
git bisect good <hash-of-commit-1>

# Git will automatically check out a commit midway.
# Check if status.txt contains "BROKEN" at that commit:
cat status.txt

# If BROKEN is NOT present: git bisect good
# If BROKEN IS present: git bisect bad
# Repeat until Git identifies the first bad commit.

git bisect reset   # Always clean up when done
```

You will find that "Commit 3" introduced the bug. This is the exact workflow for tracking down when AI broke something.

### Part 3: Stash and cherry-pick

```bash
git checkout main
echo "Unfinished thought" >> readme.txt
git stash                   # Save this without committing
echo "Finished thought" >> readme.txt
git add readme.txt
git commit -m "Complete the thought"
git stash pop                # Bring back the stashed work
# Now decide what to do with it
```

## Common mistakes

### 1. Forgetting to pull before pushing

**What happens:** You `git push` but the remote has commits you don't have. Git rejects the push because it would overwrite remote history.

**Why it happens:** Git's default behavior prevents you from accidentally destroying others' work.

**How to fix:** Run `git pull --rebase` (or just `git pull`) before pushing. If you get a "diverged" error, you need to either merge the remote changes in or rebase yours on top. Ask a teammate before force-pushing anything.

### 2. Committing directly to main

**What happens:** You commit work directly to `main` and then want to code-review it via PR. Now the "review" happens after the fact.

**Why it happens:** It feels faster to skip the branch.

**How to fix:** Get in the habit of `git checkout -b feature/my-work` immediately when starting anything. Default to branches for everything, even small changes.

### 3. Writing terrible commit messages

**What happens:** Six months later you run `git log` and see: "fix", "update", "asdf", "WIP". You have no idea what any of those commits contain.

**Why it happens:** Commit messages feel like overhead when you're in flow.

**How to fix:** Use the imperative mood: "Add user authentication", not "Added" or "Adding". A good commit message answers: what changed and why? If the "why" isn't obvious from the diff, the message should explain it.

### 4. Not using .gitignore

**What happens:** `node_modules/` (thousands of files) gets committed. The repo becomes gigabytes in size. Every clone takes forever.

**Why it happens:** `.gitignore` isn't created automatically and the temptation to track everything is real.

**How to fix:** Create `.gitignore` before your first commit. Use GitHub's [gitignore templates](https://github.com/github/gitignore) as a starting point for your language/framework.

### 5. Accidentally committing secrets

**What happens:** API keys, passwords, and tokens end up in the repository. They may be removed in a later commit but remain in history forever, searchable by anyone who clones the repo.

**Why it happens:** `.env` files aren't automatically ignored, and adding them after the fact doesn't remove them from history.

**How to fix:** Use `git secrets` or similar tools in CI. If caught early: remove the secret, commit the fix, force-push. If deeply embedded in history, use `git filter-branch` or BFG Repo-Cleaner to rewrite history and then force-push. This is a serious incident — treat it as such.

## AI-specific pitfalls

### AI suggests `git checkout -f` or `git reset --hard` to "undo" changes

Git checkout with the `-f` flag forces the checkout and **discards all uncommitted changes**. AI will sometimes suggest this to undo edits. If you have uncommitted work, it is gone.

What to look for: Any suggestion that uses `-f` or `--force` on `checkout`, or any form of `reset --hard` that doesn't first verify your working directory is clean. Always run `git status` before force operations.

**Safer alternative:** `git restore filename.txt` to discard unstaged changes, or `git stash` to save them temporarily.

### AI hallucinates branch names and commit hashes

AI may confidently tell you to "check out `feat-authentication-v2`" when no such branch exists. It may reference commit hashes that don't exist in your repository.

What to do: Always verify with `git branch -a` for branches and `git log --oneline` for commit hashes. Don't run commands AI gives you that reference names you haven't confirmed exist.

### AI doesn't understand what should be in .gitignore

AI will often include generated files, build artifacts, and dependencies in commits unless explicitly told not to. It also commonly fails to ignore `.env` files containing secrets.

What to do: Explicitly mention "do not commit node_modules, build/, .env, __pycache__, or any generated files" in your prompt. Verify with `git status` before committing.

### AI suggests `git push --force` too casually

Force-push is sometimes necessary (e.g., after amending a commit that was already pushed to a private branch), but AI often suggests it without understanding the consequences for shared branches.

What to do: Treat `--force` as a red flag. Ask: "Is this branch shared? Have others pushed to it?" Only force-push to personal, private branches. If you're unsure, use `git push --force-with-lease` instead — it refuses to push if someone else has pushed to the branch since you last fetched.

### AI doesn't know your project structure

AI may suggest commands that reference files, paths, or branch conventions that don't match your project. It generates plausible-sounding but incorrect Git commands.

What to do: Always prepend `git` commands with a dry-run flag where available (`git add -n .` to preview staging), and always verify the result with `git status` before executing.

## Quick reference

### The everyday command set

| Task | Command |
|------|---------|
| Start tracking a folder | `git init` |
| Stage a file | `git add filename.txt` |
| Stage everything | `git add .` |
| Commit | `git commit -m "message"` |
| See what changed | `git status` |
| See unstaged changes | `git diff` |
| See staged changes | `git diff --staged` |
| Discard unstaged edits | `git restore filename.txt` |
| Unstage a file | `git restore --staged filename.txt` |
| Create a branch | `git checkout -b name` |
| Switch branches | `git checkout name` |
| Merge into current branch | `git merge branchname` |
| Push commits | `git push` |
| Pull commits | `git pull` |
| Save work temporarily | `git stash` |
| Restore stashed work | `git stash pop` |
| See history | `git log --oneline` |
| Find a bad commit | `git bisect start` |

### Decision tree: Undoing things

```
Did you commit but not push?
  YES -> git reset --soft HEAD~1    (undo commit, keep changes staged)
  YES -> git reset HEAD~1          (undo commit, keep changes unstaged)
  YES -> git reset --hard HEAD~1   (undo commit, DISCARD changes — careful!)

Did you add but not commit?
  YES -> git restore --staged filename.txt

Did you edit but not add?
  YES -> git restore filename.txt   (discard edits)
  YES -> git stash                  (save for later)
```

### .gitignore rules

```
*.log           # Ignore all .log files
node_modules/   # Ignore this directory
.env            # Ignore this specific file
build/          # Ignore this directory
```

## Go deeper

- [Git Official Documentation](https://git-scm.com/doc) — The definitive reference. Bookmark it. (verified 2026-04)
- [GitHub Git Cheat Sheet](https://education.github.com/git-cheat-sheet-education.pdf) — Concise reference for the commands above. (verified 2026-04)
- [Atlassian Git Tutorial](https://www.atlassian.com/git/tutorials) — Excellent for building mental models, from beginner to advanced. (verified 2026-04)
- ["Git for Computer Scientists" — Tommy's Take](https://eagain.net/articles/git-for-computer-scientists/) — A short, precise explanation of how Git actually works under the hood using Directed Acyclic Graphs. Worth reading after you know the commands. (verified 2026-04)
- [Oh Shit, Git!?!" by Katie Sylor-Miller](https://ohshitgit.com/) — Hilarious and practical solutions for common Git disasters. The page you'll wish you had bookmarked before something went wrong. (verified 2026-04)
