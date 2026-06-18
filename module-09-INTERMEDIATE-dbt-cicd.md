# Module 9: Advanced dbt CI/CD Patterns

> **Track:** Intermediate Extension
> **Duration:** 2–3 days
> **Level:** Intermediate
> **Prerequisites:** Module 4A or 4B, Module 7: Containers for Data Engineers
> **Audience:** DEs working on dbt-heavy client projects

> **Prerequisites check:** This module assumes you have already built and run a dbt project; you should be comfortable with `profiles.yml`, `dbt_project.yml`, models, and the `target/` folder before starting. If you have not used dbt before, complete the free [dbt Fundamentals course](https://learn.getdbt.com/courses/dbt-fundamentals) first, then return here.

---

## Table of Contents

1. [Why dbt Needs Special CI/CD Patterns](#1-why-dbt-needs-special-cicd-patterns)
2. [dbt Environments and Target Configs](#2-dbt-environments-and-target-configs)
3. [dbt Artifacts and State](#3-dbt-artifacts-and-state)
4. [Slim CI: Running Only What Changed](#4-slim-ci-running-only-what-changed)
5. [dbt Tests as Pipeline Gates](#5-dbt-tests-as-pipeline-gates)
6. [dbt Cloud vs Self-Hosted Pipelines](#6-dbt-cloud-vs-self-hosted-pipelines)
7. [Deploying to Multiple Targets](#7-deploying-to-multiple-targets)
8. [Lab: Configure a Slim CI Pipeline for a dbt Project](#8-lab-configure-a-slim-ci-pipeline-for-a-dbt-project)
9. [Quick Reference](#9-quick-reference)

---

## 1. Why dbt Needs Special CI/CD Patterns

dbt (data build tool) manages SQL transformation models; sometimes hundreds or thousands of them in a single project. Running a standard CI/CD pipeline that executes all models on every PR does not scale:

- A full `dbt build` on a large project can take 30 minutes or more
- Running all models on every PR means waiting a long time for feedback
- Running all models in a CI environment can be expensive in cloud compute costs
- Tests that fail on unrelated models slow down unrelated PRs

dbt-specific CI/CD patterns solve these problems by:
- Running only the models that changed (Slim CI)
- Using lightweight CI environments with only the data needed for testing
- Treating dbt test failures as pipeline gate conditions
- Separating the CI (test) and CD (deploy) concerns cleanly

---

## 2. dbt Environments and Target Configs

### 2.1 How dbt Targets Work

dbt uses a `profiles.yml` file to define connection targets. A target specifies which database and schema to write to. You define separate targets for each environment:

```yaml
# profiles.yml
my_project:
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: DEV_DB
      schema: "dbt_{{ env_var('DBT_USER', 'dev') }}"   # Personal dev schema
      warehouse: DEV_WH

    ci:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: CI_DB
      schema: "ci_{{ env_var('PR_NUMBER', 'unknown') }}"   # Isolated per PR
      warehouse: CI_WH

    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: PROD_DB
      schema: analytics
      warehouse: PROD_WH

  target: dev   # Default target for local development
```

### 2.2 Environment Variable Injection

Notice that `profiles.yml` uses `env_var()` instead of hardcoded values. This means:

- No credentials in the file itself (safe to commit)
- The pipeline injects the right credentials at runtime per environment
- The same `profiles.yml` works locally, in CI, and in production

In your pipeline, set environment variables before running dbt:

**Azure Pipelines:**
```yaml
- script: dbt run --target ci
  env:
    SNOWFLAKE_ACCOUNT: $(snowflake-account)
    SNOWFLAKE_USER: $(snowflake-user)
    SNOWFLAKE_PASSWORD: $(snowflake-password)
    PR_NUMBER: $(System.PullRequest.PullRequestNumber)
  displayName: Run dbt models
```

**GitHub Actions:**
```yaml
- name: Run dbt models
  run: dbt run --target ci
  env:
    SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
    SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
    SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
    PR_NUMBER: ${{ github.event.pull_request.number }}
```

### 2.3 Per-PR Schema Isolation

Notice the CI target uses a schema like `ci_{{ PR_NUMBER }}`. This is a best practice; each PR gets its own isolated schema in the CI database. Benefits:

- Multiple PRs can run CI simultaneously without interfering with each other
- Failed or stale CI runs do not affect other PRs
- Schemas can be cleaned up automatically after the PR is merged or closed

---

## 3. dbt Artifacts and State

### 3.1 What are dbt Artifacts

Every `dbt run`, `dbt test`, or `dbt build` command produces artifact files in the `target/` folder:

| File | Contains |
|---|---|
| `manifest.json` | Full graph of all models, tests, and sources; their dependencies and config |
| `run_results.json` | Results of the last run; which models succeeded, failed, and their timing |
| `catalog.json` | Metadata about the database objects dbt created |

These files are the basis for Slim CI and observability.

### 3.2 Using Artifacts for State Comparison

Slim CI works by comparing two manifests:

1. The **production manifest** (`manifest.json` from the last successful prod run); represents the current state of production
2. The **PR manifest** (generated when dbt compiles the PR branch); represents the proposed changes

dbt compares the two and identifies which models have changed. Only those models (and their downstream dependencies) are run.

To make this work, you need to:
1. Store the production `manifest.json` somewhere accessible (cloud storage or dbt Cloud)
2. Download it at the start of the CI run
3. Pass it to dbt using the `--state` flag

---

## 4. Slim CI: Running Only What Changed

### 4.1 The `state:modified` Selector

dbt's state selector compares the current project against a previously built state. There are two variants and the difference matters:

```bash
# Run only models that have changed; no downstream models
dbt build --select state:modified --state ./prod-state/

# Run modified models AND all downstream models that depend on them (the + suffix)
dbt build --select state:modified+ --state ./prod-state/
```

The `+` suffix means "and everything downstream." For example, if model B depends on model A and you modify model A, `state:modified` runs only A while `state:modified+` runs both A and B.

In practice, **always use `state:modified+`** for CI runs. If you only run the changed model and skip its dependents, you may deploy a change that breaks a downstream model without knowing it.

### 4.2 Full Slim CI Implementation

**Step 1:** After every successful production run, save the `manifest.json` to cloud storage.

**Azure Pipelines (production run):**
```yaml
- script: dbt build --target prod
  displayName: Run dbt production

- task: AzureCLI@2
  inputs:
    azureSubscription: my-service-connection
    scriptType: bash
    inlineScript: |
      az storage blob upload \
        --account-name mystorageaccount \
        --container-name dbt-state \
        --file target/manifest.json \
        --name prod/manifest.json \
        --overwrite
  displayName: Save production manifest
```

**Step 2:** In CI, download the production manifest and use it for state comparison.

**Azure Pipelines (CI run on PR):**
```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: my-service-connection
    scriptType: bash
    inlineScript: |
      mkdir -p prod-state
      az storage blob download \
        --account-name mystorageaccount \
        --container-name dbt-state \
        --name prod/manifest.json \
        --file prod-state/manifest.json
  displayName: Download production manifest

- script: |
    dbt deps
    dbt build \
      --select state:modified+ \
      --state prod-state/ \
      --target ci
  env:
    SNOWFLAKE_ACCOUNT: $(snowflake-account)
    SNOWFLAKE_USER: $(snowflake-user)
    SNOWFLAKE_PASSWORD: $(snowflake-password)
    PR_NUMBER: $(System.PullRequest.PullRequestNumber)
  displayName: Run Slim CI

- script: dbt test --select state:modified+ --state prod-state/ --target ci
  displayName: Run tests on modified models
```

**GitHub Actions (CI run on PR):**
```yaml
jobs:
  slim-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dbt
        run: pip install dbt-snowflake

      - name: Configure AWS and download prod manifest
        run: |
          aws s3 cp s3://my-bucket/dbt-state/prod/manifest.json prod-state/manifest.json
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ap-southeast-1

      - name: Run Slim CI
        run: |
          dbt deps
          dbt build \
            --select state:modified+ \
            --state prod-state/ \
            --target ci
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
```

---

## 5. dbt Tests as Pipeline Gates

dbt has built-in tests (not_null, unique, accepted_values, relationships) and supports custom tests. In a CI/CD context, test failures should block the PR from merging.

### 5.1 Separating Build and Test Steps

```bash
# Run models first
dbt run --select state:modified+ --state prod-state/ --target ci

# Then run tests; pipeline fails if any test fails
dbt test --select state:modified+ --state prod-state/ --target ci
```

If `dbt test` exits with a non-zero code (which it does when tests fail), the pipeline step fails and blocks the PR.

### 5.2 dbt build vs dbt run + dbt test

`dbt build` combines `run` and `test` in the correct dependency order:

```bash
# This is equivalent to running models and tests in dependency order
dbt build --select state:modified+ --state prod-state/ --target ci
```

Use `dbt build` when you want tests to block downstream model runs (a model that depends on a failing model is not run). Use `dbt run` followed by `dbt test` when you want all models to run regardless and then check all tests at the end.

### 5.3 Capturing and Reporting Test Results

Parse `run_results.json` to post a summary as a PR comment:

**GitHub Actions:**
```yaml
- name: Parse dbt results
  if: always()
  run: |
    python - <<'EOF'
    import json

    with open("target/run_results.json") as f:
        results = json.load(f)

    passed = sum(1 for r in results["results"] if r["status"] == "pass")
    failed = sum(1 for r in results["results"] if r["status"] == "fail")
    errored = sum(1 for r in results["results"] if r["status"] == "error")

    print(f"dbt results: {passed} passed, {failed} failed, {errored} errored")

    if failed > 0 or errored > 0:
        print("FAILED TESTS:")
        for r in results["results"]:
            if r["status"] in ("fail", "error"):
                print(f"  - {r['unique_id']}: {r['status']}")
    EOF
  displayName: Summarize dbt test results
```

---

## 6. dbt Cloud vs Self-Hosted Pipelines

### 6.1 dbt Cloud Jobs

dbt Cloud has a built-in CI/CD job feature. When connected to GitHub or Azure Repos, it can:

- Trigger a CI run automatically when a PR is opened
- Run Slim CI natively (it stores the production manifest internally)
- Post results back to the PR as a status check
- Run production jobs on a schedule

dbt Cloud CI is the easiest path if your client is already on dbt Cloud.

**When to use dbt Cloud CI:**
- Client has dbt Cloud Business or Enterprise tier
- You want minimal pipeline configuration
- The client's data platform is one of dbt Cloud's supported adapters

### 6.2 Self-Hosted Pipeline Approach

If the client uses Azure Pipelines or GitHub Actions (not dbt Cloud CI), you set up dbt to run inside the pipeline as shown in this module. This gives more control and integrates with existing pipeline infrastructure.

**When to use self-hosted:**
- Client uses Azure Pipelines or GitHub Actions for everything else
- Client is on dbt Core (open source); no dbt Cloud subscription
- You need custom steps before or after dbt runs (data quality checks, notifications)

---

## 7. Deploying to Multiple Targets

Some clients have their data platform on Azure (Databricks) for some projects and AWS (Redshift) for others. A single dbt project can be configured to deploy to both by parameterizing the target.

### 7.1 Multiple Profiles

```yaml
# profiles.yml
my_project:
  outputs:
    prod_databricks:
      type: databricks
      host: "{{ env_var('DATABRICKS_HOST') }}"
      http_path: "{{ env_var('DATABRICKS_HTTP_PATH') }}"
      token: "{{ env_var('DATABRICKS_TOKEN') }}"
      schema: analytics

    prod_redshift:
      type: redshift
      host: "{{ env_var('REDSHIFT_HOST') }}"
      user: "{{ env_var('REDSHIFT_USER') }}"
      password: "{{ env_var('REDSHIFT_PASSWORD') }}"
      port: 5439
      dbname: analytics
      schema: analytics
```

### 7.2 Parameterizing the Target in a Pipeline

**Azure Pipelines:**
```yaml
parameters:
  - name: target
    type: string
    default: prod_databricks
    values:
      - prod_databricks
      - prod_redshift

steps:
  - script: dbt run --target ${{ parameters.target }}
    displayName: Run dbt on ${{ parameters.target }}
```

**GitHub Actions (manual dispatch with target selection):**
```yaml
on:
  workflow_dispatch:
    inputs:
      target:
        description: dbt target
        required: true
        default: prod_databricks
        type: choice
        options:
          - prod_databricks
          - prod_redshift

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dbt run --target ${{ github.event.inputs.target }}
```

---

## 8. Lab: Configure a Slim CI Pipeline for a dbt Project

This lab sets up a Slim CI pipeline for a dbt project that runs only modified models on PRs and does a full run on merge to main.

### Prerequisites
- A dbt project (any adapter; Snowflake, BigQuery, Databricks, or Redshift)
- Azure Pipelines or GitHub Actions set up (Module 4A or 4B)
- Cloud storage for storing the production manifest (Azure Blob or S3)
- dbt installed: `pip install dbt-<adapter>`

---

### Step 1: Configure Your profiles.yml

In your dbt project, update `profiles.yml` to use environment variables and define `dev`, `ci`, and `prod` targets as shown in Section 2.1.

Add `profiles.yml` to `.gitignore` if it contains any hardcoded values. The version in the repo should use only `env_var()` calls.

---

### Step 2: Create the Production Artifact Storage

**Azure:**
```powershell
az storage container create `
  --account-name <your-storage-account> `
  --name dbt-state
```

**AWS:**
```powershell
aws s3 mb s3://your-bucket-name/dbt-state/
```

---

### Step 3: Write the Production Pipeline

Create a pipeline file that runs `dbt build --target prod` on merge to main and saves the manifest afterward.

The key steps:
1. Install dbt
2. Run `dbt deps`
3. Run `dbt build --target prod`
4. Upload `target/manifest.json` to your storage location

Use the patterns from Modules 5A or 5B for the pipeline structure and credential injection.

---

### Step 4: Write the CI Pipeline

Create a pipeline file that triggers on PRs and runs Slim CI:

1. Install dbt
2. Download `prod/manifest.json` from storage to `prod-state/manifest.json`
3. Run `dbt deps`
4. Run `dbt build --select state:modified+ --state prod-state/ --target ci`
5. Parse and log results from `target/run_results.json`

---

### Step 5: Test the Slim CI Behavior

1. Run the production pipeline once manually to generate and store the first manifest
2. Create a feature branch; modify one dbt model
3. Open a PR; the CI pipeline should trigger
4. Check the pipeline logs; only the modified model (and its downstream models) should run; not the entire project

---

### Step 6: Verify Test Gating

Introduce a failing test in your modified model (e.g., add a `not_null` test on a column that has nulls in your CI data). Push to the PR. The pipeline should fail on the test step and the PR should be blocked from merging.

Fix the test or the data, push again, and confirm the pipeline goes green.

---

## 9. Quick Reference

| Command | What it does |
|---|---|
| `dbt run --target ci` | Run all models against the ci target |
| `dbt test --target ci` | Run all tests against the ci target |
| `dbt build --target ci` | Run models and tests in dependency order |
| `dbt deps` | Install dbt packages from `packages.yml` |
| `dbt compile` | Compile SQL without running |
| `dbt build --select state:modified+` | Run only modified models and their dependents |
| `dbt build --select state:modified+ --state ./prod-state/` | Slim CI with state comparison |
| `dbt ls --select state:modified+` | List which models would run in Slim CI |
| `target/manifest.json` | Full project graph; save after prod run |
| `target/run_results.json` | Results of last run; parse for CI reporting |

---

*Next module: Module 10: Monitoring, Observability & Alerting*
