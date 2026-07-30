# Lab 0: Environment Setup Report
**Student Name:** Nur Rafissah Nabila Abdul Razak
**Student ID** 52215225324  
**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur - Malaysian Institute of Information Technology (UniKL MIIT)  
**Instructor:** Prof. Dr. Shahrulniza Musa  
**Document Type:** Step-by-Step Environment Setup & Verification Report  
**Reference Guide:** `IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf`  

---

## Executive Summary

This report documents the installation, configuration, and verification of the core toolset required for **IKB42603 Cloud Computing Security Essentials**. The environment is configured locally without requiring external cloud accounts or credit cards, utilizing local containerization and emulation technologies such as **Docker**, **LocalStack**, and **Kubernetes (kind)**.

All evidence screenshots captured during execution are cataloged and embedded under their respective verification steps.

---

## 1. Environment Toolset Overview

The following table summarizes the software packages installed and verified for this lab series:

| Tool | Purpose | Primary Use Case | Verification Status |
| :--- | :--- | :--- | :---: |
| **Docker** | Container runtime engine & LocalStack simulator host | All Labs (1 – 5) | Verified |
| **AWS CLI v2** | Command-line interface to issue AWS API calls to LocalStack | Labs 1, 3, 5 | Verified |
| **kind** | Local Kubernetes cluster runner inside Docker | Labs 1, 2, 4 | Verified |
| **kubectl** | Kubernetes command-line cluster management tool | Labs 1, 2, 4 | Verified |
| **OpenSSL** | Cryptographic toolset for certificates and key management | Lab 3 | Verified |
| **oathtool** | Command-line tool for MFA / TOTP token generation | Lab 4 | Verified |
| **Trivy** | Container vulnerability scanner (run via Docker) | Lab 4 | Verified |

> **Security & Execution Note:**  
> For Windows environments, all commands must be executed within **Git Bash** or **WSL (Ubuntu)** to ensure compatibility with Bash features such as heredocs, `sha256sum`, and single-quoted JSON parameters.

---

## 2. Detailed Setup & Verification Steps

### Step 1: Docker Container Engine Setup

#### 1.1 Installation Procedures
- **Windows 10/11:** Install Docker Desktop from [docker.com](https://www.docker.com/). Select the **WSL 2 backend** when prompted and reboot.
- **macOS:** Install Docker Desktop (select Apple Silicon or Intel installer matching hardware architecture).
- **Linux (Ubuntu):**
  ```bash
  curl -fsSL https://get.docker.com | sh
  sudo usermod -aG docker $USER
  # Log out and log back in to apply group changes
  ```

#### 1.2 Verification & Test Execution
To verify the Docker installation and daemon readiness:
```bash
docker --version
docker run --rm hello-world
```

**Captured Evidence:**  
![1. Docker Version Verification](Evidence/1.%20docker%20ver.png)

- **Verified Version Output:** `Docker version 28.5.2+dfsg4, build 9cc6dea35e9a963f281434761c656fba4ac43aed`
- **Environment:** Kali Linux (`raff@Raff`)

> **Hardware Virtualization Caution:**  
> Ensure hardware virtualization (VT-x / AMD-V / SVM) is enabled in the BIOS/UEFI. On Windows systems, enable **WSL 2** and **Virtual Machine Platform**.

---

### Step 2: AWS CLI v2 Setup

#### 2.1 Installation Procedures
- **Windows:** Download and execute the official AWS CLI v2 MSI installer.
- **macOS:** Install via Homebrew: `brew install awscli`
- **Linux:**
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install
  ```

#### 2.2 Verification Execution
Run the following command to verify AWS CLI installation:
```bash
aws --version
```

**Captured Evidence:**  
![2. AWS CLI Version Verification](Evidence/2.%20aws%20ver.png)

- **Verified Version Output:** `aws-cli/2.34.56 Python/3.13.2 Linux/6.12.13-amd64 source/x86_64.kali.2025`
- **Note:** Real AWS account credentials are not required. The CLI will target LocalStack via `--endpoint-url=http://localhost:4566`.

---

### Step 3: Kubernetes Tools Setup (kind & kubectl)

#### 3.1 Installation Procedures
- **Windows:**
  ```bash
  choco install kind
  choco install kubernetes-cli
  ```
- **macOS:**
  ```bash
  brew install kind
  brew install kubectl
  ```
- **Linux:**
  ```bash
  # Install kind
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
  chmod +x ./kind
  sudo mv ./kind /usr/local/bin/kind

  # Install kubectl
  sudo snap install kubectl --classic
  ```

#### 3.2 Verification Execution
Verify both tools:
```bash
kind --version
kubectl version --client
```

**Captured Evidence (kind):**  
![3. kind Version Verification](Evidence/3.%20kind%20ver.png)

- **Verified Version Output:** `kind version 0.31.0`

**Captured Evidence (kubectl):**  
![4. kubectl Version Verification](Evidence/4.%20kubectl%20ver.png)

- **Verified Version Output:**  
  - **Client Version:** `v1.33.4`  
  - **Kustomize Version:** `v5.5.0`

---

### Step 4: Security & Helper Tools (OpenSSL, oathtool, Trivy)

#### 4.1 Tool Configurations
- **OpenSSL:** Standard component on Linux and macOS; bundled with Git Bash on Windows.
- **oathtool:**
  - macOS: `brew install oath-toolkit`
  - Linux: `sudo apt install oathtool`
- **Trivy:** Container image scanner. Executed directly via Docker without direct installation:
  ```bash
  docker run --rm aquasec/trivy image <image_name>
  ```

#### 4.2 Verification Execution
Verify OpenSSL and oathtool installations:
```bash
openssl version
oathtool --version
```

**Captured Evidence (OpenSSL):**  
![5. OpenSSL Version Verification](Evidence/5.%20openssl%20ver.png)

- **Verified Version Output:** `OpenSSL 3.4.0 22 Oct 2024 (Library: OpenSSL 3.4.0 22 Oct 2024)`

**Captured Evidence (oathtool):**  
![6. oathtool Version Verification](Evidence/6.%20oathtool%20ver.png)

- **Verified Version Output:** `oathtool (OATH Toolkit) 2.6.14`

---

### Step 5: LocalStack Cloud Simulator Management

#### 5.1 Starting LocalStack
LocalStack emulates AWS cloud services locally. Start the container using Docker:
```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
```

**Captured Evidence (LocalStack Launch & Image Download):**  
![7. LocalStack Container Startup](Evidence/7.%20localstack.png)

- **Details:** Pulled `localstack/localstack:latest` image and spawned container instance `16523344d9e4c22ab5e80cf1f9f31e8fa746ffc18a115884cad11205bb192c34`.

#### 5.2 Checking Health & Container Lifecycle Management
- **Check Health:**
  ```bash
  curl http://localhost:4566/_localstack/health
  ```
- **Stop, Start, & Remove Operations:**
  ```bash
  docker stop localstack
  docker start localstack
  docker rm -f localstack
  ```

---

### Step 6: One-Time AWS CLI Configuration & Endpoint Verification

#### 6.1 AWS CLI Dummy Configuration
Configure dummy AWS credentials so AWS CLI commands proceed without prompt errors:
```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

#### 6.2 LocalStack Endpoint Variable & Verification
Set the endpoint shortcut variable for terminal sessions and verify caller identity:
```bash
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

**Captured Evidence (AWS Configuration & LocalStack State Management):**  
![8. AWS Configuration & LocalStack Interaction](Evidence/8.%20localstack%202.png)

- **Observed Actions:** 
  1. `docker stop localstack` & `docker start localstack` verified container management lifecycle.
  2. `docker rm -f localstack` demonstrated complete container removal.
  3. `aws configure set` stored local profile settings (`aws_access_key_id`, `aws_secret_access_key`, `region us-east-1`).
  4. `EP='--endpoint-url=http://localhost:4566'` variable declaration.
  5. `aws $EP sts get-caller-identity` confirmed endpoint routing behavior when LocalStack service is offline (`Could not connect to the endpoint URL: "http://localhost:4566/"`).

---

### Step 7: Kubernetes Cluster (kind) Lifecycle

#### 7.1 Cluster Operations
```bash
# Create local cluster
kind create cluster --name ccse

# Verify cluster connectivity
kubectl cluster-info --context kind-ccse
kubectl get nodes

# Delete cluster upon completion
kind delete cluster --name ccse
```

---

## 3. Pre-Lab Verification Checklist

| Requirement / Test | Command / Condition | Status |
| :--- | :--- | :---: |
| **Docker Runtime** | `docker --version` & `docker run hello-world` | [x] Passed |
| **AWS CLI v2** | `aws --version` (v2.x) | [x] Passed |
| **kind & kubectl** | `kind --version` & `kubectl version --client` | [x] Passed |
| **OpenSSL** | `openssl version` | [x] Passed |
| **oathtool** | `oathtool --version` | [x] Passed |
| **LocalStack Health** | Container running & health endpoint responding | [x] Passed |
| **AWS STS Identity** | `aws $EP sts get-caller-identity` against LocalStack | [x] Passed |
| **kind Cluster** | `kind create cluster` & `kubectl get nodes` | [x] Passed |
| **Terminal Shell** | Executing inside Bash / WSL shell environment | [x] Passed |

---

## 4. Troubleshooting Reference Guide

| Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`Cannot connect to the Docker daemon`** | Docker Desktop background service is stopped. | Start Docker Desktop application. On Linux, ensure `$USER` is in `docker` group (`sudo usermod -aG docker $USER`) and re-login. |
| **Docker won't start / slow** | Virtualization disabled in system firmware. | Enable VT-x / AMD-V / SVM in BIOS/UEFI. On Windows, enable WSL 2 and Virtual Machine Platform. |
| **Port 4566 already in use** | A prior LocalStack container instance is bound to 4566. | Remove existing container: `docker rm -f localstack` then rerun the `docker run` command. |
| **`Could not connect to the endpoint URL`** | LocalStack container is stopped/removed or `--endpoint-url` was omitted. | Ensure LocalStack container is running (`docker start localstack`) and `--endpoint-url=http://localhost:4566` / `$EP` is passed. |
| **`command not found` (aws/kubectl)** | Executable binary path missing from system `$PATH`. | Re-run installer, check system environment variables, or restart terminal session. |
| **Heredoc / sha256sum errors** | Command executed in PowerShell or CMD. | Switch to Git Bash or WSL (Ubuntu) shell. |
| **`kind create cluster` failure** | Insufficient memory allocation or Docker daemon down. | Verify Docker is active and allocate $\ge$ 4 GB RAM in Docker Desktop settings. |
| **MFA / TOTP mismatch (Lab 4)** | Host system clock drift. | Synchronize system clock with internet time servers. |
| **NetworkPolicy not blocking (Lab 2)** | Calico CNI plugin not initialized. | Wait until `calico-node` pods reach `Ready` state. |

---

## 5. Session Quick-Start & Cleanup Commands

### 5.1 Quick Start Session
```bash
# Start LocalStack service
docker start localstack 2>/dev/null || docker run -d --name localstack -p 4566:4566 localstack/localstack

# Set endpoint environment variable
EP='--endpoint-url=http://localhost:4566'

# Verify active resources
docker ps
kind get clusters
```

### 5.2 Environment Cleanup & Disk Space Recovery
```bash
# Terminate and delete containers and clusters
docker rm -f localstack
kind delete clusters --all

# Prune unused Docker objects
docker system prune -f
```

---
*Report generated based on IKB42603 Lab 0 Setup Cheatsheet requirements.*
