# Module 4 (non-Azure): CI/CD with GitHub Actions

> **Track:** Core Track
> **Duration:** 3–4 days
> **Level:** Beginner
> **Prerequisites:** Module 3: CI/CD Fundamentals
> **Note:** Complete either Module 4A or Module 4B based on your client platform. Both cover equivalent core skills.

---

## Table of Contents

1. [GitHub Actions Overview](#1-github-actions-overview)
2. [Workflow YAML Structure](#2-workflow-yaml-structure)
3. [Using Marketplace Actions](#3-using-marketplace-actions)
4. [Multi-Job Workflows](#4-multi-job-workflows)
5. [Secrets and GitHub Environments](#5-secrets-and-github-environments)
6. [Environment Protection Rules](#6-environment-protection-rules)
7. [Triggering Workflows](#7-triggering-workflows)
8. [Debugging Failed Runs](#8-debugging-failed-runs)
9. [GitHub Actions vs Azure Pipelines](#9-github-actions-vs-azure-pipelines)
10. [Lab: Build a Multi-Job Workflow for a Python Project](#10-lab-build-a-multi-job-workflow-for-a-python-project)
11. [Quick Reference](#11-quick-reference)

---

## 1. GitHub Actions Overview

GitHub Actions is GitHub's built-in CI/CD platform. Workflows are defined as YAML files committed to your repo inside the `.github/workflows/` folder. When a trigger fires, GitHub runs the workflow on a runner (GitHub's term for agent).

Key characteristics:

- **YAML-based:** Workflows are code, versioned alongside your project
- **GitHub-hosted runners:** Free virtual machines per run; no infrastructure to manage
- **Marketplace:** Thousands of pre-built actions for common tasks (AWS auth, Docker push, Slack notify, etc.)
- **Tight GitHub integration:** Directly tied to PRs, issues, releases, and GitHub's permission model
- **AWS-native:** Commonly used on AWS-based client projects; integrates natively with AWS IAM via OIDC

---

## 2. Workflow YAML Structure

### 2.1 Minimal Workflow

```yaml
name: CI

on:
  push:
    branches:
      - main

jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
      - name: Say hello
        run: echo "Hello from GitHub Actions"
```

Every workflow needs: a name, an `on` trigger, at least one job, and steps inside the job.

### 2.2 Full Structure Reference

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:                                  # Workflow-level environment variables
  PYTHON_VERSION: "3.11"
  AWS_REGION: ap-southeast-1

jobs:
  test:
    name: Lint and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4     # Marketplace action to clone the repo

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: pip install flake8 pytest

      - name: Lint with flake8
        run: flake8 src/ tests/

      - name: Run tests
        run: python -m pytest tests/ --junitxml=test-results.xml

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: test                       # Only runs if test job succeeded
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy
        run: echo "Deploying build ${{ github.run_id }}"
```

### 2.3 Expressions and Contexts

GitHub Actions uses `${{ }}` syntax for expressions:

```yaml
# Access context values
${{ github.sha }}              # Commit SHA
${{ github.ref }}              # Full ref (e.g., refs/heads/main)
${{ github.event_name }}       # What triggered the workflow (push, pull_request, etc.)
${{ runner.os }}               # OS of the runner

# Access secrets
${{ secrets.MY_SECRET }}

# Access environment variables
${{ env.PYTHON_VERSION }}

# Conditional expression
if: github.ref == 'refs/heads/main'
```

---

## 3. Using Marketplace Actions

Marketplace actions are reusable steps published by GitHub, AWS, Microsoft, and the community. Instead of writing shell commands for common tasks, you call a pre-built action with `uses:`.

### 3.1 Essential Actions

**Checkout code (always the first step):**
```yaml
- name: Checkout code
  uses: actions/checkout@v4
```

**Set up Python:**
```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.11"
    cache: pip               # Caches pip dependencies between runs for speed
```

**Configure AWS credentials:**
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-southeast-1
```

**Upload artifact:**
```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: app-package
    path: dist/
    retention-days: 7
```

**Download artifact:**
```yaml
- name: Download artifact
  uses: actions/download-artifact@v4
  with:
    name: app-package
    path: dist/
```

**Post a comment on a PR:**
```yaml
- name: Comment on PR
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: 'Tests passed!'
      })
```

### 3.2 Finding Actions

Browse the marketplace at [github.com/marketplace](https://github.com/marketplace?type=actions). Always check:

- The number of stars and downloads (popularity indicator)
- Whether it is published by a verified organization (GitHub, AWS, Microsoft)
- The version; always pin to a specific version tag (e.g., `@v4`) not `@latest`

---

## 4. Multi-Job Workflows

### 4.1 Job Dependencies with `needs`

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: python -m pytest tests/

  build:
    runs-on: ubuntu-latest
    needs: test               # Only runs after test succeeds
    steps:
      - run: echo "Building"

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: echo "Deploying to staging"

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production   # Requires approval (Section 6)
    steps:
      - run: echo "Deploying to production"
```

### 4.2 Parallel Jobs

Jobs without `needs` run in parallel automatically:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: flake8 src/

  test:
    runs-on: ubuntu-latest
    steps:
      - run: python -m pytest tests/

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running security scan"

  # All three above run simultaneously
  deploy:
    needs: [lint, test, security-scan]   # Wait for all three
    runs-on: ubuntu-latest
    steps:
      - run: echo "All checks passed; deploying"
```

### 4.3 Passing Data Between Jobs

Use artifacts to pass files:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: zip -r app.zip src/
      - uses: actions/upload-artifact@v4
        with:
          name: app-package
          path: app.zip

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-package
      - run: echo "Deploying app.zip"
```

Use job outputs to pass small values (strings, numbers):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

---

## 5. Secrets and GitHub Environments

### 5.1 Adding Secrets to a Repo

1. Go to your GitHub repository
2. Click **Settings; Secrets and variables; Actions**
3. Click **New repository secret**
4. Add a name (e.g., `AWS_ACCESS_KEY_ID`) and value
5. Save

Reference in your workflow:

```yaml
steps:
  - name: Use secret
    run: echo "Secret is set (value masked)"
    env:
      MY_SECRET: ${{ secrets.MY_SECRET }}
```

Secrets are masked in all logs; if a step accidentally prints a secret, GitHub replaces it with `***`.

### 5.2 Organization Secrets

If your GitHub organization has shared secrets (e.g., a company-wide AWS role), they are available automatically in all repos. You reference them the same way: `${{ secrets.SECRET_NAME }}`.

### 5.3 Environment Secrets

For environment-specific secrets (dev vs staging vs production), use GitHub Environments:

1. Go to **Settings; Environments**
2. Create environments: `dev`, `staging`, `production`
3. Add secrets per environment (e.g., different AWS credentials for each)

Reference in your workflow:

```yaml
jobs:
  deploy:
    environment: production           # Activates production environment secrets
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}       # Production key
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
```

---

## 6. Environment Protection Rules

GitHub Environments can require approval before a job runs; equivalent to Azure Pipelines environment approvals.

### Setting Up Protection Rules

1. Go to **Settings; Environments**
2. Click on `production`
3. Enable **Required reviewers**
4. Add the GitHub usernames of required approvers
5. Save

When a workflow reaches a job with `environment: production`, it pauses and sends a notification to the reviewers. The workflow continues only after approval.

### Other Protection Options

| Rule | What it does |
|---|---|
| Required reviewers | Specific people must approve |
| Wait timer | Waits N minutes before allowing the job to proceed |
| Deployment branches | Only allows deployments from specific branches (e.g., only `main`) |

---

## 7. Triggering Workflows

### Push Trigger

```yaml
on:
  push:
    branches:
      - main
      - develop
    paths:
      - src/**            # Only trigger if files in src/ changed
      - requirements.txt
```

### PR Trigger

```yaml
on:
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize      # Re-runs when new commits are pushed to the PR
      - reopened
```

### Schedule Trigger

```yaml
on:
  schedule:
    - cron: "0 6 * * *"   # Every day at 6:00 AM UTC
```

### Manual Trigger

```yaml
on:
  workflow_dispatch:        # Adds a "Run workflow" button in the GitHub UI
    inputs:
      environment:
        description: Target environment
        required: true
        default: dev
        type: choice
        options:
          - dev
          - staging
          - production
```

### Combined Triggers

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1"   # Every Monday at 6 AM UTC
  workflow_dispatch:
```

---

## 8. Debugging Failed Runs

### Reading Workflow Logs

Go to your repository; click **Actions**; select the failed workflow run. Each job is expandable; click on a job to see all its steps. The failed step shows the exact error output. Look for:

- The command that failed and its exit code
- Missing environment variables or files
- Authentication errors (common with AWS credentials)

### Common Failure Causes

| Symptom | Likely cause |
|---|---|
| `Error: Process completed with exit code 1` | Script command failed; read the lines above for the actual error |
| `credentials: not found` | AWS or cloud credentials not configured; check secrets |
| `uses: unknown action` | Typo in action name or version; check marketplace spelling |
| Workflow not triggering on PR | Check branch filters in `pull_request:` trigger |
| `needs` job skipped | An upstream job failed or was skipped; check that job's result |

### Re-running Failed Jobs

In the GitHub Actions UI:

1. Open the failed workflow run
2. Click **Re-run failed jobs** (retries only failed jobs) or **Re-run all jobs** (starts fresh)

### Enabling Debug Logging

Add these as repository secrets (not workflow variables) to enable verbose logging:

- `ACTIONS_RUNNER_DEBUG` = `true`
- `ACTIONS_STEP_DEBUG` = `true`

---

## 9. GitHub Actions vs Azure Pipelines

Since you may encounter both, here is a direct comparison of the key syntax and concept differences:

| Concept | Azure Pipelines | GitHub Actions |
|---|---|---|
| Pipeline file location | Anywhere in repo (configured at registration) | Must be in `.github/workflows/` |
| Trigger keyword | `trigger:` (push); `pr:` (PR) | `on: push:` / `on: pull_request:` |
| Agent/runner | `pool: vmImage:` | `runs-on:` |
| Step (script) | `- script:` | `- run:` |
| Step (task) | `- task: Name@version` | `- uses: org/action@version` |
| Variable reference | `$(variable-name)` | `${{ env.VARIABLE_NAME }}` |
| Secret reference | `$(secret-name)` | `${{ secrets.SECRET_NAME }}` |
| Stage | `stages: - stage:` | No stages; use jobs with `needs:` |
| Job dependency | `dependsOn:` (stage level) | `needs:` (job level) |
| Condition | `condition: succeeded()` | `if: success()` |
| Environment approval | Environment in Azure DevOps UI | `environment:` key with protection rules |
| Artifact upload | `PublishBuildArtifacts@1` task | `actions/upload-artifact@v4` |
| Artifact download | `DownloadBuildArtifacts@1` task | `actions/download-artifact@v4` |

---

## 10. Lab: Build a Multi-Job Workflow for a Python Project

This lab has two parts. Part A is a short warm-up that completes the pipeline you read in Module 3. Part B builds the full multi-job workflow.

### Part A: Write Your First GitHub Actions Workflow (Warm-Up)

In Module 3 you read and annotated a complete GitHub Actions workflow. Now write your own version from scratch using what you learned in this module.

**Goal:** A workflow that triggers on every PR to `main`, installs Python, runs `flake8` on a Python file, and fails the workflow if linting fails.

Create `.github/workflows/ci.yml` in your practice repo:

```yaml
name: Lint

on:
  pull_request:
    branches:
      - main

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install flake8
        run: pip install flake8

      - name: Lint with flake8
        run: flake8 src/
```

Create `src/hello.py`:

```python
def greet(name):
    message = "Hello, " + name + "!"
    return message

print(greet("Data Engineer"))
```

Push on a feature branch, open a PR, and watch the workflow trigger. Then introduce a linting error, push again, and confirm the workflow fails. Fix it and confirm it goes green.

Once it is green, proceed to Part B.

---

### Part B: Full Multi-Job Workflow

### Prerequisites
- Part A of this lab completed
- A GitHub repository (free account at [github.com](https://github.com))
- AWS account with Secrets Manager access (ask your team lead)

---

### Step 1: Prepare Your Repo

In your GitHub repo, create the following files:

`src/hello.py`:
```python
def greet(name):
    message = "Hello, " + name + "!"
    return message
```

`tests/test_hello.py`:
```python
from src.hello import greet

def test_greet():
    assert greet("World") == "Hello, World!"

def test_greet_engineer():
    assert greet("Engineer") == "Hello, Engineer!"
```

`requirements.txt`:
```
flake8
pytest
boto3
```

Commit and push to a feature branch.

---

### Step 2: Add AWS Credentials as Secrets

1. Go to your repo; **Settings; Secrets and variables; Actions**
2. Add:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

---

### Step 3: Create the Workflow File

Create `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  PYTHON_VERSION: "3.11"
  AWS_REGION: ap-southeast-1

jobs:
  test:
    name: Lint and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Lint with flake8
        run: flake8 src/ tests/

      - name: Run tests
        run: python -m pytest tests/ --junitxml=test-results.xml

      - name: Publish test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results.xml

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: dev

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Retrieve secret from AWS Secrets Manager
        run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id de-practice/dev/db-password \
            --query SecretString \
            --output text)
          echo "::add-mask::$SECRET"
          echo "DB_PASSWORD=$SECRET" >> $GITHUB_ENV

      - name: Deploy
        run: |
          echo "Deploying build ${{ github.run_id }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "DB_PASSWORD is set: ${{ env.DB_PASSWORD != '' }}"
```

---

### Step 4: Test the CI Job with a PR

Push your files on a feature branch and open a PR targeting `main`. The workflow should trigger and run the `test` job. Confirm:

- Lint passes (or fails if you have a style issue; fix it)
- Tests pass
- The `deploy` job is skipped (because this is a PR, not a push to `main`)

---

### Step 5: Introduce a Test Failure

Edit `tests/test_hello.py` to make a test fail:

```python
def test_greet():
    assert greet("World") == "Wrong answer"   # Intentional failure
```

Push and watch the workflow fail on the test step. Notice the `deploy` job is blocked. Fix the test, push again, and confirm the workflow goes green.

---

### Step 6: Merge and Observe the Deploy Job

Merge your PR to `main`. The workflow triggers again. This time both jobs run; `test` first, then `deploy` after it succeeds.

Confirm in the Actions UI that:
- The `deploy` job only ran on the push to `main`, not on the PR
- The AWS secret was retrieved and masked in the logs

---

## 11. Quick Reference

| Concept | Syntax |
|---|---|
| Push trigger | `on: push: branches:` |
| PR trigger | `on: pull_request: branches:` |
| Schedule trigger | `on: schedule: cron:` |
| Manual trigger | `on: workflow_dispatch:` |
| Runner | `runs-on: ubuntu-latest` |
| Checkout | `uses: actions/checkout@v4` |
| Script step | `- run:` |
| Action step | `- uses: org/action@version` |
| Job dependency | `needs: job-name` |
| Condition | `if: github.ref == 'refs/heads/main'` |
| Secret reference | `${{ secrets.SECRET_NAME }}` |
| Env variable | `${{ env.VAR_NAME }}` |
| GitHub context | `${{ github.sha }}` / `${{ github.run_id }}` |
| Environment approval | `environment: production` with protection rules |
| Upload artifact | `uses: actions/upload-artifact@v4` |
| Download artifact | `uses: actions/download-artifact@v4` |
| Mask a value in logs | `echo "::add-mask::$VALUE"` |

---

*Next module: Module 5: Environment & Secret Management (skip Module 4A if you completed 4B)*
