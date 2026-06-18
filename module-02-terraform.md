# Module 2: Infrastructure as Code with Terraform

> **Track:** Core Track
> **Duration:** 4–5 days
> **Level:** Beginner
> **Prerequisites:** Module 1: Version Control with Git

---

## Table of Contents

1. [Why Infrastructure as Code](#1-why-infrastructure-as-code)
2. [Core Concepts](#2-core-concepts)
3. [Installing Terraform on Windows](#3-installing-terraform-on-windows)
4. [HCL Syntax Basics](#4-hcl-syntax-basics)
5. [The Terraform Workflow](#5-the-terraform-workflow)
6. [State Management](#6-state-management)
7. [Terraform Modules](#7-terraform-modules)
8. [Provisioning Data Resources](#8-provisioning-data-resources)
9. [Terraform Cloud](#9-terraform-cloud)
10. [Integrating Terraform into CI/CD](#10-integrating-terraform-into-cicd)
11. [Lab: Provision Cloud Storage with Terraform Cloud](#11-lab-provision-cloud-storage-with-terraform-cloud)
12. [Quick Reference](#12-quick-reference)

---

## 1. Why Infrastructure as Code

In a traditional setup, cloud resources like storage accounts, databases, and virtual machines are created by clicking through the cloud portal. This works for one-off experiments but creates serious problems on client projects:

- **No repeatability:** Another engineer cannot recreate the same environment without following the same manual steps
- **No history:** There is no record of what was created, when, or why
- **Environment drift:** Dev, staging, and production end up with slightly different configurations over time because changes were made manually in one but not the others
- **No review process:** Changes go straight to production with no approval or audit trail

Infrastructure as Code (IaC) solves this by treating cloud infrastructure the same way you treat application code; written in files, committed to Git, reviewed via PRs, and applied automatically through a pipeline.

Terraform is the industry standard IaC tool. It works across Azure, AWS, GCP, and hundreds of other providers using the same language and workflow. This is why it is in the Core Track; clients expect consultants to provision resources reproducibly, not through the portal.

---

## 2. Core Concepts

### Declarative vs. Imperative

Terraform is **declarative**. You describe the desired end state and Terraform figures out how to get there. You do not write steps like "first create this, then attach that"; you write "I want a storage account with these properties" and Terraform handles the rest.

This is different from **imperative** tools like shell scripts or Azure CLI where you write the exact sequence of commands to run.

### Idempotency

Running `terraform apply` multiple times with the same configuration always produces the same result. If the resource already exists and nothing has changed, Terraform does nothing. If something has drifted from the desired state, Terraform corrects it. This makes Terraform safe to run repeatedly.

### Provider

A provider is a plugin that tells Terraform how to interact with a specific platform. For example:

```hcl
provider "azurerm" {
  features {}
}

provider "aws" {
  region = "ap-southeast-1"
}
```

Providers are downloaded automatically when you run `terraform init`.

### Resource

A resource is a single piece of infrastructure you want Terraform to manage; a storage account, a virtual machine, a database, etc.

### State

Terraform keeps a record of everything it has created in a **state file** (`terraform.tfstate`). This is how it knows what already exists and what needs to change. State management is covered in detail in Section 6.

---

## 3. Installing Terraform on Windows

### Option A: PowerShell (using winget)

```powershell
winget install HashiCorp.Terraform
```

Close and reopen PowerShell after installation, then verify:

```powershell
terraform --version
# Expected: Terraform v1.x.x
```

### Option B: WSL (Ubuntu)

```bash
sudo apt update && sudo apt install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform

terraform --version
```

### Option C: Manual Install (PowerShell)

1. Download the Windows zip from [https://developer.hashicorp.com/terraform/downloads](https://developer.hashicorp.com/terraform/downloads)
2. Extract the `terraform.exe` file
3. Move it to a folder like `C:\Tools\terraform\`
4. Add that folder to your system PATH:
   - Search "Environment Variables" in the Start menu
   - Edit the `Path` variable under System variables
   - Add `C:\Tools\terraform\`
5. Reopen PowerShell and run `terraform --version`

---

## 4. HCL Syntax Basics

Terraform uses HashiCorp Configuration Language (HCL). It is designed to be readable and straightforward. Here is a breakdown of the key building blocks.

### 4.1 Resources

A resource block defines a piece of infrastructure:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-de-practice"
  location = "Southeast Asia"
}
```

The structure is:
```
resource "<provider>_<type>" "<local_name>" {
  argument = value
}
```

- `azurerm_resource_group` is the resource type (from the Azure provider)
- `main` is a local name you choose; used to reference this resource elsewhere
- Inside the block are the arguments specific to that resource type

---

### 4.2 Variables

Variables make your configuration reusable and avoid hardcoding values:

```hcl
variable "location" {
  type        = string
  description = "Azure region for all resources"
  default     = "Southeast Asia"
}

variable "project_name" {
  type        = string
  description = "Name prefix for all resources"
}
```

Reference a variable with `var.`:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-${var.project_name}"
  location = var.location
}
```

Pass variable values in a `terraform.tfvars` file (never commit this if it contains secrets):

```hcl
location     = "Southeast Asia"
project_name = "de-practice"
```

---

### 4.3 Outputs

Outputs expose values from your configuration; useful for sharing resource details with other configurations or for inspecting after apply:

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.main.name
}

output "storage_account_name" {
  value = azurerm_storage_account.main.name
}
```

After `terraform apply`, outputs are printed to the terminal. You can also retrieve them with `terraform output`.

---

### 4.4 Locals

Locals are computed values you define once and reuse:

```hcl
locals {
  common_tags = {
    project     = var.project_name
    environment = "dev"
    managed_by  = "terraform"
  }
}

resource "azurerm_resource_group" "main" {
  name     = "rg-${var.project_name}"
  location = var.location
  tags     = local.common_tags
}
```

---

### 4.5 File Structure

A typical Terraform project is organized like this:

```
my-terraform-project/
├── main.tf           # Main resource definitions
├── variables.tf      # Variable declarations
├── outputs.tf        # Output definitions
├── providers.tf      # Provider configuration
├── terraform.tfvars  # Variable values (gitignored if secrets)
└── .gitignore
```

This is not enforced by Terraform; you can put everything in one file. But this structure is the widely adopted convention and what you will see on client projects.

---

## 5. The Terraform Workflow

Every Terraform project follows the same four-command workflow:

### Step 1: `terraform init`

Downloads the required providers and sets up the backend. Run this once when starting a new project or after changing providers.

```powershell
terraform init
```

You will see Terraform downloading the Azure or AWS provider plugin.

---

### Step 2: `terraform plan`

Shows exactly what Terraform will do before making any changes. Think of it as a dry run. Nothing is created or modified.

```powershell
terraform plan
```

The output uses symbols to show what will happen:

| Symbol | Meaning |
|---|---|
| `+` | Resource will be created |
| `-` | Resource will be destroyed |
| `~` | Resource will be updated in place |
| `-/+` | Resource will be destroyed and recreated |

Always review the plan output carefully before applying, especially on production environments.

---

### Step 3: `terraform apply`

Applies the changes shown in the plan. Terraform will show the plan again and ask for confirmation:

```powershell
terraform apply
```

Type `yes` when prompted. To skip the prompt (useful in CI/CD pipelines):

```powershell
terraform apply -auto-approve
```

---

### Step 4: `terraform destroy`

Destroys all resources managed by this configuration. Use with care; this is irreversible on production.

```powershell
terraform destroy
```

Useful for cleaning up practice environments after a lab to avoid incurring cloud costs.

---

## 6. State Management

### 6.1 What is State

Terraform stores a record of everything it has created in a file called `terraform.tfstate`. This JSON file maps your HCL configuration to the real resources in the cloud. Without it, Terraform has no way of knowing what already exists.

### 6.2 Why Not Local State

By default, state is stored locally on your machine. This causes problems on teams:

- If you run `terraform apply` and a colleague also runs it from their machine, they have different state files and will conflict
- If your machine crashes or the state file is deleted, Terraform loses track of what it created
- Local state files often contain sensitive values (like connection strings) and should never be committed to Git

### 6.3 Remote Backends

A remote backend stores state in a shared, central location. The two most common options for your clients are:

**Azure Blob Storage:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "de-practice.terraform.tfstate"
  }
}
```

**AWS S3:**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "de-practice/terraform.tfstate"
    region = "ap-southeast-1"
  }
}
```

**Terraform Cloud (covered in Section 9):** The simplest option; no need to create a storage account or S3 bucket yourself.

### 6.4 State Locking

When using a remote backend, Terraform automatically locks the state file while an operation is running. This prevents two people from running `terraform apply` at the same time and corrupting the state. Azure Blob Storage and S3 both support state locking.

---

## 7. Terraform Modules

A module is a reusable package of Terraform configuration. Instead of copying the same resource blocks across multiple projects, you write them once as a module and call it wherever needed.

### 7.1 Using a Module

```hcl
module "storage" {
  source       = "./modules/storage-account"
  project_name = var.project_name
  location     = var.location
  environment  = "dev"
}
```

### 7.2 Writing a Simple Module

Create a folder `modules/storage-account/` with its own `main.tf`, `variables.tf`, and `outputs.tf`:

```hcl
# modules/storage-account/variables.tf
variable "project_name" { type = string }
variable "location"     { type = string }
variable "environment"  { type = string }

# modules/storage-account/main.tf
resource "azurerm_storage_account" "this" {
  name                     = "st${var.project_name}${var.environment}"
  resource_group_name      = "rg-${var.project_name}"
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# modules/storage-account/outputs.tf
output "storage_account_name" {
  value = azurerm_storage_account.this.name
}
```

### 7.3 Public Module Registry

Terraform has a public registry at [registry.terraform.io](https://registry.terraform.io) with pre-built modules for common patterns. For example, the official Azure modules from Microsoft are available there. On client projects, you may be asked to use these rather than writing everything from scratch.

---

## 8. Provisioning Data Resources

Here are the most common data infrastructure resources you will provision on client projects, with examples for both Azure and AWS.

### 8.1 Azure Examples

**Resource Group:**
```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-${var.project_name}-${var.environment}"
  location = var.location
  tags     = local.common_tags
}
```

**Storage Account (Data Lake):**
```hcl
resource "azurerm_storage_account" "datalake" {
  name                     = "st${var.project_name}${var.environment}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  is_hns_enabled           = true   # Enables hierarchical namespace for ADLS Gen2
  tags                     = local.common_tags
}
```

**Key Vault:**
```hcl
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.project_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
  tags                = local.common_tags
}
```

---

### 8.2 AWS Examples

**S3 Bucket (Data Lake):**
```hcl
resource "aws_s3_bucket" "datalake" {
  bucket = "${var.project_name}-${var.environment}-datalake"
  tags   = local.common_tags
}

resource "aws_s3_bucket_versioning" "datalake" {
  bucket = aws_s3_bucket.datalake.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

**Glue Database:**
```hcl
resource "aws_glue_catalog_database" "main" {
  name        = "${var.project_name}_${var.environment}"
  description = "Glue catalog database for ${var.project_name}"
}
```

**Secrets Manager Secret:**
```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name        = "${var.project_name}/${var.environment}/db-password"
  description = "Database password for ${var.project_name}"
  tags        = local.common_tags
}
```

---

## 9. Terraform Cloud

Terraform Cloud is HashiCorp's managed platform for running Terraform. Instead of running `terraform apply` from your local machine, Terraform Cloud runs it remotely in a consistent, controlled environment. It also handles remote state automatically; no need to set up a storage account or S3 bucket for state.

### 9.1 Why Terraform Cloud

| Feature | Local Terraform | Terraform Cloud |
|---|---|---|
| State storage | Manual setup required | Automatic; built in |
| State locking | Depends on backend | Automatic |
| Run history | None | Full history of all plans and applies |
| Team access | No control | Role-based access per workspace |
| CI/CD integration | Manual | Built-in VCS integration |
| Secret management | Manual | Encrypted variable store |

### 9.2 Key Concepts in Terraform Cloud

**Organization:** The top-level account in Terraform Cloud; usually one per company.

**Workspace:** Equivalent to a single Terraform configuration with its own state, variables, and run history. You typically have one workspace per environment (e.g., `de-practice-dev`, `de-practice-prod`).

**Run:** A single execution of `terraform plan` or `terraform apply` in Terraform Cloud. Every run is logged and can be reviewed.

---

### 9.3 Setting Up Terraform Cloud

1. Create a free account at [app.terraform.io](https://app.terraform.io)
2. Create an organization (use your company name or a practice name)
3. Create a new workspace; choose **CLI-driven workflow** for now
4. In your terminal, log in:

```powershell
terraform login
```

This opens a browser where you generate an API token. Paste the token when prompted.

---

### 9.4 Connecting Your Configuration to Terraform Cloud

Add a `cloud` block to your `providers.tf` or `main.tf`:

```hcl
terraform {
  cloud {
    organization = "your-org-name"

    workspaces {
      name = "de-practice-dev"
    }
  }
}
```

Then run:

```powershell
terraform init
```

Terraform will migrate to the cloud backend. From this point, `terraform plan` and `terraform apply` are executed remotely in Terraform Cloud while you see the output streamed to your terminal.

---

### 9.5 Managing Variables in Terraform Cloud

Instead of a local `terraform.tfvars` file, store variable values directly in the workspace:

1. Go to your workspace in [app.terraform.io](https://app.terraform.io)
2. Click **Variables**
3. Add your variables under **Terraform Variables** (equivalent to `tfvars`)
4. For secrets (like cloud credentials), add them as **Environment Variables** and mark them as **Sensitive**

Sensitive variables are write-only; once saved, nobody can read them back from the UI.

---

### 9.6 VCS Integration (Optional)

Terraform Cloud can connect directly to Azure Repos or GitHub. When you push to your repo, Terraform Cloud automatically triggers a plan. After review, you apply from the Terraform Cloud UI. This is the pattern used when Terraform Cloud is the CI/CD mechanism rather than Azure Pipelines or GitHub Actions.

---

## 10. Integrating Terraform into CI/CD

This section is a preview. Full pipeline implementation is covered in Modules 5A and 5B. The standard pattern is:

- **On Pull Request:** Run `terraform plan` and post the output as a PR comment so reviewers can see exactly what infrastructure changes the PR will make
- **On merge to main:** Run `terraform apply` to apply the approved changes

This means infrastructure changes go through the same review process as code changes; a PR with a plan, review, approve, merge, apply.

The commands used in a pipeline are the same ones you have already learned:

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

The `-out=tfplan` flag saves the plan to a file so the apply step uses exactly the plan that was reviewed, not a new one generated at apply time.

---

## 11. Lab: Provision Cloud Storage with Terraform Cloud

This lab provisions a storage resource (Azure Storage Account or AWS S3 bucket) using Terraform with Terraform Cloud as the backend.

### Prerequisites
- Terraform installed (Section 3)
- Terraform Cloud account (Section 9.3)
- Azure subscription or AWS account (ask your team lead for access to a practice subscription)
- Azure CLI or AWS CLI installed (for authentication)

---

### Step 1: Install the Cloud CLI

**Azure CLI (PowerShell):**
```powershell
winget install Microsoft.AzureCLI
az login
```

**AWS CLI (PowerShell):**
```powershell
winget install Amazon.AWSCLI
aws configure
# Enter your Access Key ID, Secret Access Key, and region when prompted
```

**WSL equivalents:**
```bash
# Azure
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login

# AWS
sudo apt install awscli
aws configure
```

---

### Step 2: Create Your Project Folder

**PowerShell:**
```powershell
mkdir terraform-practice
cd terraform-practice
```

**WSL:**
```bash
mkdir terraform-practice && cd terraform-practice
```

---

### Step 3: Write the Configuration

Create a file called `providers.tf`:

**For Azure:**
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }

  cloud {
    organization = "your-org-name"
    workspaces {
      name = "de-practice-dev"
    }
  }
}

provider "azurerm" {
  features {}
}
```

**For AWS:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }

  cloud {
    organization = "your-org-name"
    workspaces {
      name = "de-practice-dev"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}
```

---

### Step 4: Write the Resource

Create a file called `main.tf`:

**For Azure:**
```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-de-practice"
  location = "Southeast Asia"
}

resource "azurerm_storage_account" "main" {
  name                     = "stpractice${random_id.suffix.hex}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "random_id" "suffix" {
  byte_length = 4
}
```

**For AWS:**
```hcl
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "main" {
  bucket = "de-practice-${random_id.suffix.hex}"

  tags = {
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

Create a file called `outputs.tf`:

**For Azure:**
```hcl
output "storage_account_name" {
  value = azurerm_storage_account.main.name
}
```

**For AWS:**
```hcl
output "bucket_name" {
  value = aws_s3_bucket.main.bucket
}
```

---

### Step 5: Add a .gitignore

Create a `.gitignore` file:

```gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
```

> **Note on `.terraform.lock.hcl`:** Do NOT add this file to `.gitignore`. HashiCorp's official recommendation is to commit the lock file so that everyone on the team uses the same provider versions. It works the same way as `package-lock.json` in Node.js or `requirements.txt` pinning in Python.

---

### Step 6: Log in to Terraform Cloud and Initialize

```powershell
terraform login
terraform init
```

After init, go to [app.terraform.io](https://app.terraform.io) and confirm your workspace was created.

---

### Step 7: Set Cloud Credentials in Terraform Cloud

Go to your workspace in Terraform Cloud; Variables; add the following as **Environment Variables** marked **Sensitive**:

**Azure:**
- `ARM_CLIENT_ID`
- `ARM_CLIENT_SECRET`
- `ARM_SUBSCRIPTION_ID`
- `ARM_TENANT_ID`

(Ask your team lead for a service principal with Contributor access to a practice subscription.)

**AWS:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

---

### Step 8: Plan and Apply

```powershell
terraform plan
```

Watch the output stream from Terraform Cloud to your terminal. You will see the plan show a `+` for each resource being created.

If the plan looks correct:

```powershell
terraform apply
```

Type `yes` when prompted. Go to your cloud portal and confirm the resource was created.

Also check the run history in Terraform Cloud; you will see a full log of the plan and apply.

---

### Step 9: Inspect Outputs and State

```powershell
terraform output
```

In Terraform Cloud, go to **States** to see the current state file. Notice it is stored remotely; there is no local `terraform.tfstate` file.

---

### Step 10: Clean Up

```powershell
terraform destroy
```

Type `yes` to confirm. Verify in your cloud portal that the resource has been removed. This avoids leaving resources running and incurring costs.

---

## 12. Quick Reference

| Command | What it does |
|---|---|
| `terraform init` | Initialize providers and backend |
| `terraform plan` | Show what will change; no modifications made |
| `terraform apply` | Apply changes |
| `terraform apply -auto-approve` | Apply without confirmation prompt |
| `terraform destroy` | Destroy all managed resources |
| `terraform output` | Show output values |
| `terraform show` | Show current state in readable format |
| `terraform state list` | List all resources in state |
| `terraform login` | Authenticate with Terraform Cloud |
| `terraform fmt` | Auto-format HCL files |
| `terraform validate` | Check configuration for syntax errors |

---

*Next module: Module 3: CI/CD Fundamentals*
