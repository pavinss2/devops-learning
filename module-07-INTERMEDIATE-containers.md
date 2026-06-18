# Module 7: Containers for Data Engineers

> **Track:** Intermediate Extension
> **Duration:** 3–4 days
> **Level:** Intermediate
> **Prerequisites:** Module 5: Environment & Secret Management

---

## A Note on Podman vs Docker

This module uses **Podman** as the primary container tool. Podman is a daemonless, rootless container engine that is fully compatible with Docker's image format and the OCI (Open Container Initiative) standard. It requires no licensing and runs without a background service, making it the standard tool for this training track.

If you are onboarded to a client that uses Docker, the transition is straightforward. Docker and Podman share the same image format and almost identical command-line syntax. Wherever the two differ, this module shows both so you are comfortable in either environment.

> **Key fact:** `Dockerfile` syntax is an open standard. The same `Dockerfile` builds correctly with both `podman build` and `docker build`. You never need to rewrite a Dockerfile when switching between the two tools.

---

## Table of Contents

1. [What Containers Solve](#1-what-containers-solve)
2. [Core Concepts](#2-core-concepts)
3. [Installing Podman on Windows](#3-installing-podman-on-windows)
4. [Writing a Dockerfile](#4-writing-a-dockerfile)
5. [Essential Podman Commands](#5-essential-podman-commands)
6. [Podman Compose for Local Development](#6-podman-compose-for-local-development)
7. [Container Registries: ACR and ECR](#7-container-registries-acr-and-ecr)
8. [Using Containers in CI/CD Pipelines](#8-using-containers-in-cicd-pipelines)
9. [Lab: Containerize a Python Data Script](#9-lab-containerize-a-python-data-script)
10. [Quick Reference](#10-quick-reference)

---

## 1. What Containers Solve

The most common frustration on software teams is: "it works on my machine." A script runs perfectly locally but fails in the pipeline, on a colleague's machine, or in the client's environment because of differences in:

- Python version (3.9 vs 3.11)
- Installed libraries and their versions
- Operating system differences
- Environment variables and config files
- File paths

Containers solve this by packaging your code together with everything it needs to run; the Python version, all libraries, config files, and system dependencies; into a single self-contained unit. That unit runs identically everywhere: on your laptop, in the CI/CD pipeline, and in the client's production environment.

For data engineers, this matters because:
- Pipeline environments on clients are often different from your local setup
- Containers are the standard unit of deployment for most enterprise data platforms
- Airflow, Spark, and other data tools are frequently distributed and run as containers

---

## 2. Core Concepts

### Image
An image is a read-only template that defines what a container will contain: the OS layer, the runtime (Python), the libraries, the application code. You build an image from a `Dockerfile`. Images are stored in registries and shared between machines.

Think of an image as a recipe.

### Container
A container is a running instance of an image. You can run many containers from the same image simultaneously. Containers are isolated from each other and from the host machine. When a container stops, any changes made inside it are lost unless you use volumes.

Think of a container as a dish made from the recipe.

### Dockerfile
A `Dockerfile` is a text file with instructions for building an image. Each instruction adds a layer to the image. The `Dockerfile` format is an open standard; the same file works with both Podman and Docker.

### Registry
A registry is a storage service for images. You push images to a registry and pull them from it. Common registries:

| Registry | Provider | When used |
|---|---|---|
| Azure Container Registry (ACR) | Azure | Azure-based client projects |
| Amazon Elastic Container Registry (ECR) | AWS | AWS-based client projects |
| Docker Hub | Docker | Public images; open-source base images |
| Quay.io | Red Hat | Common in Podman-native environments |

### Volume
A volume persists data outside a container. Without volumes, data written inside a container disappears when it stops. Volumes are commonly used for local development to mount your code folder into the container so changes reflect immediately without rebuilding.

---

## 3. Installing Podman on Windows

### Option A: Podman Desktop (Recommended)

Podman Desktop is a GUI application that includes the Podman CLI and manages the underlying Linux VM automatically. It is the easiest way to get started on Windows.

1. Download Podman Desktop from [https://podman-desktop.io](https://podman-desktop.io)
2. Run the installer and follow the prompts
3. Podman Desktop will install the Podman CLI and set up a Podman machine (a lightweight Linux VM) automatically
4. Open Podman Desktop and wait for the machine to start (the status indicator turns green when ready)

Verify in PowerShell:

```powershell
podman --version
podman run hello-world
```

The `hello-world` command pulls a test image, runs it, and prints a confirmation message. If you see it, Podman is working correctly.

### Option B: WSL (Ubuntu)

```bash
sudo apt update && sudo apt install -y podman
podman --version
```

### Podman Machine Commands (PowerShell)

The Podman machine is the Linux VM Podman uses on Windows. You rarely need to interact with it directly, but these commands are useful:

```powershell
podman machine start    # Start the VM if it is not running
podman machine stop     # Stop the VM
podman machine list     # Check VM status
```

---

## 4. Writing a Dockerfile

A Dockerfile is a series of instructions that Podman (or Docker) executes top-to-bottom to build an image. The syntax is identical for both tools.

### 4.1 Basic Dockerfile Structure

```dockerfile
# Start from an official base image
FROM python:3.11-slim

# Set the working directory inside the container
WORKDIR /app

# Copy dependency file first (for layer caching)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY src/ ./src/

# Set environment variables
ENV PYTHONUNBUFFERED=1

# Define the command to run when the container starts
CMD ["python", "src/main.py"]
```

### 4.2 Key Instructions

| Instruction | What it does |
|---|---|
| `FROM` | Sets the base image; always the first instruction |
| `WORKDIR` | Sets the working directory for subsequent instructions |
| `COPY` | Copies files from your machine into the image |
| `RUN` | Executes a command during the build (installs packages, etc.) |
| `ENV` | Sets an environment variable inside the container |
| `ARG` | Build-time variable (not available at runtime) |
| `EXPOSE` | Documents which port the container listens on |
| `CMD` | The default command to run when the container starts |
| `ENTRYPOINT` | Like CMD but harder to override; used for single-purpose containers |

### 4.3 Choosing a Base Image

| Base image | Size | When to use |
|---|---|---|
| `python:3.11` | ~900 MB | Full Debian; most compatible; use when unsure |
| `python:3.11-slim` | ~130 MB | Smaller; missing some system tools; good default |
| `python:3.11-alpine` | ~50 MB | Smallest; may have compilation issues with some libraries |

Always pin to a specific version (`python:3.11-slim`) rather than `python:latest` to ensure reproducibility.

### 4.4 Layer Caching

Both Podman and Docker cache each layer of the image. If a layer has not changed, the cached version is reused on the next build. This makes subsequent builds much faster.

Best practice: copy `requirements.txt` and run `pip install` before copying your application code. This way, if only your code changes, the pip install layer is reused:

```dockerfile
# GOOD: dependencies cached separately from code
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src/ ./src/

# BAD: any code change invalidates the pip install cache
COPY . .
RUN pip install -r requirements.txt
```

### 4.5 .containerignore / .dockerignore

Podman reads `.containerignore` to exclude files from the build context. It also falls back to `.dockerignore` if `.containerignore` is not present; so either name works. On client projects using Docker you will see `.dockerignore`; for new projects use `.containerignore`.

```
.git/
.venv/
venv/
__pycache__/
*.pyc
.env
*.log
tests/
.pytest_cache/
```

---

## 5. Essential Podman Commands

All commands below use `podman`. If you are on a client project using Docker, replace `podman` with `docker`; the commands are identical unless noted otherwise.

### Building an Image

```powershell
# Build an image from the Dockerfile in the current directory
podman build -t my-pipeline:latest .

# Build with a specific tag
podman build -t my-pipeline:1.0.0 .

# Docker equivalent (identical syntax)
# docker build -t my-pipeline:latest .
```

### Running a Container

```powershell
# Run a container; remove it automatically when done
podman run --rm my-pipeline:latest

# Run with environment variables
podman run --rm -e DB_HOST=localhost -e DB_PORT=5432 my-pipeline:latest

# Run with a .env file
podman run --rm --env-file .env my-pipeline:latest

# Run interactively (opens a shell inside the container)
podman run --rm -it my-pipeline:latest bash

# Run with a volume mount (maps local folder into container)
podman run --rm -v ${PWD}/data:/app/data my-pipeline:latest

# Docker equivalents (identical syntax)
# docker run --rm my-pipeline:latest
# docker run --rm -e DB_HOST=localhost my-pipeline:latest
```

### Viewing and Managing Containers

```powershell
# List running containers
podman ps

# List all containers including stopped
podman ps -a

# View logs from a container
podman logs <container-id>

# Follow logs in real time
podman logs -f <container-id>

# Execute a command in a running container
podman exec -it <container-id> bash

# Stop a running container
podman stop <container-id>

# Remove a stopped container
podman rm <container-id>

# Docker equivalents (identical syntax)
# docker ps / docker logs / docker exec / docker stop / docker rm
```

### Viewing and Managing Images

```powershell
# List local images
podman images

# Remove an image
podman rmi my-pipeline:latest

# Remove all unused images
podman image prune

# Docker equivalents (identical syntax)
# docker images / docker rmi / docker image prune
```

### Pushing and Pulling Images

```powershell
# Log in to a registry
podman login <registry-url>

# Tag an image for a specific registry
podman tag my-pipeline:latest myregistry.azurecr.io/my-pipeline:latest

# Push to registry
podman push myregistry.azurecr.io/my-pipeline:latest

# Pull from registry
podman pull myregistry.azurecr.io/my-pipeline:latest

# Docker equivalents (identical syntax)
# docker login / docker tag / docker push / docker pull
```

> **Where Podman and Docker differ:** Podman is rootless by default; it does not require a running daemon (`dockerd`). This means `podman` commands do not need `sudo` and there is no background service to manage. On some client systems still using Docker, you may need to use `sudo docker` or be added to the `docker` group. This is a system administration difference; the commands themselves are the same.

---

## 6. Podman Compose for Local Development

Podman Compose is the Podman equivalent of Docker Compose. It reads the same `docker-compose.yml` / `compose.yml` file format, so files are fully portable between the two tools.

### Install Podman Compose

**PowerShell / WSL:**
```bash
pip install podman-compose
```

### Example: Python App + PostgreSQL

```yaml
# compose.yml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: dev_user
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: dev_database
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  app:
    build: .
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: dev_user
      DB_PASSWORD: dev_password
      DB_NAME: dev_database
    volumes:
      - ./src:/app/src

volumes:
  postgres-data:
```

### Podman Compose Commands

```powershell
# Start all services
podman-compose up

# Start in background
podman-compose up -d

# View logs
podman-compose logs

# Follow logs for a specific service
podman-compose logs -f app

# Stop all services
podman-compose down

# Stop and remove volumes
podman-compose down -v

# Rebuild images
podman-compose build

# Docker Compose equivalents (replace podman-compose with docker compose)
# docker compose up -d
# docker compose logs -f app
# docker compose down
```

> **Note for client projects:** If a client uses Docker Compose, the `compose.yml` file you wrote with Podman Compose works without modification. Simply run `docker compose up` instead of `podman-compose up`.

---

## 7. Container Registries: ACR and ECR

After building an image, you push it to a registry so it can be pulled by CI/CD pipelines, Kubernetes clusters, or other machines.

### 7.1 Azure Container Registry (ACR)

**Log in with Podman:**
```powershell
podman login <registry-name>.azurecr.io `
  --username <username> `
  --password <password-or-token>
```

Or using the Azure CLI to get credentials automatically:
```powershell
az acr login --name <registry-name> --expose-token
# Copy the token output and use it as the password above
```

> **Docker equivalent:** `docker login` uses the same syntax and accepts the same credentials.

**Tag and push:**
```powershell
podman tag my-pipeline:latest <registry-name>.azurecr.io/my-pipeline:latest
podman push <registry-name>.azurecr.io/my-pipeline:latest

# Docker equivalent
# docker tag my-pipeline:latest <registry-name>.azurecr.io/my-pipeline:latest
# docker push <registry-name>.azurecr.io/my-pipeline:latest
```

**Pull:**
```powershell
podman pull <registry-name>.azurecr.io/my-pipeline:latest
# docker pull <registry-name>.azurecr.io/my-pipeline:latest
```

### 7.2 Amazon Elastic Container Registry (ECR)

**Log in with Podman:**
```powershell
aws ecr get-login-password --region ap-southeast-1 | `
  podman login --username AWS --password-stdin `
  <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

# Docker equivalent
# aws ecr get-login-password --region ap-southeast-1 | `
#   docker login --username AWS --password-stdin `
#   <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com
```

**Create a repository (first time only):**
```powershell
aws ecr create-repository `
  --repository-name my-pipeline `
  --region ap-southeast-1
```

**Tag and push:**
```powershell
podman tag my-pipeline:latest `
  <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/my-pipeline:latest

podman push `
  <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/my-pipeline:latest

# Docker equivalents: replace podman with docker; syntax is identical
```

---

## 8. Using Containers in CI/CD Pipelines

CI/CD pipelines typically use Docker rather than Podman because most hosted runners (GitHub Actions, Azure Pipelines) have Docker pre-installed. The commands in pipeline YAML files use `docker`; this is expected and fine. Your local development uses Podman; the pipeline uses Docker. The images are compatible because they share the same OCI format.

### Azure Pipelines

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: my-service-connection
              scriptType: bash
              inlineScript: |
                az acr login --name myregistry
                docker build -t myregistry.azurecr.io/my-pipeline:$(Build.BuildId) .
                docker push myregistry.azurecr.io/my-pipeline:$(Build.BuildId)
            displayName: Build and push image
```

### GitHub Actions

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Log in to ECR
        run: |
          aws ecr get-login-password --region ap-southeast-1 | \
            docker login --username AWS --password-stdin \
            ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com

      - name: Build and push
        run: |
          IMAGE=${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com/my-pipeline
          docker build -t $IMAGE:${{ github.sha }} .
          docker push $IMAGE:${{ github.sha }}
```

> **Why `docker` in pipelines?** GitHub-hosted and Microsoft-hosted runners have Docker pre-installed. Switching runners to use Podman would require self-hosted runner setup, which is out of scope for most client projects. Using `docker` in pipelines and `podman` locally is a standard and accepted pattern.

---

## 9. Lab: Containerize a Python Data Script

This lab containerizes a Python script using Podman, runs it locally, and pushes the image to ACR or ECR.

### Prerequisites
- Podman Desktop installed and the Podman machine running (Section 3)
- Azure CLI or AWS CLI authenticated (Module 2 lab)
- Access to ACR or ECR (ask your team lead)

---

### Step 1: Create the Project Files

Create a new folder called `container-practice`. Inside it, create the following files:

`src/main.py`:
```python
import os

def process_data():
    source = os.environ.get("DATA_SOURCE", "local")
    destination = os.environ.get("DATA_DESTINATION", "output")

    print("Starting data processing job")
    print(f"Source: {source}")
    print(f"Destination: {destination}")

    records = [
        {"id": 1, "name": "Alice", "score": 95},
        {"id": 2, "name": "Bob", "score": 87},
        {"id": 3, "name": "Charlie", "score": 92},
    ]

    print(f"Processed {len(records)} records")
    print("Job complete")

if __name__ == "__main__":
    process_data()
```

`requirements.txt`:
```
# No external libraries needed for this example
```

---

### Step 2: Write the Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

ENV PYTHONUNBUFFERED=1

CMD ["python", "src/main.py"]
```

---

### Step 3: Write the .containerignore

```
.git/
__pycache__/
*.pyc
.env
*.log
```

---

### Step 4: Build the Image

```powershell
cd container-practice
podman build -t data-processor:latest .
```

Watch the build output. You should see each Dockerfile instruction execute in order. The first build takes longer; subsequent builds use cached layers and are much faster.

List your local images to confirm it was built:

```powershell
podman images
```

> **Docker equivalent:** `docker build -t data-processor:latest .` and `docker images`

---

### Step 5: Run the Container

```powershell
# Basic run
podman run --rm data-processor:latest

# Run with environment variables
podman run --rm `
  -e DATA_SOURCE=azure_blob `
  -e DATA_DESTINATION=staging_db `
  data-processor:latest
```

Verify the output shows the environment variable values you passed in.

> **Docker equivalent:** Replace `podman run` with `docker run`; syntax is identical.

---

### Step 6: Run Interactively

Open a shell inside the container to explore its contents:

```powershell
podman run --rm -it data-processor:latest bash
```

Inside the container:
```bash
ls /app
cat /app/src/main.py
python --version
exit
```

> **Docker equivalent:** `docker run --rm -it data-processor:latest bash`

---

### Step 7: Push to ACR or ECR

**Azure (ACR):**
```powershell
az acr login --name <your-registry-name> --expose-token
# Use the token as the password when prompted

podman tag data-processor:latest <your-registry-name>.azurecr.io/data-processor:latest
podman push <your-registry-name>.azurecr.io/data-processor:latest
```

**AWS (ECR):**
```powershell
aws ecr get-login-password --region ap-southeast-1 | `
  podman login --username AWS --password-stdin `
  <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

aws ecr create-repository --repository-name data-processor --region ap-southeast-1

podman tag data-processor:latest `
  <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/data-processor:latest

podman push `
  <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/data-processor:latest
```

Verify in the Azure Portal or AWS Console that the image appears in the registry.

---

### Step 8: Pull and Run from Registry

Delete your local image and pull it fresh from the registry to confirm the push worked:

```powershell
podman rmi data-processor:latest
podman pull <your-registry-url>/data-processor:latest
podman run --rm <your-registry-url>/data-processor:latest
```

> **Docker equivalents:** Replace `podman` with `docker`; syntax is identical.

---

## 10. Quick Reference

### Command Comparison: Podman vs Docker

| Action | Podman | Docker |
|---|---|---|
| Build image | `podman build -t name:tag .` | `docker build -t name:tag .` |
| Run container | `podman run --rm name:tag` | `docker run --rm name:tag` |
| Run with env var | `podman run --rm -e VAR=val name:tag` | `docker run --rm -e VAR=val name:tag` |
| Run with .env file | `podman run --rm --env-file .env name:tag` | `docker run --rm --env-file .env name:tag` |
| Interactive shell | `podman run --rm -it name:tag bash` | `docker run --rm -it name:tag bash` |
| List running | `podman ps` | `docker ps` |
| List all | `podman ps -a` | `docker ps -a` |
| View logs | `podman logs <id>` | `docker logs <id>` |
| Follow logs | `podman logs -f <id>` | `docker logs -f <id>` |
| Exec into container | `podman exec -it <id> bash` | `docker exec -it <id> bash` |
| Stop container | `podman stop <id>` | `docker stop <id>` |
| List images | `podman images` | `docker images` |
| Remove image | `podman rmi name:tag` | `docker rmi name:tag` |
| Prune images | `podman image prune` | `docker image prune` |
| Tag image | `podman tag src dst` | `docker tag src dst` |
| Push image | `podman push name:tag` | `docker push name:tag` |
| Pull image | `podman pull name:tag` | `docker pull name:tag` |
| Login to registry | `podman login <url>` | `docker login <url>` |
| Compose up | `podman-compose up -d` | `docker compose up -d` |
| Compose down | `podman-compose down` | `docker compose down` |
| Compose logs | `podman-compose logs -f` | `docker compose logs -f` |

### Key Differences to Remember

| Aspect | Podman | Docker |
|---|---|---|
| Daemon required | No; daemonless | Yes; `dockerd` must be running |
| Root required | No; rootless by default | Requires `docker` group or `sudo` |
| License | Free; open source | Free tier available; Desktop requires paid license for commercial use |
| Dockerfile format | OCI standard; same as Docker | OCI standard |
| Image compatibility | Fully compatible with Docker images | Fully compatible with Podman images |
| CI/CD pipelines | Use `docker` on hosted runners | Native |

---

*Next module: Module 8: Kubernetes for Data Engineers*
