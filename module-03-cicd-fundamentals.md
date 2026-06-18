# Module 3: CI/CD Fundamentals

> **Track:** Core Track
> **Duration:** 3–4 days
> **Level:** Beginner
> **Prerequisites:** Module 2: Infrastructure as Code with Terraform

---

## Table of Contents

1. [What is CI/CD and Why it Exists](#1-what-is-cicd-and-why-it-exists)
2. [The CI/CD Lifecycle](#2-the-cicd-lifecycle)
3. [Key Concepts](#3-key-concepts)
4. [YAML Pipeline Basics](#4-yaml-pipeline-basics)
5. [Secrets and Environment Variables in Pipelines](#5-secrets-and-environment-variables-in-pipelines)
6. [CI vs CD: When to Use Each](#6-ci-vs-cd-when-to-use-each)
7. [Common Pipeline Patterns for Data Projects](#7-common-pipeline-patterns-for-data-projects)
8. [Lab: Your First YAML Pipeline](#8-lab-your-first-yaml-pipeline)
9. [Quick Reference](#9-quick-reference)

---

## 1. What is CI/CD and Why it Exists

CI/CD stands for Continuous Integration and Continuous Delivery (or Deployment). It is the practice of automating the steps between a developer writing code and that code reaching its destination; whether that is a test environment, a staging database, or production.

### The Problem Without CI/CD

Without CI/CD, deploying changes is a manual, error-prone process:

- A developer finishes a script and emails it to a colleague to deploy
- Someone manually copies files to a server or runs commands by hand
- Nobody ran tests before deploying; a bug reaches production
- Two people deploy different versions at the same time and overwrite each other
- There is no record of what was deployed, when, or by whom

On client projects, this is not acceptable. Clients expect a professional, repeatable, auditable delivery process.

### What CI/CD Solves

- **Continuous Integration (CI):** Every time code is pushed or a PR is opened, automated checks run; tests, linting, validation. Problems are caught early, before they reach the main branch.
- **Continuous Delivery (CD):** After code passes CI checks and is merged, the deployment to a target environment is triggered automatically or with a single approval click.

Together they create a pipeline; a series of automated stages that code travels through from commit to deployment.

---

## 2. The CI/CD Lifecycle

A typical CI/CD lifecycle for a data engineering project looks like this:

```
Developer pushes code
        |
        v
  [CI: Triggered on PR]
  1. Lint (check code style)
  2. Unit tests
  3. Validate configs (Terraform plan, schema checks)
        |
        v
  PR reviewed and approved
        |
        v
  Merge to main
        |
        v
  [CD: Triggered on merge]
  4. Build (package code, build Docker image)
  5. Deploy to staging
  6. Run integration tests
  7. Approve (manual gate for production)
  8. Deploy to production
```

Not every project uses every step. A simple project might just lint on PR and deploy on merge. A mature project might have multiple environments and approval gates.

---

## 3. Key Concepts

### Trigger
A trigger defines when the pipeline runs. Common triggers:

| Trigger | When it fires |
|---|---|
| Pull Request | When a PR is opened or updated against a target branch |
| Push to branch | When commits are pushed to a specific branch |
| Schedule | At a fixed time (e.g., every night at midnight) |
| Manual | When a person clicks "Run" in the UI |

### Agent / Runner
The agent (Azure Pipelines term) or runner (GitHub Actions term) is the machine that executes the pipeline steps. It can be:

- **Microsoft-hosted / GitHub-hosted:** A fresh virtual machine spun up by the platform for each run. No maintenance required; the most common choice.
- **Self-hosted:** A machine you manage yourself. Used when you need access to resources inside a private network or when you need specific software pre-installed.

### Stage
A stage is a major phase of the pipeline; for example, a CI stage and a CD stage. Stages can run sequentially or in parallel, and a stage can be configured to only run if a previous stage succeeded.

### Job
A stage contains one or more jobs. Each job runs on its own agent. Jobs within a stage can run in parallel.

### Step
A step is a single action within a job; running a script, calling a built-in task, or using a marketplace action. Steps run sequentially within a job.

### Artifact
An artifact is a file or set of files produced by a pipeline run and passed between stages or jobs. For example; a packaged zip file produced in the CI stage and consumed by the CD stage for deployment.

---

## 4. YAML Pipeline Basics

Both Azure Pipelines and GitHub Actions use YAML to define pipelines. The syntax is slightly different between the two, but the concepts are identical. This section covers the shared concepts using generic examples. Platform-specific syntax is covered in Modules 4A and 4B.

### 4.1 Basic Structure

A pipeline file is a YAML file committed to your repo. YAML uses indentation (spaces, not tabs) to define structure. A basic pipeline has:

- A trigger definition
- One or more jobs
- Steps inside each job

```yaml
# Generic pipeline structure (concepts only; not runnable as-is)

trigger:
  - main                  # Run when code is pushed to main

jobs:
  - job: run_checks
    steps:
      - step: install dependencies
      - step: run linter
      - step: run tests
```

### 4.2 YAML Formatting Rules

YAML is strict about formatting. Common mistakes that cause pipeline failures:

| Mistake | Example | Fix |
|---|---|---|
| Using tabs instead of spaces | `	name: value` | Always use spaces |
| Inconsistent indentation | Mixing 2-space and 4-space | Pick one and be consistent |
| Missing space after colon | `name:value` | Always `name: value` |
| Wrong data type | `number: "42"` when integer expected | `number: 42` |

A YAML linter (available as a VS Code extension) catches these before you push.

### 4.3 Variables in YAML

Variables let you avoid hardcoding values in your pipeline:

```yaml
variables:
  environment: dev
  region: southeastasia

steps:
  - script: echo "Deploying to $(environment) in $(region)"
```

Variables can be defined at the pipeline level, stage level, or job level. They can also be passed in at runtime or pulled from a secret store.

### 4.4 Conditions

You can control whether a step or job runs based on a condition:

```yaml
# Only run this step if the previous step succeeded
condition: succeeded()

# Only run on the main branch
condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')

# Always run (even if previous steps failed); useful for cleanup
condition: always()
```

### 4.5 Multi-Stage Example (Conceptual)

```yaml
stages:
  - stage: CI
    jobs:
      - job: test
        steps:
          - script: echo "Running tests"

  - stage: CD
    dependsOn: CI          # Only runs if CI stage succeeded
    condition: succeeded()
    jobs:
      - job: deploy
        steps:
          - script: echo "Deploying"
```

---

## 5. Secrets and Environment Variables in Pipelines

### 5.1 Why Never Hardcode Secrets

Never put passwords, API keys, connection strings, or tokens directly in your pipeline YAML file. YAML files are committed to Git; anyone with repo access can see them. Even if you delete the value later, it remains in Git history.

```yaml
# WRONG - never do this
steps:
  - script: python deploy.py --password "mySecretPassword123"
```

### 5.2 How to Handle Secrets

Secrets are stored outside the pipeline file and injected at runtime as environment variables. The pipeline only references the name of the secret, never the value.

```yaml
# CORRECT - reference a secret by name
steps:
  - script: python deploy.py
    env:
      DB_PASSWORD: $(db-password)    # Azure Pipelines syntax
```

Where secrets come from depends on the platform:

| Platform | Secret store |
|---|---|
| Azure Pipelines | Azure Key Vault linked variable group, or pipeline secret variables |
| GitHub Actions | GitHub Secrets (repository or environment level) |
| Terraform Cloud | Workspace environment variables (marked sensitive) |

### 5.3 Environment Variables vs Pipeline Variables

- **Environment variables:** Available to the operating system and any process running in a step. Set with `env:` in the step definition.
- **Pipeline variables:** Available within the pipeline YAML itself using `$(variable-name)` syntax. Can be promoted to environment variables when needed.

Secrets should always be passed as environment variables to the step that needs them; never as command-line arguments (which appear in logs).

---

## 6. CI vs CD: When to Use Each

A common question is what belongs in CI and what belongs in CD. Here is a practical guide for data engineering projects:

### Continuous Integration (runs on every PR)

The goal of CI is to catch problems before they reach the main branch. CI should be fast (ideally under 10 minutes) so developers get feedback quickly.

| Check | Why |
|---|---|
| Code linting | Catches syntax errors and style violations |
| Unit tests | Verifies individual functions work correctly |
| `terraform plan` | Shows what infrastructure changes the PR will make |
| Schema validation | Catches breaking changes to data models early |
| Security scanning | Flags hardcoded secrets or known vulnerabilities |

### Continuous Delivery (runs on merge to main)

The goal of CD is to get approved changes to their destination reliably. CD steps are allowed to take longer.

| Step | Why |
|---|---|
| Build and package | Creates a deployable artifact |
| Deploy to staging | Verifies the change works in a production-like environment |
| Integration tests | Tests against real infrastructure |
| Manual approval gate | A human confirms before deploying to production |
| Deploy to production | The actual release |

---

## 7. Common Pipeline Patterns for Data Projects

### Pattern 1: Lint and Test on PR

The simplest and most universal pattern. Every PR triggers a lint check and test run. Nothing is deployed until the PR is merged.

```
PR opened
  --> lint Python files
  --> run unit tests
  --> post result to PR
Merge
  --> deploy
```

### Pattern 2: Terraform Plan on PR; Apply on Merge

Used when your repo contains Terraform infrastructure code alongside application code.

```
PR opened
  --> terraform init
  --> terraform plan
  --> post plan output as PR comment
Merge to main
  --> terraform apply
```

### Pattern 3: Multi-Environment Promotion

Used on mature client projects with dev; staging; production environments.

```
Merge to main
  --> deploy to dev (automatic)
  --> run integration tests on dev
  --> manual approval
  --> deploy to staging (automatic after approval)
  --> manual approval
  --> deploy to production
```

### Pattern 4: Scheduled Pipeline

Used for recurring data jobs like daily ingestion runs or weekly reports. Not triggered by code changes; triggered by a schedule.

```
Every day at 6:00 AM
  --> run ingestion script
  --> validate row counts
  --> send success/failure alert
```

---

## 8. Lab: Read and Annotate a CI/CD Pipeline

This lab builds your understanding of YAML pipeline structure before you write one from scratch in Module 4A or 4B. You will read a complete working pipeline, identify its parts, and answer questions about what each section does.

No platform access is needed for this exercise. It is a reading and annotation exercise.

---

### The Pipeline

Read through this complete GitHub Actions workflow carefully:

```yaml
name: Data Pipeline CI

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

env:
  PYTHON_VERSION: "3.11"

jobs:
  lint-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: pip install flake8 pytest

      - name: Lint
        run: flake8 src/

      - name: Test
        run: python -m pytest tests/

  deploy:
    runs-on: ubuntu-latest
    needs: lint-and-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy
        run: echo "Deploying to production"
        env:
          API_KEY: ${{ secrets.DEPLOY_API_KEY }}
```

---

### Questions

Answer these in your notes before moving on. The answers are in the section below.

1. How many **jobs** does this pipeline have? What are their names?
2. What are the two **triggers** for this pipeline? Under what conditions does each fire?
3. Which job runs first? How does the pipeline enforce this order?
4. The `deploy` job has an `if:` condition. In plain English, what does it mean?
5. Where is `PYTHON_VERSION` defined and how is it referenced inside a step?
6. The `API_KEY` is a secret. Where is its value stored? Does it appear in this file?
7. If the `lint-and-test` job fails, what happens to the `deploy` job?
8. Which step in `lint-and-test` checks out the code from the repo? Why is this always needed as the first step?

---

### Answers

1. Two jobs: `lint-and-test` and `deploy`.
2. Two triggers: `pull_request` (fires when a PR is opened or updated targeting `main`) and `push` (fires when code is pushed directly to `main`, which happens when a PR is merged).
3. `lint-and-test` runs first because `deploy` has `needs: lint-and-test`; it will not start until `lint-and-test` completes successfully.
4. The `deploy` job only runs if the trigger was a push event AND the branch is `main`. This means it runs on merge to main but NOT on PRs.
5. `PYTHON_VERSION` is defined in the `env:` block at the top of the workflow and referenced as `${{ env.PYTHON_VERSION }}` inside the step.
6. The value of `API_KEY` is stored in GitHub Secrets; a secure store outside the YAML file. The file only references its name (`secrets.DEPLOY_API_KEY`), never its value.
7. The `deploy` job is skipped entirely; `needs:` means the deploy job only starts if its dependency succeeded.
8. The `actions/checkout@v4` step clones the repository onto the runner machine. Without it, the runner has an empty workspace and cannot find any of your files.

---

### Next Step

You will write your own version of this pipeline from scratch in the first lab of Module 4A (Azure Pipelines) or Module 4B (GitHub Actions). Having read and understood this one, writing it will feel much more natural.

---

## 9. Quick Reference

| Concept | Description |
|---|---|
| CI | Automated checks on every PR; lint, test, validate |
| CD | Automated deployment after merge |
| Trigger | What causes the pipeline to run |
| Agent/Runner | The machine executing the pipeline |
| Stage | A major phase of the pipeline |
| Job | A unit of work running on one agent |
| Step | A single action within a job |
| Artifact | A file passed between pipeline stages |
| Variable | A named value used in the pipeline |
| Secret | A sensitive variable stored outside the YAML file |
| Condition | A rule controlling whether a step/job runs |

---

*Next module: Module 4 (Azure): CI/CD with Azure Pipelines or Module 4 (non-Azure): CI/CD with GitHub Actions*
