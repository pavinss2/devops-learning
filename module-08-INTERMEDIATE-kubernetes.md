# Module 8: Kubernetes for Data Engineers

> **Track:** Intermediate Extension
> **Duration:** 4–5 days
> **Level:** Intermediate
> **Prerequisites:** Module 7: Containers for Data Engineers

---

## Table of Contents

1. [Why Data Engineers Need Kubernetes](#1-why-data-engineers-need-kubernetes)
2. [Core Concepts](#2-core-concepts)
3. [Setting Up a Local Cluster with Minikube](#3-setting-up-a-local-cluster-with-minikube)
4. [kubectl Essentials](#4-kubectl-essentials)
5. [Writing Kubernetes Manifests](#5-writing-kubernetes-manifests)
6. [Kubernetes on Azure (AKS) and AWS (EKS)](#6-kubernetes-on-azure-aks-and-aws-eks)
7. [Running Data Workloads on Kubernetes](#7-running-data-workloads-on-kubernetes)
8. [Helm Basics](#8-helm-basics)
9. [Lab: Deploy a Python Data App to Kubernetes](#9-lab-deploy-a-python-data-app-to-kubernetes)
10. [Quick Reference](#10-quick-reference)

---

## 1. Why Data Engineers Need Kubernetes

Kubernetes (often abbreviated as K8s) is the industry standard platform for running containerized workloads at scale. You built Docker images in Module 7; Kubernetes is the system that runs, manages, and scales those images in production.

As a data engineering consultant, you are unlikely to be asked to set up or administer a Kubernetes cluster. That is a platform or DevOps engineer's responsibility. However, you will regularly need to:

- Deploy your data pipelines or services to an existing cluster
- Read logs and debug a failing pod
- Scale a workload up or down
- Understand why a deployment is stuck or crashing
- Work with Airflow or Spark running on Kubernetes

This module gives you the operational knowledge to do those things confidently without needing to be a Kubernetes administrator.

---

## 2. Core Concepts

### Cluster
A Kubernetes cluster is a set of machines (nodes) that run containerized workloads. It has a **control plane** (manages the cluster) and **worker nodes** (run the actual containers). On AKS or EKS, Microsoft and AWS manage the control plane for you.

### Node
A node is a single machine (virtual or physical) in the cluster. Each node runs one or more pods.

### Pod
A pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network and storage. In most cases, one pod contains one container.

Pods are ephemeral; they can be created, killed, and recreated at any time. You never connect to a pod expecting it to be there forever; that is why Deployments exist.

### Deployment
A Deployment manages a set of identical pods. It ensures the desired number of pods are always running. If a pod crashes, the Deployment automatically creates a replacement. This is the standard way to run a long-running application on Kubernetes.

### Service
A Service provides a stable network address for a set of pods. Because pods are ephemeral and their IP addresses change, a Service gives them a fixed endpoint that other applications can connect to.

### Namespace
A namespace is a virtual partition within a cluster. Teams or applications are separated into namespaces to avoid name collisions and to manage permissions. You will often be given access to a specific namespace on a client cluster rather than the whole cluster.

### ConfigMap
A ConfigMap stores non-sensitive configuration data as key-value pairs. Applications read this config at runtime without it being hardcoded in the container image.

### Secret
A Kubernetes Secret stores sensitive data (passwords, tokens, keys) in base64-encoded form. Pods reference Secrets as environment variables or mounted files. Note: Kubernetes Secrets are not encrypted by default; on production clusters, they are typically integrated with Azure Key Vault or AWS Secrets Manager for proper encryption.

---

## 3. Setting Up a Local Cluster with Minikube

Minikube runs a single-node Kubernetes cluster on your local machine inside a VM or Docker container. It is the standard way to learn and test Kubernetes locally.

### Install Minikube

**PowerShell:**
```powershell
winget install Kubernetes.minikube
```

**WSL:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### Install kubectl

kubectl is the command-line tool for interacting with any Kubernetes cluster.

**PowerShell:**
```powershell
winget install Kubernetes.kubectl
```

**WSL:**
```bash
sudo apt update && sudo apt install -y kubectl
```

### Start Minikube

```powershell
minikube start --driver=docker
```

This uses Docker Desktop (already installed from Module 7) as the underlying VM. The first start takes a few minutes.

Verify the cluster is running:

```powershell
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.x.x
```

### Useful Minikube Commands

```powershell
minikube status          # Check cluster status
minikube stop            # Stop the cluster (preserves state)
minikube delete          # Delete the cluster entirely
minikube dashboard       # Open the Kubernetes web UI in your browser
```

---

## 4. kubectl Essentials

kubectl is how you interact with a Kubernetes cluster. All commands follow the pattern:

```
kubectl <verb> <resource-type> <resource-name> [flags]
```

### Getting Information

```powershell
# List all pods in the current namespace
kubectl get pods

# List all pods in all namespaces
kubectl get pods --all-namespaces

# List deployments
kubectl get deployments

# List services
kubectl get services

# List everything
kubectl get all

# Get detailed information about a specific pod
kubectl describe pod <pod-name>

# Get YAML definition of a running resource
kubectl get pod <pod-name> -o yaml
```

### Viewing Logs

```powershell
# View logs from a pod
kubectl logs <pod-name>

# Follow logs in real time
kubectl logs -f <pod-name>

# View logs from a specific container in a multi-container pod
kubectl logs <pod-name> -c <container-name>

# View last N lines
kubectl logs <pod-name> --tail=100
```

### Executing Commands in a Pod

```powershell
# Open an interactive shell inside a pod
kubectl exec -it <pod-name> -- bash

# Run a single command
kubectl exec <pod-name> -- python --version
```

### Applying and Deleting Resources

```powershell
# Apply a manifest file (creates or updates the resource)
kubectl apply -f deployment.yaml

# Apply all manifests in a folder
kubectl apply -f ./manifests/

# Delete a resource
kubectl delete -f deployment.yaml

# Delete a specific pod (Deployment will recreate it)
kubectl delete pod <pod-name>
```

### Scaling

```powershell
# Scale a deployment to 3 replicas
kubectl scale deployment <deployment-name> --replicas=3

# Check the rollout status
kubectl rollout status deployment/<deployment-name>
```

### Switching Namespaces

```powershell
# Run a command in a specific namespace
kubectl get pods -n <namespace-name>

# Set a default namespace for your session
kubectl config set-context --current --namespace=<namespace-name>
```

---

## 5. Writing Kubernetes Manifests

A manifest is a YAML file that describes a Kubernetes resource. You apply manifests to the cluster with `kubectl apply -f`.

### 5.1 Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-processor
  namespace: default
  labels:
    app: data-processor
spec:
  replicas: 2                     # Run 2 pods at all times
  selector:
    matchLabels:
      app: data-processor
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      containers:
        - name: data-processor
          image: myregistry.azurecr.io/data-processor:latest
          resources:
            requests:
              memory: "128Mi"     # Minimum memory guaranteed
              cpu: "100m"         # 100 millicores = 0.1 CPU
            limits:
              memory: "256Mi"     # Maximum allowed
              cpu: "500m"
          env:
            - name: DATA_SOURCE
              value: "azure_blob"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret   # Name of the Kubernetes Secret
                  key: password
```

### 5.2 Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: data-processor-service
spec:
  selector:
    app: data-processor           # Routes traffic to pods with this label
  ports:
    - protocol: TCP
      port: 80                    # Port the service listens on
      targetPort: 8080            # Port the container listens on
  type: ClusterIP                 # Only accessible within the cluster
```

**Service types:**

| Type | Access |
|---|---|
| `ClusterIP` | Only within the cluster (default) |
| `NodePort` | Exposed on each node's IP at a specific port |
| `LoadBalancer` | Creates a cloud load balancer with a public IP (AKS/EKS) |

### 5.3 ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATA_SOURCE: azure_blob
  BATCH_SIZE: "1000"
  LOG_LEVEL: INFO
```

Reference in a Deployment:

```yaml
env:
  - name: DATA_SOURCE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DATA_SOURCE
```

Or mount the entire ConfigMap as environment variables:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

### 5.4 Secret

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: bXktc2VjcmV0LXBhc3N3b3Jk   # base64 encoded value
```

Generate a base64 value:

```powershell
# PowerShell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("my-secret-password"))

# WSL
echo -n "my-secret-password" | base64
```

> On production clusters, avoid creating Secrets manually like this. Use Azure Key Vault or AWS Secrets Manager integration instead (covered in Module 6).

---

## 6. Kubernetes on Azure (AKS) and AWS (EKS)

### 6.1 Connecting to an AKS Cluster

```powershell
# Download the kubeconfig credentials for the cluster
az aks get-credentials \
  --resource-group <resource-group> \
  --name <cluster-name>

# Verify connection
kubectl get nodes
```

After running `get-credentials`, kubectl is configured to talk to the AKS cluster. All subsequent kubectl commands go to that cluster until you switch context.

### 6.2 Connecting to an EKS Cluster

```powershell
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name <cluster-name>

kubectl get nodes
```

### 6.3 Switching Between Clusters

When you work on multiple clients, you will have multiple cluster credentials configured. Use contexts to switch:

```powershell
# List all configured clusters/contexts
kubectl config get-contexts

# Switch to a different cluster
kubectl config use-context <context-name>

# See which cluster you are currently connected to
kubectl config current-context
```

### 6.4 Key Differences: AKS vs EKS

| | AKS | EKS |
|---|---|---|
| **Auth** | Azure Active Directory | AWS IAM |
| **Node types** | Azure VM sizes | EC2 instance types |
| **Container registry** | Azure Container Registry (ACR) | Amazon ECR |
| **Secret integration** | Azure Key Vault (via CSI driver) | AWS Secrets Manager (via CSI driver) |
| **Load balancer** | Azure Load Balancer | AWS ELB/ALB |
| **CLI command** | `az aks get-credentials` | `aws eks update-kubeconfig` |

---

## 7. Running Data Workloads on Kubernetes

### 7.1 One-Off Jobs

For a data pipeline that runs to completion (not a long-running service), use a Kubernetes **Job** instead of a Deployment:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: daily-ingestion
spec:
  template:
    spec:
      containers:
        - name: ingestion
          image: myregistry.azurecr.io/ingestion-pipeline:latest
          env:
            - name: RUN_DATE
              value: "2024-01-15"
      restartPolicy: Never      # Do not restart if it fails; let it fail cleanly
  backoffLimit: 2               # Retry up to 2 times on failure
```

Apply and monitor:

```powershell
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/daily-ingestion
```

### 7.2 Scheduled Jobs (CronJobs)

For pipelines that run on a schedule:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-ingestion-cron
spec:
  schedule: "0 6 * * *"        # Every day at 6 AM UTC
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: ingestion
              image: myregistry.azurecr.io/ingestion-pipeline:latest
          restartPolicy: OnFailure
```

### 7.3 Airflow on Kubernetes (KubernetesExecutor)

When Airflow runs with the KubernetesExecutor, each task in a DAG gets its own pod. When a task is triggered, Airflow creates a pod, runs the task, and deletes the pod when done.

As a data engineer, what you need to know:

- **Task logs** are in the pod logs; use `kubectl logs` to view them if Airflow UI is not showing them
- **Failed tasks** may leave behind failed pods; `kubectl get pods` will show them with status `Error` or `OOMKilled`
- **OOMKilled** means the pod ran out of memory; the task needs more memory resources configured in its Kubernetes pod spec override

### 7.4 Spark on Kubernetes

When Spark jobs run on Kubernetes, the Spark driver and executor each get their own pod. Key things to know:

- The driver pod submits the job and coordinates executors
- Watch the driver pod logs for job progress: `kubectl logs <spark-driver-pod>`
- Executor pods are created dynamically and deleted when the job finishes

---

## 8. Helm Basics

Helm is the package manager for Kubernetes. Instead of writing and managing dozens of YAML manifests yourself, Helm lets you install, configure, and upgrade pre-packaged Kubernetes applications called **charts**.

Think of Helm like `pip` for Kubernetes.

### 8.1 Installing Helm

**PowerShell:**
```powershell
winget install Helm.Helm
```

**WSL:**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:
```powershell
helm version
```

### 8.2 Key Helm Concepts

**Chart:** A package of Kubernetes manifests. Charts are stored in repositories.

**Release:** An installed instance of a chart. You can install the same chart multiple times with different configurations; each installation is a separate release.

**Values:** Configuration options that customize a chart. You override defaults using a `values.yaml` file or `--set` flags.

**Repository:** A collection of charts. Similar to a package registry.

### 8.3 Common Helm Commands

```powershell
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo airflow

# See default values for a chart
helm show values bitnami/airflow

# Install a chart
helm install my-airflow bitnami/airflow \
  --namespace airflow \
  --create-namespace \
  --values my-values.yaml

# List installed releases
helm list --all-namespaces

# Upgrade an existing release
helm upgrade my-airflow bitnami/airflow --values my-values.yaml

# Uninstall a release
helm uninstall my-airflow --namespace airflow
```

### 8.4 Custom Values File

Instead of accepting all defaults, create a `values.yaml` to configure the chart:

```yaml
# my-airflow-values.yaml
executor: KubernetesExecutor

web:
  replicas: 1

scheduler:
  replicas: 1

postgresql:
  enabled: true
  auth:
    password: "dev-password-only"
```

Apply with:
```powershell
helm install my-airflow bitnami/airflow -f my-airflow-values.yaml --namespace airflow --create-namespace
```

---

## 9. Lab: Deploy a Python Data App to Kubernetes

This lab deploys a Python app to a local Minikube cluster, exposes it as a service, scales it, and inspects logs.

### Prerequisites
- Minikube and kubectl installed (Section 3)
- Docker installed (Module 7)
- Minikube running (`minikube start`)

---

### Step 1: Prepare the App and Build the Image

Create a folder called `k8s-practice`. Inside it, create the following files:

`src/main.py`:
```python
import os
import time

def process_data():
    source = os.environ.get("DATA_SOURCE", "local")
    destination = os.environ.get("DATA_DESTINATION", "output")

    print(f"Starting data processing job")
    print(f"Source: {source}")
    print(f"Destination: {destination}")

    records = [
        {"id": 1, "name": "Alice", "score": 95},
        {"id": 2, "name": "Bob", "score": 87},
        {"id": 3, "name": "Charlie", "score": 92},
    ]

    print(f"Processed {len(records)} records")
    print("Job complete. Staying alive for lab exercises...")

    # Keeps the container running so kubectl exec and log exercises work.
    # In a real pipeline job you would remove this.
    time.sleep(120)

if __name__ == "__main__":
    process_data()
```

`Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY src/ ./src/
ENV PYTHONUNBUFFERED=1
CMD ["python", "src/main.py"]
```

Build the image locally:

```powershell
docker build -t data-processor:latest .
```

---

### Step 2: Load the Image into Minikube

Minikube runs in its own Docker environment. Load the locally built image into it so Kubernetes can use it without pulling from a registry:

```powershell
minikube image load data-processor:latest
```

Verify it is available:

```powershell
minikube image ls
```

---

### Step 3: Create a ConfigMap

Create `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: processor-config
data:
  DATA_SOURCE: minikube-local
  DATA_DESTINATION: local-output
```

Apply it:

```powershell
kubectl apply -f configmap.yaml
kubectl get configmaps
```

---

### Step 4: Create a Deployment

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-processor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-processor
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      containers:
        - name: data-processor
          image: data-processor:latest
          imagePullPolicy: Never      # Use local image; do not try to pull from registry
          envFrom:
            - configMapRef:
                name: processor-config
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

Apply it:

```powershell
kubectl apply -f deployment.yaml
kubectl get pods
```

Wait until the pod status shows `Running`. The `time.sleep(120)` in the script keeps it alive long enough to complete all lab steps.

---

### Step 5: View Logs

```powershell
# Get the pod name
kubectl get pods

# View its logs
kubectl logs <pod-name>
```

You should see the output from `src/main.py` including the DATA_SOURCE value from the ConfigMap.

---

### Step 6: Describe the Pod

```powershell
kubectl describe pod <pod-name>
```

Look at the Events section at the bottom. This is where Kubernetes reports what happened when it scheduled and started the pod; very useful for debugging.

---

### Step 7: Scale the Deployment

```powershell
kubectl scale deployment data-processor --replicas=3
kubectl get pods
```

You should see 3 pods. Scale back down:

```powershell
kubectl scale deployment data-processor --replicas=1
```

---

### Step 8: Open an Interactive Shell

The `time.sleep(120)` in the script keeps the pod running for 2 minutes; long enough to complete this step. Open a shell inside the running pod:

```powershell
kubectl exec -it <pod-name> -- bash
```

Inside the container:
```bash
ls /app
cat /app/src/main.py
python --version
exit
```

---

### Step 9: Clean Up

```powershell
kubectl delete -f deployment.yaml
kubectl delete -f configmap.yaml
minikube stop
```

---

## 10. Quick Reference

| Command | What it does |
|---|---|
| `kubectl get pods` | List pods in current namespace |
| `kubectl get pods -n <ns>` | List pods in a specific namespace |
| `kubectl get all` | List all resources |
| `kubectl describe pod <name>` | Detailed info and events for a pod |
| `kubectl logs <pod>` | View pod logs |
| `kubectl logs -f <pod>` | Follow pod logs |
| `kubectl exec -it <pod> -- bash` | Open shell in pod |
| `kubectl apply -f file.yaml` | Create or update resource from file |
| `kubectl delete -f file.yaml` | Delete resource defined in file |
| `kubectl delete pod <name>` | Delete a specific pod |
| `kubectl scale deployment <name> --replicas=N` | Scale a deployment |
| `kubectl rollout status deployment/<name>` | Check rollout progress |
| `kubectl config get-contexts` | List configured clusters |
| `kubectl config use-context <name>` | Switch cluster |
| `kubectl config current-context` | Show active cluster |
| `minikube start` | Start local cluster |
| `minikube stop` | Stop local cluster |
| `minikube image load <image>` | Load local Docker image into Minikube |
| `helm install <release> <chart>` | Install a Helm chart |
| `helm list` | List installed Helm releases |
| `helm upgrade <release> <chart>` | Upgrade a release |
| `helm uninstall <release>` | Remove a release |

---

*Next module: Module 9: Advanced dbt CI/CD Patterns*
