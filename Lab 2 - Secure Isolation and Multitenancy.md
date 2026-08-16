# Lab Report: Secure Isolation & Multi-Tenancy

**Course**: IKB42603 Cloud Computing Security Essentials  
**Institution**: UniKL MIIT — Prof. Dr. Shahrulniza Musa  
**Topic**: Compute, Network, and Storage Isolation — Docker & Kubernetes  
**Student/User Identity**: Rafissah (`raff@Raff`)  

---

## Course & Assessment Mapping

- **Course Learning Outcome (CLO)**: CLO2 — Construct secure cloud operations that safeguard data integrity
- **Lecture Topics**: Week 3 (Secure Isolation of Physical & Logical Infrastructure)
- **Value / Skill Clusters**: VBE3 (Integrity) · SC8 (Integrated Problem-Solving)
- **Assessment**: Lab Report (Screenshots + CLI Output + Short Answers) — contributes to the Lab Assignment

---

## Executive Summary

This lab report documents the implementation and evaluation of multi-tenant isolation mechanisms within a shared Kubernetes cluster infrastructure (`ccse-lab2`). Multi-tenancy introduces significant security risks when workloads from different organizations or units share the same physical hardware, control plane, and network substrate. 

The lab is divided into two primary sessions:
1. **Session A (Week 3)**: Compute isolation through Kubernetes namespaces, demonstrating the inherent risk of default-open inter-pod networking across namespaces, and mitigating noisy-neighbor resource exhaustion using `ResourceQuota`.
2. **Session B (Week 4)**: Network and storage isolation via default-deny `NetworkPolicy` enforced by Project Calico CNI, fine-grained secret access control using Kubernetes RBAC (`ServiceAccount`, `Role`, `RoleBinding`), and demonstrating data remanence alongside secure data wiping in container storage volumes.

---

# Session A (Week 3) — Compute Isolation & the Default-Open Risk

## Setup — Cluster with Policy Enforcement

To evaluate network policy enforcement, a local Kubernetes cluster was created using `kind` with the default CNI disabled (`disableDefaultCNI: true`). Project Calico was subsequently deployed as the CNI plugin to enforce network isolation rules.

```bash
# 1. Create a Kind cluster with default CNI disabled
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF
```
![Kind Cluster Creation](EVIDENCE/setup%201%20.png)

```bash
# 2. Deploy Project Calico manifest for network policy enforcement
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```
![Apply Calico CNI Manifest](EVIDENCE/setup%202.png)

```bash
# 3. Verify Calico controller and node daemonset health
kubectl get pods -n kube-system
```
![Verify Calico Deployment](EVIDENCE/setup%20calico%20verification.png)

*Observation*: As shown in `EVIDENCE/setup calico verification.png`, `calico-kube-controllers` and `calico-node` pods successfully entered the `Running` state in the `kube-system` namespace.

---

## Task 1 — Two Tenants on One Cluster

To model a multi-tenant cloud environment where two separate customers share physical host infrastructure, two logical boundaries (`tenant-a` and `tenant-b` namespaces) were created, each running an isolated `nginx` web deployment exposed on port 80.

### Step 1.1: Create Tenant Namespaces
```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```
![Create Tenant Namespaces](EVIDENCE/task%201%20create%20tenant%20a%20%26%20b.png)

### Step 1.2: Deploy Nginx Web Server for Each Tenant
```bash
# Deploy web server in tenant-a
kubectl -n tenant-a create deployment web --image=nginx
```
![Deploy Web Server in tenant-a](EVIDENCE/task%201%20Deploy%20nginx%20in%20tenant-a.png)

```bash
# Deploy web server in tenant-b
kubectl -n tenant-b create deployment web --image=nginx
```
![Deploy Web Server in tenant-b](EVIDENCE/task%201%20Deploy%20nginx%20in%20tenant-b.png)

### Step 1.3: Expose Deployments as ClusterIP Services
```bash
# Expose tenant-a web service
kubectl -n tenant-a expose deployment web --port=80
```
![Expose Service in tenant-a](EVIDENCE/task%201%20Expose%20the%20tenant-a%20web%20deployment.png)

```bash
# Expose tenant-b web service
kubectl -n tenant-b expose deployment web --port=80
```
![Expose Service in tenant-b](EVIDENCE/task%201Expose%20the%20tenant-b%20web%20deployment.png)

### Step 1.4: Verify Running Pods and Services in Each Namespace
```bash
kubectl get pods,svc -n tenant-a
```
![Verify Pods and Services in tenant-a](EVIDENCE/task%201%20Check%20pods%20and%20services%20in%20tenant-a.png)

```bash
kubectl get pods,svc -n tenant-b
```
![Verify Pods and Services in tenant-b](EVIDENCE/task%201%20Check%20pods%20and%20service%20in%20tenant-b.png)

*Actual Results*:
- `tenant-a`: Pod `pod/web-68d995574f-tf6hz` (`1/1 Running`), Service `service/web` (`ClusterIP 10.96.111.213`, Port 80/TCP).
- `tenant-b`: Pod `pod/web-68d995574f-dvd9m` (`1/1 Running`), Service `service/web` (`ClusterIP 10.96.182.98`, Port 80/TCP).

---

## Task 2 — Observe the Default-Open Risk

By default, Kubernetes uses a flat network model where pods across different namespaces can directly communicate with one another without restriction. To demonstrate this security vulnerability, a test probe container was launched in `tenant-a` to query `tenant-b`'s internal web service.

### Step 2.1: Retrieve Tenant B's ClusterIP
```bash
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
```
![Get Tenant B Service IP](EVIDENCE/task%202%20Get%20tenant-b's%20service%20IP.png)

*Output*: `10.96.182.98`

### Step 2.2: Execute Cross-Tenant Probe from Tenant A to Tenant B
```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.182.98 -o /dev/null -w 'HTTP %{http_code}\n'
```
![Test Cross-Tenant Reachability](EVIDENCE/task%202%20Test%20from%20tenant-a%20%E2%86%92%20tenant-b.png)

*Actual Results*:
```text
HTTP 200
pod "probe" deleted
```

> **Security Risk Alert**: Returning `HTTP 200` proves that logical separation via namespaces alone does **NOT** provide network isolation. An attacker inside `tenant-a` can freely probe and exploit internal applications hosted by `tenant-b`.

---

## Task 3 — Contain the Noisy Neighbour (Resource Quotas)

In multi-tenant environments, a single tenant can monopolize shared node compute resources (CPU, RAM, max pod count), starving adjacent tenants ("noisy-neighbor" syndrome). To prevent resource exhaustion, a `ResourceQuota` object was declared for `tenant-a`.

### Step 3.1: Apply ResourceQuota to Tenant A
```bash
kubectl -n tenant-a apply -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
EOF
```
![Apply Resource Quota](EVIDENCE/task%203%20Resource%20Quota.png)

### Step 3.2: Verify Enforced Resource Quota
```bash
kubectl get resourcequota -n tenant-a
```
![Verify Resource Quota](EVIDENCE/task%203%20Verify%20the%20quota.png)

*Actual Results*:
```text
NAME             REQUEST                                     LIMIT                               AGE
tenant-a-quota   pods: 1/5, requests.cpu: 0/1, requests.memory: 0/1Gi   limits.cpu: 0/2, limits.memory: 0/2Gi   80s
```

---

# Session B (Week 4) — Network & Storage Isolation

## Task 4 — Default-Deny Network Isolation

To eliminate the default-open cross-tenant vulnerability observed in Task 2, a default-deny ingress `NetworkPolicy` was enforced on `tenant-b`. This adheres to the security principle of **least privilege and zero-trust segmentation**: deny all inbound traffic by default, permit only explicitly authorized paths.

### Step 4.1: Apply Default-Deny Ingress Policy to Tenant B
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```
![Apply Default Deny NetworkPolicy](EVIDENCE/session%20b%20task%204.png)

### Step 4.2: Re-run Cross-Tenant Probe from Tenant A to Tenant B
```bash
# Execute probe and inspect execution log
kubectl logs -n tenant-a probe
```
![Re-run Cross-Tenant Probe After Policy](EVIDENCE/session%20b%20task%204.2.png)

*Actual Results*:
```text
HTTP 000
```

*Comparison Analysis*:
- **Before Policy (Task 2)**: Probe output returned `HTTP 200` (successful cross-tenant HTTP connection).
- **After Policy (Task 4)**: Probe output returned `HTTP 000` (connection timed out and was blocked by Calico CNI).

---

## Task 5 — Storage & Secret Isolation

Multi-tenancy requires strict isolation of sensitive credentials, API keys, and secret stores. Kubernetes Role-Based Access Control (RBAC) was configured to verify that service accounts in `tenant-a` cannot read secrets belonging to `tenant-b`.

### Step 5.1: Create Secrets in Both Tenant Namespaces
```bash
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B
```
![Create Tenant Secrets](EVIDENCE/session%20b%20task%205%20Create%20a%20secret%20in%20each%20tenant.png)

### Step 5.2: Create Scoped ServiceAccount, Role, and RoleBinding in Tenant A
```bash
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a
```
![Create ServiceAccount and RBAC Rules](EVIDENCE/session%20b%20task%205%20A%20service%20account%20scoped%20to%20tenant-a%20only.png)

### Step 5.3: Verify Authorization Boundaries (`kubectl auth can-i`)
```bash
SA=system:serviceaccount:tenant-a:app-a

# Test access to tenant-a secrets (Expected: yes)
kubectl auth can-i get secrets -n tenant-a --as=$SA

# Test access to tenant-b secrets (Expected: no)
kubectl auth can-i get secrets -n tenant-b --as=$SA
```
![Verify RBAC Secret Access](EVIDENCE/session%20b%20task%205%20Check%20tenant-a%20permission%20and%20test%20access%20to%20tenant-b.png)

*Actual Results*:
- `kubectl auth can-i get secrets -n tenant-a --as=$SA` $\rightarrow$ **`yes`**
- `kubectl auth can-i get secrets -n tenant-b --as=$SA` $\rightarrow$ **`no`**

---

## Task 6 — Data Remanence & Secure Deletion

Data remanence occurs when unlinked file data persists on underlying disk blocks after a standard file system `rm` command. In a shared storage architecture, residual data can be inspected by unauthorized parties.

### Step 6.1: Demonstrate Standard Deletion (File System Unlinking)
```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
  grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```
![Standard Deletion Scan](EVIDENCE/session%20b%20task%206%20.png)

*Actual Output*: `scan-done`

### Step 6.2: Demonstrate Secure Wipe (Zero-Fill Overwrite via `dd`)
```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
  dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
  echo wiped'
```
![Secure Wipe Output](EVIDENCE/session%20b%20task%206.2.png)

*Actual Output*:
```text
1+0 records in
1+0 records out
1024 bytes (1.0KB) copied, 0.000122 seconds, 8.0MB/s
wiped
```

---

## Deliverables & Assessment

### 1. Verification Commands Output
```bash
# Verify active network policies across all namespaces
kubectl get networkpolicy -A

# Describe applied resource quota in tenant-a
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

### 2. Short-Answer Questions

#### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?
> **Answer**: Kubernetes namespaces are logical organizational boundaries used for API object grouping and RBAC scoping; they do not provide automatic network segmentation. Under standard CNI models, Kubernetes implements a flat pod-to-pod network where any pod can assign an IP and route traffic to any other pod across namespaces. In a multi-tenant cloud, this default-open behavior allows compromised containers or malicious tenants to perform IP scanning, lateral movement, and unauthenticated network attacks against services hosted in adjacent tenant namespaces.

#### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.
> **Answer**: The default-deny principle mandates that all traffic is blocked by default, and access is granted strictly through explicit allow lists (Zero Trust model). In Task 4, the `default-deny-ingress` `NetworkPolicy` specifies an empty `podSelector: {}` matching all pods within `tenant-b` and declares `policyTypes: [Ingress]`. Because no ingress `from` rules are defined, the Calico CNI drops all incoming network packets directed at `tenant-b` pods, changing the cross-tenant probe response from `HTTP 200` to a timeout (`HTTP 000`).

#### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?
> **Answer**: Containers share the underlying host operating system kernel and rely on Linux kernel features (`namespaces`, `cgroups`, `seccomp`) for isolation. A kernel vulnerability or container escape flaw can grant an attacker full host root access. Virtual Machines (VMs) utilize hypervisors to provide hardware-level virtualization, giving each VM its own dedicated kernel. A VM boundary (or sandboxed container runtime like gVisor/Kata Containers) should be added when executing untrusted tenant code, processing sensitive untrusted payloads, or complying with strict regulatory standards (PCI-DSS, HIPAA) in multi-tenant environments.

#### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?
> **Answer**: Data remanence is the persistent physical representation of data that remains on storage media even after logical file deletion. In multi-tenant cloud infrastructure, physical storage hardware is shared across tenants and managed entirely by the cloud provider, making low-level physical disk overwriting (`dd`, degaussing, physical destruction) impractical or impossible for tenants. Cryptographic erasure (crypto-shredding)—where data is stored encrypted and deletion is performed by securely destroying the encryption key—renders the remaining ciphertext permanently unrecoverable without requiring physical disk access.

#### Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?
> **Answer**:
> - **Task 1 & Task 3**: **Compute Isolation** — Multi-tenant logical isolation using namespaces (Task 1) and resource capacity boundary enforcement via `ResourceQuota` (Task 3).
> - **Task 2 & Task 4**: **Network Isolation** — Observing default-open CNI inter-pod networking (Task 2) and enforcing default-deny ingress network policy via Calico CNI (Task 4).
> - **Task 5 & Task 6**: **Storage & Secret Isolation** — Scoping secret read permissions using Kubernetes RBAC (Task 5) and evaluating volume data remanence and secure block overwriting (Task 6).

---

## Security Best-Practices Checklist

| Security Control | Implementation Status | Description |
| :--- | :---: | :--- |
| **Namespace Isolation** | ✅ Completed | Tenants separated into distinct logical namespaces (`tenant-a`, `tenant-b`). |
| **Default-Deny Network Policy** | ✅ Completed | Cross-tenant ingress traffic blocked using `default-deny-ingress` NetworkPolicy (verified before `HTTP 200` vs after `HTTP 000`). |
| **Resource Quotas** | ✅ Completed | `ResourceQuota` enforced in `tenant-a` to prevent noisy-neighbor CPU/RAM/pod starvation. |
| **RBAC Storage/Secret Isolation** | ✅ Completed | Per-tenant secrets configured; `app-a` ServiceAccount permitted in `tenant-a` (`yes`) and blocked in `tenant-b` (`no`). |
| **Data Remanence & Secure Wipe** | ✅ Completed | Demonstrated file system unlinking vs zero-fill overwrite (`dd`) and cryptographic erasure principles. |

---

## Cleanup & Teardown

To tear down the laboratory environment and release local system resources:

```bash
# Delete local Kind Kubernetes cluster
kind delete cluster --name ccse-lab2

# Remove Docker persistent volume
docker volume rm ccse-vol
```
