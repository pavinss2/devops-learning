# Module 1: Version Control with Git

> **Track:** Core Track
> **Duration:** 3–4 days
> **Level:** Beginner
> **Prerequisites:** None

---

## Table of Contents

1. [Why Git Matters for Data Engineers](#1-why-git-matters-for-data-engineers)
2. [Core Concepts](#2-core-concepts)
3. [Installing and Configuring Git](#3-installing-and-configuring-git)
4. [Working with a Repository](#4-working-with-a-repository)
5. [Branching and Merging](#5-branching-and-merging)
6. [Collaborating with Pull Requests](#6-collaborating-with-pull-requests)
7. [.gitignore Best Practices for Data Projects](#7-gitignore-best-practices-for-data-projects)
8. [Lab: Your First Git Workflow](#8-lab-your-first-git-workflow)
9. [Quick Reference](#9-quick-reference)

---

## 1. Why Git Matters for Data Engineers

As a data engineering consultant, you will almost always work on a shared codebase with other engineers. Without version control, teams run into problems like:

- Overwriting each other's work
- No record of who changed what and why
- No way to roll back a broken deployment
- Difficulty reviewing code before it goes to production

Git solves all of these. It tracks every change made to every file in a project, lets multiple people work in parallel without conflict, and creates a full audit trail of the codebase. Every DevOps workflow, CI/CD pipeline, and deployment process in this training track starts from a Git repository.

---

## 2. Core Concepts

Before touching any commands, it helps to understand what Git is actually doing.

### Repository (Repo)
A repository is a folder that Git is tracking. It contains your project files plus a hidden `.git` folder where Git stores its history and configuration. A repo can live locally on your machine or remotely on a platform like Azure Repos or GitHub.

### Commit
A commit is a saved snapshot of your changes. Every commit has a unique ID (called a hash), an author, a timestamp, and a message describing what changed. Think of commits as checkpoints you can always go back to.

### Branch
A branch is an independent line of development. The default branch is usually called `main`. When you want to work on something new without affecting `main`, you create a new branch, make your changes there, and merge it back when ready.

```
main:    A --- B --- C
                      \
feature:               D --- E
```

### Staging Area
Git has a two-step save process. First you `add` changes to the staging area (telling Git "I want to include this in my next commit"), then you `commit` them. This lets you choose exactly which changes go into each commit.

### Remote
A remote is a copy of the repo hosted somewhere else, typically on Azure Repos or GitHub. `origin` is the conventional name for your main remote. You `push` to send your commits to the remote, and `pull` to bring changes from the remote into your local copy.

---

## 3. Installing and Configuring Git

### Which Terminal Should I Use?

All commands in this module work in both terminals below. Use whichever you are more comfortable with.

| Terminal | How to open |
|---|---|
| **PowerShell** | Start menu; search "PowerShell" |
| **WSL (Ubuntu)** | Start menu; search "Ubuntu" or "WSL" |

> **Note:** Git commands are identical in both terminals. The only difference is how you navigate folders (`cd`, `mkdir`) which also works the same way in both.

---

### Install Git

**Option A: PowerShell**

Download the installer from [https://git-scm.com](https://git-scm.com) and run it. During installation, keep all default options. Git Bash will also be installed but you do not need it; use PowerShell instead.

Verify the install:
```powershell
git --version
# Expected: git version 2.x.x
```

**Option B: WSL (Ubuntu)**
```bash
sudo apt update && sudo apt install git
git --version
```

---

### Configure Your Identity

Run these once after installing. Git will attach your name and email to every commit you make.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@company.com"
git config --global init.defaultBranch main
```

> These commands are the same in both PowerShell and WSL.

Verify your config:
```bash
git config --list
```

---

## 4. Working with a Repository

### 4.1 Creating a New Repo

**PowerShell:**
```powershell
mkdir my-project
cd my-project
git init
```

**WSL:**
```bash
mkdir my-project && cd my-project
git init
```

This creates a new local repository. A hidden `.git` folder will appear inside the folder.

---

### 4.2 Cloning an Existing Repo

More often, you will clone a repo that already exists on a remote platform:

```bash
git clone https://dev.azure.com/your-org/your-project/_git/your-repo
# or from GitHub:
git clone https://github.com/your-org/your-repo.git
```

This downloads the full repo and sets `origin` as the remote automatically. Works the same in both PowerShell and WSL.

---

### 4.3 The Basic Daily Workflow

This is the sequence you will use every day:

```bash
# 1. Check what has changed in your working directory
git status

# 2. See the actual line-by-line differences
git diff

# 3. Stage a specific file
git add notes.txt

# 4. Or stage everything changed
git add .

# 5. Commit with a descriptive message
git commit -m "Add project notes"

# 6. Push your commits to the remote
git push origin your-branch-name
```

---

### 4.4 Pulling Changes from Remote

Before starting work each day, pull the latest changes so your local copy is up to date:

```bash
git pull origin main
```

---

### 4.5 Checking History

```bash
# Full commit history
git log

# Compact one-line view
git log --oneline

# Visual graph showing branches
git log --oneline --graph --all
```

---

## 5. Branching and Merging

### 5.1 Creating and Switching Branches

Never commit directly to `main`. Always create a feature branch for your work:

```bash
# Create a new branch and switch to it (modern syntax; Git 2.23+)
git switch -c feature/my-new-feature

# List all local branches (current branch has an asterisk)
git branch

# Switch back to main
git switch main
```

> **Note on `git checkout`:** You will often see `git checkout -b feature/my-branch` in older tutorials and on client projects. It does the same thing as `git switch -c`. Both work; `git switch` is the modern, clearer alternative introduced in Git 2.23. Use `git switch` going forward but do not be surprised when you see `git checkout` in the wild.

---

### 5.2 Branching Strategy

On client projects, you will typically follow a branching model like this:

| Branch | Purpose |
|---|---|
| `main` | Production-ready code only; protected, no direct commits |
| `develop` | Integration branch; feature branches merge here first |
| `feature/your-feature` | Your working branch for a specific task |
| `hotfix/issue-name` | Emergency fix branched directly from `main` |

A typical flow:

```
1. Branch off develop:   git switch -c feature/my-task
2. Do your work and commit regularly
3. Push to remote:       git push origin feature/my-task
4. Open a Pull Request from feature/my-task into develop
5. After review and approval, merge and delete the feature branch
```

---

### 5.3 Merging

After a PR is approved, the merge usually happens through the platform UI. But it is useful to know the commands:

```bash
# Switch to the target branch
git switch develop

# Merge your feature branch into it
git merge feature/my-new-feature

# Delete the feature branch after merging
git branch -d feature/my-new-feature
```

---

### 5.4 Resolving Merge Conflicts

A conflict happens when two branches have changed the same part of the same file. Git will mark the conflict in the file like this:

```
<<<<<<< HEAD
Project owner: Alice
=======
Project owner: Bob
>>>>>>> feature/update-owner
```

To resolve:

1. Open the file and decide which version to keep (or combine both)
2. Remove all the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
3. Stage and commit the resolved file:

```bash
git add notes.txt
git commit -m "Resolve merge conflict in project notes"
```

---

### 5.5 Rebase vs. Merge

| | Merge | Rebase |
|---|---|---|
| What it does | Creates a merge commit joining two branches | Replays your commits on top of another branch |
| History | Preserves full branch history | Creates a cleaner, linear history |
| When to use | Default for merging via PR | Updating your feature branch with latest main before opening a PR |
| Risk | Low | Do NOT rebase shared or public branches |

A safe use of rebase; updating your feature branch before opening a PR:

```bash
git fetch origin
git rebase origin/main
```

---

## 6. Collaborating with Pull Requests

A Pull Request (PR) is a request to merge your branch into another branch. It is the standard code review mechanism on all client projects. PRs are not a Git feature; they exist on the platform (Azure Repos or GitHub), but they are built on top of Git branches.

### 6.1 Opening a PR

1. Push your feature branch to the remote
2. Go to Azure Repos or GitHub in your browser
3. Find the prompt to open a PR for your recently pushed branch
4. Fill in: title, description, target branch, and assign reviewers

**A good PR description includes:**
- What changed and why
- Any related work item or ticket number
- How to verify the change
- Any known limitations

---

### 6.2 Reviewing a PR

As a reviewer, check:
- Does the change do what the description says?
- Are there any obvious mistakes or missing cases?
- Is naming clear and consistent?
- Are there any hardcoded credentials or environment-specific values?

Leave comments on specific lines, request changes if needed, or approve and complete the merge.

---

### 6.3 Keeping Your PR Up to Date

If `main` or `develop` moves forward while your PR is open:

```bash
git fetch origin
git rebase origin/develop
git push origin feature/your-branch --force-with-lease
```

> Use `--force-with-lease` instead of `--force`; it is safer because it checks that nobody else has pushed to your branch since your last fetch.

---

## 7. .gitignore Best Practices for Data Projects

A `.gitignore` file tells Git which files and folders to never track. This is critical for data projects because you never want to accidentally commit credentials, large data files, or local config files.

### Example `.gitignore` for a Data Engineering Project

```gitignore
# Python
__pycache__/
*.py[cod]
.eggs/

# Virtual environments
.venv/
venv/
env/

# Environment and secrets (NEVER commit these)
.env
.env.*
secrets.json
credentials.json

# Data files (never commit raw data)
*.csv
*.parquet
data/raw/
data/processed/

# Logs
*.log
logs/

# IDE and OS files
.vscode/
.idea/
.DS_Store
Thumbs.db

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars

# dbt (if applicable)
target/
dbt_packages/
```

### Key Rules

- Always create a `.gitignore` before your first commit
- If you accidentally committed a secret, removing it from `.gitignore` does not erase it from Git history; rotate the credential immediately and contact your team lead
- Use [gitignore.io](https://www.toptal.com/developers/gitignore) to generate templates for your specific stack

---

## 8. Lab: Your First Git Workflow

This lab walks through a complete feature branch workflow from start to finish. No coding experience is needed; the "project files" are just plain text files.

### Prerequisites
- Git installed and configured (Section 3)
- Access to Azure Repos or GitHub (ask your team lead for a practice repo, or create a free account at [github.com](https://github.com))

---

### Step 1: Clone the Practice Repo

```bash
git clone <url-provided-by-your-team-lead>
cd practice-repo
```

If you do not have a practice repo yet, create one locally:

**PowerShell:**
```powershell
mkdir practice-repo
cd practice-repo
git init
```

**WSL:**
```bash
mkdir practice-repo && cd practice-repo
git init
```

---

### Step 2: Create an Initial File and Commit

Create a plain text file called `project-notes.txt` with any text editor (Notepad is fine):

```
Project: Data Pipeline v1
Owner: Your Name
Status: In Progress
```

Then stage and commit it:

```bash
git add project-notes.txt
git commit -m "Add initial project notes"
```

---

### Step 3: Create a Feature Branch

```bash
git switch -c feature/update-project-notes
```

Verify you are on the new branch:

```bash
git branch
# * feature/update-project-notes
#   main
```

---

### Step 4: Make a Change on the Feature Branch

Open `project-notes.txt` and add a new line:

```
Project: Data Pipeline v1
Owner: Your Name
Status: In Progress
Notes: Added ingestion for customer data source.
```

Stage and commit:

```bash
git add project-notes.txt
git commit -m "Add ingestion note to project notes"
```

---

### Step 5: Push to Remote and Open a PR

```bash
git push origin feature/update-project-notes
```

Then go to your Azure Repos or GitHub URL, open a PR with:
- **Title:** Update project notes with ingestion detail
- **Target branch:** `main`
- **Description:** Added a notes field to project-notes.txt documenting the new customer data ingestion.

---

### Step 6: Simulate a Conflict (Optional but Recommended)

This step simulates a colleague editing the same file on `main` while you were working on your feature branch.

First, switch to `main`:

```bash
git switch main
```

Open `project-notes.txt` in Notepad and change this line:

```
Status: In Progress
```

to:

```
Status: Under Review
```

Save the file, then come back to the terminal and commit the change:

```bash
git add project-notes.txt
git commit -m "Update status to Under Review"
```

Now switch back to your feature branch:

```bash
git switch feature/update-project-notes
```

Now rebase your feature branch onto the updated `main`:

```bash
git fetch origin
git rebase origin/main
```

Git will pause at the conflict. Open `project-notes.txt`, resolve it by keeping both changes, then:

```bash
git add project-notes.txt
git rebase --continue
```

---

### Step 7: Merge and Clean Up

Once your PR is approved and merged via the UI:

```bash
git switch main
git pull origin main
git branch -d feature/update-project-notes
```

You have completed a full feature branch workflow.

---

## 9. Quick Reference

| Command | What it does |
|---|---|
| `git init` | Initialize a new local repo |
| `git clone <url>` | Clone a remote repo locally |
| `git status` | Show changed and staged files |
| `git diff` | Show unstaged line-by-line changes |
| `git add <file>` | Stage a file |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Commit staged changes |
| `git push origin <branch>` | Push branch to remote |
| `git pull origin <branch>` | Pull latest from remote branch |
| `git switch -c <branch>` | Create and switch to new branch (modern) |
| `git switch <branch>` | Switch to an existing branch |
| `git checkout -b <branch>` | Create and switch to new branch (older syntax; still works) |
| `git merge <branch>` | Merge branch into current branch |
| `git rebase origin/main` | Replay commits on top of main |
| `git log --oneline --graph` | Compact visual history |
| `git stash` | Temporarily save uncommitted changes |
| `git stash pop` | Restore stashed changes |

---

*Next module: Module 2: Infrastructure as Code with Terraform*
