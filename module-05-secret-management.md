# Module 5: Environment & Secret Management

> **Track:** Core Track
> **Duration:** 1–2 days
> **Level:** Beginner
> **Prerequisites:** Module 4A or Module 4B

---

## Table of Contents

1. [Why Secret Management Matters](#1-why-secret-management-matters)
2. [Environment Variables and .env Files](#2-environment-variables-and-env-files)
3. [Azure Key Vault](#3-azure-key-vault)
4. [AWS Secrets Manager and Parameter Store](#4-aws-secrets-manager-and-parameter-store)
5. [Managing Dev; Staging; and Production Configs](#5-managing-dev-staging-and-production-configs)
6. [Secret Rotation Basics](#6-secret-rotation-basics)
7. [Lab: Refactor Hardcoded Credentials to a Secret Store](#7-lab-refactor-hardcoded-credentials-to-a-secret-store)
8. [Quick Reference](#8-quick-reference)

---

> **Coming from Module 4A or 4B?** You have already seen secrets used inside pipelines; variable groups, Key Vault integration, GitHub Secrets, and secret masking in logs. This module does not repeat those. Instead it goes deeper on three things that were not covered there: reading secrets at runtime using the Python SDK (so your application code can fetch secrets directly, not just pipelines), multi-environment config separation, and secret rotation. If a section feels familiar, skim it and move on.

---

## 1. Why Secret Management Matters

Hardcoded credentials are one of the most common and most damaging mistakes made by engineers on client projects. A secret committed to a Git repository is permanently exposed; even after deletion, it remains in the commit history and can be retrieved by anyone with access to the repo.

Real consequences on client projects:
- Exposed database passwords giving attackers access to production data
- Leaked cloud credentials allowing unauthorized resource creation (and massive bills)
- Compliance violations (GDPR, SOC 2, ISO 27001) that put the client engagement at risk
- Emergency credential rotation at 2 AM while trying to find every place the secret was hardcoded

Proper secret management means: secrets are never in code, never in YAML pipeline files, and never printed to logs. They live in a dedicated secret store and are injected at runtime only when needed.

---

## 2. Environment Variables and .env Files

### 2.1 The Basic Pattern

The simplest form of secret management is environment variables. Instead of hardcoding a value in your script, you read it from the environment:

**Wrong:**
```python
# config.py
DB_PASSWORD = "my-super-secret-password"
DB_HOST = "prod-db.company.com"
```

**Correct:**
```python
# config.py
import os

DB_PASSWORD = os.environ["DB_PASSWORD"]
DB_HOST = os.environ["DB_HOST"]
```

The script no longer contains the secret. The secret is provided by whatever runs the script; a developer's local machine, a CI/CD pipeline, or a container.

### 2.2 .env Files for Local Development

For local development, engineers store their personal environment variables in a `.env` file:

```bash
# .env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=dev_database
DB_USER=dev_user
DB_PASSWORD=local-dev-password-only
```

Load `.env` files in Python using the `python-dotenv` library:

```python
from dotenv import load_dotenv
import os

load_dotenv()   # Reads .env file and sets environment variables

DB_PASSWORD = os.environ["DB_PASSWORD"]
```

### 2.3 Critical Rules for .env Files

- **Always add `.env` to `.gitignore`; no exceptions**
- `.env` files are for local development only; never use them in pipelines or production
- Create a `.env.example` file with placeholder values and commit that instead; it shows colleagues what variables are needed without exposing real values:

```bash
# .env.example  (safe to commit)
DB_HOST=your-database-host
DB_PORT=5432
DB_NAME=your-database-name
DB_USER=your-db-username
DB_PASSWORD=your-db-password
```

---

## 3. Azure Key Vault

Azure Key Vault is Microsoft's managed secret store for Azure. It stores secrets, encryption keys, and certificates centrally with access controlled by Azure Active Directory.

### 3.1 Key Vault Concepts

| Concept | Description |
|---|---|
| **Secret** | A string value with a name; connection strings, passwords, API keys |
| **Key** | A cryptographic key for encryption; usually managed by the service, not engineers |
| **Certificate** | TLS/SSL certificates with automatic renewal |
| **Access Policy / RBAC** | Controls who or what can read, write, or manage secrets |
| **Soft delete** | Deleted secrets can be recovered within a retention period |

### 3.2 Creating a Secret in Key Vault

**Via Azure Portal:**
1. Open your Key Vault resource
2. Click **Secrets; Generate/Import**
3. Set a name (use hyphens, not underscores: `db-password` not `db_password`)
4. Set the value
5. Click **Create**

**Via Azure CLI:**
```bash
az keyvault secret set \
  --vault-name my-key-vault \
  --name db-password \
  --value "my-secret-value"
```

**PowerShell:**
```powershell
az keyvault secret set `
  --vault-name my-key-vault `
  --name db-password `
  --value "my-secret-value"
```

### 3.3 Reading a Secret in Python

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import os

vault_url = os.environ["KEY_VAULT_URL"]   # e.g., https://my-key-vault.vault.azure.net/

credential = DefaultAzureCredential()
client = SecretClient(vault_url=vault_url, credential=credential)

db_password = client.get_secret("db-password").value
```

`DefaultAzureCredential` automatically uses the right authentication method depending on where the code runs:

| Where it runs | Authentication method used |
|---|---|
| Local machine | Your `az login` session |
| Azure VM / App Service | Managed Identity |
| GitHub Actions / Azure Pipelines | Service Principal via environment variables |

This means the same code works locally and in production without any changes.

### 3.4 Referencing Key Vault Secrets in Azure Pipelines

Link Key Vault to a variable group (covered in Module 4A):

```yaml
variables:
  - group: my-keyvault-variable-group   # Contains secrets from Key Vault

steps:
  - script: python deploy.py
    env:
      DB_PASSWORD: $(db-password)       # Injected from Key Vault at runtime
```

### 3.5 Key Vault Access Control

Access to Key Vault is controlled by Azure RBAC. On client projects, your pipeline's service principal needs the **Key Vault Secrets User** role to read secrets. As a consultant you will not usually set this up yourself; ask the client's Azure administrator.

---

## 4. AWS Secrets Manager and Parameter Store

AWS has two services for managing secrets. Knowing when to use each is important on AWS client projects.

### 4.1 AWS Secrets Manager

Secrets Manager is designed for secrets that need to be rotated automatically; database passwords, API keys, OAuth tokens.

**Key features:**
- Automatic secret rotation (e.g., rotate an RDS password every 30 days)
- Built-in integration with RDS, Redshift, and other AWS services
- Versioning; you can retrieve the current or previous version of a secret
- Higher cost than Parameter Store

**Creating a secret via AWS CLI:**
```bash
aws secretsmanager create-secret \
  --name de-practice/dev/db-password \
  --secret-string "my-secret-value" \
  --region ap-southeast-1
```

**PowerShell:**
```powershell
aws secretsmanager create-secret `
  --name de-practice/dev/db-password `
  --secret-string "my-secret-value" `
  --region ap-southeast-1
```

**Reading a secret in Python:**
```python
import boto3
import json
import os

client = boto3.client("secretsmanager", region_name="ap-southeast-1")

response = client.get_secret_value(SecretId="de-practice/dev/db-password")
secret = response["SecretString"]

# If the secret is a JSON object (common for database credentials):
# secret_dict = json.loads(secret)
# db_password = secret_dict["password"]
```

### 4.2 AWS Systems Manager Parameter Store

Parameter Store is designed for configuration values and non-rotating secrets. It is simpler and cheaper than Secrets Manager.

| | Secrets Manager | Parameter Store |
|---|---|---|
| **Use case** | Rotating credentials; database passwords | Config values; non-rotating secrets |
| **Cost** | Per secret per month | Free tier available; cost per API call |
| **Auto-rotation** | Yes | No |
| **Max value size** | 65 KB | 4 KB (Standard) / 8 KB (Advanced) |
| **Encryption** | Always encrypted | Optional (use SecureString type) |

**Creating a parameter:**
```bash
aws ssm put-parameter \
  --name "/de-practice/dev/db-host" \
  --value "my-db.cluster.ap-southeast-1.rds.amazonaws.com" \
  --type SecureString \
  --region ap-southeast-1
```

**Reading a parameter in Python:**
```python
import boto3

client = boto3.client("ssm", region_name="ap-southeast-1")

response = client.get_parameter(
    Name="/de-practice/dev/db-host",
    WithDecryption=True   # Required for SecureString type
)

db_host = response["Parameter"]["Value"]
```

### 4.3 Referencing AWS Secrets in GitHub Actions

```yaml
steps:
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: ap-southeast-1

  - name: Get secret from Secrets Manager
    run: |
      SECRET=$(aws secretsmanager get-secret-value \
        --secret-id de-practice/dev/db-password \
        --query SecretString \
        --output text)
      echo "::add-mask::$SECRET"
      echo "DB_PASSWORD=$SECRET" >> $GITHUB_ENV

  - name: Use secret
    run: python deploy.py
    env:
      DB_PASSWORD: ${{ env.DB_PASSWORD }}
```

---

## 5. Managing Dev; Staging; and Production Configs

On real client projects, you will have multiple environments; each with different resource names, connection strings, and credentials. The key principle is: **the same code runs in all environments; only the configuration changes.**

### 5.1 Environment-Specific Secret Naming

Use a consistent naming convention that includes the environment:

**Azure Key Vault** (one vault per environment is common):
```
kv-myproject-dev
kv-myproject-staging
kv-myproject-prod
```

Or one vault with prefixed secret names:
```
dev-db-password
staging-db-password
prod-db-password
```

**AWS Secrets Manager / Parameter Store** (path-based naming):
```
/myproject/dev/db-password
/myproject/staging/db-password
/myproject/prod/db-password
```

### 5.2 Injecting the Right Config Per Environment

In your pipeline, pass the environment name as a variable and use it to retrieve the correct secrets:

**Azure Pipelines:**
```yaml
variables:
  - name: environment
    value: dev

steps:
  - script: |
      SECRET=$(az keyvault secret show \
        --vault-name kv-myproject-$(environment) \
        --name db-password \
        --query value -o tsv)
      echo "##vso[task.setvariable variable=DB_PASSWORD;issecret=true]$SECRET"
    displayName: Get environment secret
```

**GitHub Actions:**
```yaml
jobs:
  deploy-dev:
    environment: dev
    env:
      ENVIRONMENT: dev
    steps:
      - run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id /myproject/${{ env.ENVIRONMENT }}/db-password \
            --query SecretString --output text)
          echo "::add-mask::$SECRET"
          echo "DB_PASSWORD=$SECRET" >> $GITHUB_ENV
```

### 5.3 Never Promote Secrets Between Environments

Each environment must have its own separate credentials. Never copy a production secret to a staging environment or vice versa. If a staging environment is compromised, production remains safe.

---

## 6. Secret Rotation Basics

Secret rotation means periodically replacing a secret with a new value to limit the damage if it is ever exposed.

### Why it Matters on Client Projects

Many enterprise clients have security policies requiring credential rotation every 30, 60, or 90 days. As a consultant, you need to understand how rotation works so you can build systems that handle it without downtime.

### How Rotation Works

1. A new secret value is generated (by AWS, Azure, or a custom rotation function)
2. The application or pipeline retrieves the secret fresh on each run (not cached)
3. The old value is kept available briefly during the transition (versioning)
4. The old value is then retired

The critical implication for your code: **always fetch secrets at runtime, never cache them in memory for long periods or store them in environment variables that persist across many runs.**

### AWS Secrets Manager Auto-Rotation

AWS Secrets Manager can rotate RDS database passwords automatically using a Lambda function. When rotation happens:

- Version `AWSCURRENT` holds the new password
- Version `AWSPREVIOUS` holds the old password (available briefly for in-flight connections)

Your code using `get_secret_value` without specifying a version always gets `AWSCURRENT`; the latest value.

### Azure Key Vault Rotation

Azure Key Vault supports automatic rotation for storage account keys and certificates. For other secrets, you set an expiry date and configure an Event Grid alert to notify you when rotation is needed.

---

## 7. Lab: Refactor Hardcoded Credentials to a Secret Store

This lab takes a Python script with hardcoded credentials and refactors it to pull secrets from Azure Key Vault (Azure path) or AWS Secrets Manager (AWS path) at runtime.

### Prerequisites
- Python installed
- Azure CLI or AWS CLI installed and authenticated (from Module 2 lab)
- Access to a Key Vault or Secrets Manager instance (ask your team lead)

---

### Step 1: The Starting Point (What We Are Fixing)

Create a file called `data_connector.py` with intentionally bad practice:

```python
# data_connector.py - BAD EXAMPLE; do not use this pattern

DB_HOST = "prod-db.company.com"
DB_USER = "admin"
DB_PASSWORD = "SuperSecret123!"   # Hardcoded; dangerous

def connect():
    print(f"Connecting to {DB_HOST} as {DB_USER}")
    print("Connection successful (simulated)")

connect()
```

This is what you are replacing.

---

### Step 2: Store the Secret

**Azure Key Vault:**
```powershell
az keyvault secret set `
  --vault-name <your-key-vault-name> `
  --name db-password `
  --value "SuperSecret123!"
```

**AWS Secrets Manager:**
```powershell
aws secretsmanager create-secret `
  --name de-practice/dev/db-password `
  --secret-string "SuperSecret123!"
```

---

### Step 3: Create a .env File for Local Config

Create `.env` (add to `.gitignore`):

```bash
# Azure
KEY_VAULT_URL=https://your-key-vault-name.vault.azure.net/
DB_HOST=prod-db.company.com
DB_USER=admin

# AWS (use one or the other; not both)
AWS_REGION=ap-southeast-1
DB_HOST=prod-db.company.com
DB_USER=admin
```

---

### Step 4: Install Dependencies

**PowerShell / WSL:**
```bash
pip install python-dotenv azure-identity azure-keyvault-secrets boto3
```

---

### Step 5: Refactor the Script

**Azure version:**
```python
# data_connector.py - Azure Key Vault version
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
from dotenv import load_dotenv
import os

load_dotenv()

vault_url = os.environ["KEY_VAULT_URL"]
DB_HOST = os.environ["DB_HOST"]
DB_USER = os.environ["DB_USER"]

credential = DefaultAzureCredential()
client = SecretClient(vault_url=vault_url, credential=credential)

DB_PASSWORD = client.get_secret("db-password").value

def connect():
    print(f"Connecting to {DB_HOST} as {DB_USER}")
    print("Password retrieved from Key Vault (not printed)")
    print("Connection successful (simulated)")

connect()
```

**AWS version:**
```python
# data_connector.py - AWS Secrets Manager version
import boto3
from dotenv import load_dotenv
import os

load_dotenv()

AWS_REGION = os.environ["AWS_REGION"]
DB_HOST = os.environ["DB_HOST"]
DB_USER = os.environ["DB_USER"]

client = boto3.client("secretsmanager", region_name=AWS_REGION)
response = client.get_secret_value(SecretId="de-practice/dev/db-password")
DB_PASSWORD = response["SecretString"]

def connect():
    print(f"Connecting to {DB_HOST} as {DB_USER}")
    print("Password retrieved from Secrets Manager (not printed)")
    print("Connection successful (simulated)")

connect()
```

---

### Step 6: Run Locally

```powershell
python data_connector.py
```

Verify that the script runs, prints the connection message, and does not expose the password value.

---

### Step 7: Add to Your Pipeline

Add a pipeline step that runs this script using the secret from the store (not from `.env`). In your pipeline YAML:

**Azure Pipelines:**
```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: my-service-connection
    scriptType: bash
    inlineScript: |
      export KEY_VAULT_URL="https://your-key-vault.vault.azure.net/"
      export DB_HOST="prod-db.company.com"
      export DB_USER="admin"
      python data_connector.py
  displayName: Run connector with Key Vault secrets
```

**GitHub Actions:**
```yaml
- name: Run connector
  run: python data_connector.py
  env:
    KEY_VAULT_URL: https://your-key-vault.vault.azure.net/
    DB_HOST: prod-db.company.com
    DB_USER: admin
```

Push and confirm the pipeline runs successfully without any secret values appearing in the logs.

---

## 8. Quick Reference

| Concept | Azure | AWS |
|---|---|---|
| Secret store | Azure Key Vault | Secrets Manager (rotating) / Parameter Store (config) |
| CLI create secret | `az keyvault secret set` | `aws secretsmanager create-secret` |
| CLI get secret | `az keyvault secret show` | `aws secretsmanager get-secret-value` |
| Python SDK | `azure-keyvault-secrets` + `azure-identity` | `boto3` |
| Auth (local) | `az login` via `DefaultAzureCredential` | `aws configure` |
| Auth (pipeline) | Service connection | IAM credentials as GitHub/pipeline secrets |
| Naming convention | `env-secret-name` or separate vault per env | `/project/env/secret-name` |
| Never do | Hardcode secrets in code or YAML | Same |
| Always do | Add `.env` to `.gitignore` | Same |

---

*Next module: Module 6: Monitoring: Azure Monitor & CloudWatch*
