# Module 4 (Azure): CI/CD with Azure Pipelines

> **Track:** Core Track
> **Duration:** 3–4 days
> **Level:** Beginner
> **Prerequisites:** Module 3: CI/CD Fundamentals
> **Note:** Complete either Module 4A or Module 4B based on your client platform. Both cover equivalent core skills.

---

## Table of Contents

1. [Azure Pipelines Overview](#1-azure-pipelines-overview)
2. [Pipeline YAML Structure](#2-pipeline-yaml-structure)
3. [Built-in Tasks](#3-built-in-tasks)
4. [Multi-Stage Pipelines](#4-multi-stage-pipelines)
5. [Pipeline Variables and Variable Groups](#5-pipeline-variables-and-variable-groups)
6. [Service Connections](#6-service-connections)
7. [Environment Approvals and Gates](#7-environment-approvals-and-gates)
8. [Triggering Pipelines](#8-triggering-pipelines)
9. [Debugging Failed Runs](#9-debugging-failed-runs)
10. [Lab: Build a Multi-Stage Pipeline for a Python Project](#10-lab-build-a-multi-stage-pipeline-for-a-python-project)
11. [Quick Reference](#11-quick-reference)

---

## 1. Azure Pipelines Overview

Azure Pipelines is the CI/CD service inside Azure DevOps. It runs automated pipelines defined in YAML files committed to your repo. When a trigger fires (a PR, a push, a schedule), Azure Pipelines picks up the YAML file and executes it on an agent.

Key characteristics:

- **YAML-based:** Pipelines are defined as code, versioned alongside your project
- **Microsoft-hosted agents:** No infrastructure to manage; agents are fresh VMs per run
- **Deep Azure integration:** Native connections to Azure Key Vault, Azure Container Registry, Azure Kubernetes Service, and more
- **Multi-stage:** One pipeline file can define the entire journey from code to production

---

## 2. Pipeline YAML Structure

### 2.1 Minimal Pipeline

```yaml
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - script: echo "Hello from Azure Pipelines"
    displayName: Say hello
```

Every pipeline needs at minimum: a trigger, a pool (agent), and steps.

### 2.2 Full Structure Reference

```yaml
trigger:                          # What branch pushes trigger this pipeline
  branches:
    include:
      - main
      - develop

pr:                               # What PR targets trigger this pipeline
  branches:
    include:
      - main
      - develop

variables:                        # Pipeline-level variables
  pythonVersion: "3.11"
  environment: dev

pool:                             # Agent pool configuration
  vmImage: ubuntu-latest          # Microsoft-hosted Ubuntu agent

stages:
  - stage: CI
    displayName: Continuous Integration
    jobs:
      - job: RunTests
        displayName: Run Tests
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: $(pythonVersion)
            displayName: Set Python version

          - script: pip install flake8 pytest
            displayName: Install dependencies

          - script: flake8 .
            displayName: Lint with flake8

          - script: python -m pytest tests/
            displayName: Run tests

  - stage: CD
    displayName: Continuous Delivery
    dependsOn: CI
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: Deploy
        displayName: Deploy to dev
        steps:
          - script: echo "Deploying to $(environment)"
            displayName: Deploy
```

---

## 3. Built-in Tasks

Azure Pipelines has a library of built-in tasks that abstract common operations. Tasks are more reliable than raw scripts for platform integrations because they handle authentication and error handling automatically.

### Commonly Used Tasks

**UsePythonVersion:** Set the Python version on the agent

```yaml
- task: UsePythonVersion@0
  inputs:
    versionSpec: "3.11"
    addToPath: true
  displayName: Use Python 3.11
```

**Bash / Script:** Run shell commands

```yaml
- script: |
    pip install -r requirements.txt
    python -m pytest tests/ --junitxml=test-results.xml
  displayName: Install and test
```

**CopyFiles:** Copy files to a staging directory

```yaml
- task: CopyFiles@2
  inputs:
    sourceFolder: $(Build.SourcesDirectory)
    contents: "**/*.py"
    targetFolder: $(Build.ArtifactStagingDirectory)
  displayName: Copy Python files
```

**PublishBuildArtifacts:** Publish files so they can be downloaded or used by later stages

```yaml
- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: $(Build.ArtifactStagingDirectory)
    artifactName: python-package
  displayName: Publish artifact
```

**DownloadBuildArtifacts:** Download artifacts published by a previous stage

```yaml
- task: DownloadBuildArtifacts@1
  inputs:
    artifactName: python-package
    downloadPath: $(System.ArtifactsDirectory)
  displayName: Download artifact
```

### Predefined Variables

Azure Pipelines provides useful built-in variables you can reference in your pipeline:

| Variable | Value |
|---|---|
| `Build.SourceBranch` | Full ref of the branch (e.g., `refs/heads/main`) |
| `Build.SourceBranchName` | Short branch name (e.g., `main`) |
| `Build.BuildId` | Unique ID for this pipeline run |
| `Build.Repository.Name` | Name of the repo |
| `System.DefaultWorkingDirectory` | Root folder where the repo is checked out |
| `Build.ArtifactStagingDirectory` | Temp folder for artifacts |

---

## 4. Multi-Stage Pipelines

Multi-stage pipelines are the standard pattern for professional data engineering deployments. A single YAML file defines the full journey from code to production.

### 4.1 Stage Dependencies

```yaml
stages:
  - stage: CI
    jobs:
      - job: Test
        steps:
          - script: python -m pytest tests/

  - stage: DeployStaging
    dependsOn: CI
    condition: succeeded()
    jobs:
      - job: Deploy
        steps:
          - script: echo "Deploy to staging"

  - stage: DeployProduction
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployProd
        environment: production       # Links to an Azure Pipelines Environment with approvals
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy to production"
```

### 4.2 Passing Data Between Stages

Use pipeline artifacts to pass files between stages:

```yaml
stages:
  - stage: Build
    jobs:
      - job: Package
        steps:
          - script: zip -r app.zip src/
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: app.zip
              artifactName: app-package

  - stage: Deploy
    dependsOn: Build
    jobs:
      - job: Release
        steps:
          - task: DownloadBuildArtifacts@1
            inputs:
              artifactName: app-package
          - script: echo "Deploying app-package"
```

---

## 5. Pipeline Variables and Variable Groups

### 5.1 Inline Variables

Defined directly in the YAML:

```yaml
variables:
  environment: dev
  region: southeastasia
  pythonVersion: "3.11"
```

### 5.2 Variable Groups

Variable groups are collections of variables defined once in Azure DevOps and reused across multiple pipelines. Useful for shared config like environment names, resource names, or non-secret settings.

Create a variable group:
1. Go to **Pipelines; Library**
2. Click **Variable Group**
3. Add key-value pairs
4. Save with a name (e.g., `de-practice-dev-config`)

Reference in your pipeline:

```yaml
variables:
  - group: de-practice-dev-config   # Imports all variables from this group
  - name: pythonVersion
    value: "3.11"                   # Inline variable alongside the group
```

### 5.3 Linking Azure Key Vault to a Variable Group

For secrets, link a variable group directly to Azure Key Vault:

1. Create a variable group
2. Enable **Link secrets from an Azure key vault as variables**
3. Select your Key Vault and add the secret names you want to expose
4. The pipeline can then reference them as `$(secret-name)`

This means secrets never leave Key Vault; they are injected at runtime and masked in logs.

### 5.4 Secret Variables in the Pipeline UI

For quick secrets without Key Vault:

1. Go to your pipeline; click **Edit**
2. Click **Variables** (top right)
3. Add a variable; tick **Keep this value secret**
4. Reference as `$(variable-name)` in your YAML

Marked-as-secret variables are masked in all pipeline logs.

---

## 6. Service Connections

A service connection is a stored, authenticated connection from Azure Pipelines to an external service. It lets your pipeline interact with Azure resources, container registries, or other platforms without embedding credentials in the YAML.

### Creating a Service Connection

1. Go to **Project Settings** (bottom left in Azure DevOps)
2. Click **Service Connections**
3. Click **New service connection**
4. Choose the type (Azure Resource Manager, Docker Registry, etc.)
5. Follow the authentication prompts

### Common Service Connection Types

**Azure Resource Manager:** Connects to an Azure subscription. Used for deploying resources, accessing Key Vault, pushing to storage, etc.

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: my-azure-service-connection   # Name of the service connection
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az storage blob upload \
        --account-name $(storageAccount) \
        --container-name data \
        --file output.csv \
        --name output.csv
  displayName: Upload to Azure Storage
```

**Docker Registry:** Connects to Azure Container Registry or Docker Hub for pushing and pulling images.

---

## 7. Environment Approvals and Gates

An Environment in Azure Pipelines is a named deployment target (e.g., `staging`, `production`) that can have approval requirements attached to it.

### Creating an Environment

1. Go to **Pipelines; Environments**
2. Click **New environment**
3. Give it a name (e.g., `production`)
4. Click on the environment; go to **Approvals and checks**
5. Add an **Approvals** check; specify who must approve before the pipeline continues

### Using an Environment in a Pipeline

```yaml
- stage: DeployProduction
  jobs:
    - deployment: ReleaseToProd
      environment: production         # Pipeline pauses here for approval
      strategy:
        runOnce:
          deploy:
            steps:
              - script: echo "Deploying to production"
```

When the pipeline reaches this stage, it pauses and sends a notification to the approvers. The pipeline only continues after an approver clicks **Approve** in the Azure DevOps UI.

---

## 8. Triggering Pipelines

### Push Trigger

```yaml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/**         # Only trigger if files in src/ changed
    exclude:
      - docs/**        # Never trigger for docs changes
```

### PR Trigger

```yaml
pr:
  branches:
    include:
      - main
  paths:
    include:
      - src/**
```

### Scheduled Trigger

```yaml
schedules:
  - cron: "0 6 * * *"           # Every day at 6:00 AM UTC
    displayName: Daily 6AM run
    branches:
      include:
        - main
    always: true                 # Run even if no code changed
```

### Disabling Automatic Triggers

```yaml
trigger: none    # Only run manually or when called from another pipeline
pr: none
```

---

## 9. Debugging Failed Runs

### Reading Pipeline Logs

When a pipeline fails, click on the failed job in the Azure DevOps UI. Each step is expandable; the failed step will show the error output. Look for:

- The exact command that failed
- The error message and exit code
- Any missing environment variables or files

### Common Failure Causes

| Symptom | Likely cause |
|---|---|
| `command not found` | Tool not installed on agent; add an install step |
| `Permission denied` | Service connection missing a role on the Azure resource |
| `No such file or directory` | Wrong working directory; check `workingDirectory` input |
| Secret shows as `***` in error | The secret value itself is wrong; update it in the variable group or Key Vault |
| Pipeline not triggering on PR | Check branch filters in the `pr:` block |

### Re-running a Pipeline

You can re-run a failed pipeline from the Azure DevOps UI without pushing new code:

1. Open the failed run
2. Click **Rerun failed jobs** (to retry only failed jobs) or **Run new** (to start fresh)

### Enabling Debug Logging

Add this variable to get verbose output:

```yaml
variables:
  system.debug: true
```

---

## 10. Lab: Build a Multi-Stage Pipeline for a Python Project

This lab has two parts. Part A is a short warm-up that completes the pipeline you read in Module 3. Part B builds the full multi-stage pipeline.

### Part A: Write Your First Azure Pipeline (Warm-Up)

In Module 3 you read and annotated a GitHub Actions workflow. Now write the Azure Pipelines equivalent from scratch using what you learned in this module.

**Goal:** A pipeline that triggers on every PR to `main`, installs Python, runs `flake8` on a Python file, and fails the pipeline if linting fails.

Create `.azure-pipelines/ci.yml` in your practice repo:

```yaml
trigger: none   # No push trigger for this warm-up

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: "3.11"
    displayName: Set up Python

  - script: pip install flake8
    displayName: Install flake8

  - script: flake8 src/
    displayName: Lint with flake8
```

Create `src/hello.py`:

```python
def greet(name):
    message = "Hello, " + name + "!"
    return message

print(greet("Data Engineer"))
```

Register the pipeline in Azure DevOps (Pipelines; New Pipeline; select your repo and point to this file). Push on a feature branch, open a PR, and watch it trigger. Then introduce a linting error, push again, and confirm the pipeline fails.

Once it is green, proceed to Part B.

---

### Part B: Full Multi-Stage Pipeline

### Prerequisites
- Part A of this lab completed
- practice repo from Module 1
- A service connection to an Azure subscription (ask your team lead)

---

### Step 1: Prepare Your Repo

In your practice repo, create the following files:

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
```

Commit and push these to your feature branch.

---

### Step 2: Create the Pipeline YAML

Create a file at `.azure-pipelines/ci-cd.yml`:

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
  pythonVersion: "3.11"

stages:
  - stage: CI
    displayName: Continuous Integration
    jobs:
      - job: LintAndTest
        displayName: Lint and Test
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: $(pythonVersion)
            displayName: Use Python $(pythonVersion)

          - script: pip install -r requirements.txt
            displayName: Install dependencies

          - script: flake8 src/ tests/
            displayName: Lint with flake8

          - script: python -m pytest tests/ --junitxml=test-results.xml
            displayName: Run tests

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: JUnit
              testResultsFiles: test-results.xml
            condition: always()
            displayName: Publish test results

  - stage: CD
    displayName: Continuous Delivery
    dependsOn: CI
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: Deploy
        displayName: Deploy
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: |
              echo "Deploying build $(Build.BuildId) to $(environment)"
              echo "Source branch: $(Build.SourceBranchName)"
            displayName: Simulate deployment
```

---

### Step 3: Register the Pipeline in Azure DevOps

1. Go to **Pipelines; Pipelines**
2. Click **New pipeline**
3. Select **Azure Repos Git**
4. Select your repository
5. Select **Existing Azure Pipelines YAML file**
6. Select `.azure-pipelines/ci-cd.yml`
7. Click **Continue** then **Save** (do not run yet)

---

### Step 4: Test the CI Stage with a PR

Create a feature branch, make a small change to `src/hello.py`, push, and open a PR targeting `main`.

The pipeline should trigger automatically and run the CI stage. Check the results in **Pipelines; Pipelines** and in the PR itself (it shows the pipeline status).

---

### Step 5: Add a Secret from Key Vault

1. In Azure DevOps go to **Pipelines; Library**
2. Create a variable group called `de-practice-secrets`
3. Enable **Link secrets from an Azure key vault as variables**
4. Link to your Key Vault and add a secret (e.g., `db-connection-string`)
5. Update your pipeline to use the variable group:

```yaml
variables:
  - group: de-practice-secrets
  - name: pythonVersion
    value: "3.11"
```

6. Add a step that uses the secret (masked in logs):

```yaml
- script: echo "Connected using secret (value is masked)"
  env:
    CONNECTION: $(db-connection-string)
  displayName: Use secret from Key Vault
```

Push the change and verify the pipeline runs without exposing the secret value in logs.

---

### Step 6: Merge and Observe CD Stage

Merge your PR to `main`. The pipeline should trigger again and this time run both stages; CI first, then CD (the deploy stage).

Confirm in the pipeline run view that the CD stage only ran after CI succeeded and that it only triggered on the merge to `main`, not on the PR.

---

## 11. Quick Reference

| Concept | YAML key |
|---|---|
| Push trigger | `trigger: branches: include:` |
| PR trigger | `pr: branches: include:` |
| Schedule trigger | `schedules: cron:` |
| Agent pool | `pool: vmImage:` |
| Stage | `stages: - stage:` |
| Job | `jobs: - job:` |
| Step (script) | `- script:` |
| Step (task) | `- task: TaskName@version` |
| Stage dependency | `dependsOn:` |
| Stage condition | `condition: succeeded()` |
| Variable group | `variables: - group:` |
| Secret variable | Defined in Library or Key Vault; referenced as `$(name)` |
| Service connection | Referenced in task `inputs: azureSubscription:` |
| Environment approval | `- deployment:` with `environment:` |

---

*Next module: Module 5: Environment & Secret Management (skip Module 4B if you completed 4A)*
