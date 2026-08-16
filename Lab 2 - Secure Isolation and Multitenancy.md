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

This lab report documents the step-by-step implementation and empirical validation of secure multi-tenant isolation across compute, network, and storage layers within a shared Kubernetes cluster (`ccse-lab2`). In multi-tenant cloud environments, sharing physical hardware and control planes creates severe security risks such as lateral network movement, secret leakage, resource starvation, and residual data remanence.

The lab is divided into two comprehensive sessions:
1. **Session A (Week 3)**: Compute isolation using Kubernetes namespaces, demonstrating the default-open cross-tenant network risk (`HTTP 200`), and enforcing resource consumption limits via `ResourceQuota`.
2. **Session B (Week 4)**: Network isolation using a default-deny `NetworkPolicy` enforced by Calico CNI (`HTTP 000` timeout), storage and secret isolation using Kubernetes RBAC (`ServiceAccount`, `Role`, `RoleBinding`), and demonstrating data remanence versus secure zero-fill wiping on container storage volumes.

---

# Session A (Week 3) — Compute Isolation & the Default-Open Risk

## Setup — Cluster with Policy Enforcement

To evaluate network policy enforcement, a local Kubernetes cluster was created using `kind` with the default CNI disabled (`disableDefaultCNI: true`). Project Calico was subsequently deployed as the CNI plugin to enforce network isolation rules.

### Step 0.1: Create Kind Cluster with Default CNI Disabled
**What was performed**: Initialized a single-node `kind` cluster named `ccse-lab2` with `disableDefaultCNI: true` and `podSubnet: 192.168.0.0/16`.  
**Actual result shown in evidence**: Terminal output confirms `Creating cluster "ccse-lab2" ...`, node image `kindest/node:v1.35.0` prepared, control-plane started, StorageClass installed, and context set to `kind-ccse-lab2`.

![Kind Cluster Creation](EVIDENCE/setup%201%20.png)

### Step 0.2: Deploy Project Calico CNI Manifest
**What was performed**: Executed `kubectl apply -f` pointing to the official Calico v3.27.0 manifest to install the network policy enforcement engine.  
**Actual result shown in evidence**: Terminal displays creation of Calico CRDs (`globalnetworkpolicies`, `felixconfigurations`, `bgpconfigurations`, etc.), DaemonSet `calico-node`, and Deployment `calico-kube-controllers`.

![Apply Calico CNI Manifest](EVIDENCE/setup%202.png)

### Step 0.3: Verify Calico Control Plane Health
**What was performed**: Checked the status of all pods running in the `kube-system` namespace via `kubectl get pods -n kube-system`.  
**Actual result shown in evidence**: Both `calico-kube-controllers-6d76546c4c-wjbqk` and `calico-node-p9fhn` are in the `1/1 Running` state.

![Verify Calico Deployment](EVIDENCE/setup%20calico%20verification.png)

---

## Task 1 — Two Tenants on One Cluster

Model two separate customers (`tenant-a` and `tenant-b`) as independent Kubernetes namespaces sharing the same physical cluster infrastructure.

### Step 1.1: Create Tenant Namespaces
**What was performed**: Executed `kubectl create namespace tenant-a` and `kubectl create namespace tenant-b`.  
**Actual result shown in evidence**: Output confirms `namespace/tenant-a created` and `namespace/tenant-b created`.

![Create Tenant Namespaces](EVIDENCE/task%201%20create%20tenant%20a%20%26%20b.png)

### Step 1.2: Deploy Web Server for Tenant A
**What was performed**: Created an Nginx web deployment named `web` in `tenant-a` (`kubectl -n tenant-a create deployment web --image=nginx`).  
**Actual result shown in evidence**: Output confirms `deployment.apps/web created`.

![Deploy Web Server in tenant-a](EVIDENCE/task%201%20Deploy%20nginx%20in%20tenant-a.png)

### Step 1.3: Deploy Web Server for Tenant B
**What was performed**: Created an Nginx web deployment named `web` in `tenant-b` (`kubectl -n tenant-b create deployment web --image=nginx`).  
**Actual result shown in evidence**: Output confirms `deployment.apps/web created`.

![Deploy Web Server in tenant-b](EVIDENCE/task%201%20Deploy%20nginx%20in%20tenant-b.png)

### Step 1.4: Expose Tenant A Web Deployment
**What was performed**: Exposed `tenant-a`'s deployment on port 80 as a ClusterIP service (`kubectl -n tenant-a expose deployment web --port=80`).  
**Actual result shown in evidence**: Output confirms `service/web exposed`.

![Expose Service in tenant-a](EVIDENCE/task%201%20Expose%20the%20tenant-a%20web%20deployment.png)

### Step 1.5: Expose Tenant B Web Deployment
**What was performed**: Exposed `tenant-b`'s deployment on port 80 as a ClusterIP service (`kubectl -n tenant-b expose deployment web --port=80`).  
**Actual result shown in evidence**: Output confirms `service/web exposed`.

![Expose Service in tenant-b](EVIDENCE/task%201Expose%20the%20tenant-b%20web%20deployment.png)

### Step 1.6: Check Pods and Services in Tenant A
**What was performed**: Verified workload state in `tenant-a` using `kubectl get pods,svc -n tenant-a`.  
**Actual result shown in evidence**: Displays `pod/web-68d995574f-tf6hz` in `1/1 Running` state and `service/web` with ClusterIP `10.96.111.213` on port `80/TCP`.

![Verify Pods and Services in tenant-a](EVIDENCE/task%201%20Check%20pods%20and%20services%20in%20tenant-a.png)

### Step 1.7: Check Pods and Services in Tenant B
**What was performed**: Verified workload state in `tenant-b` using `kubectl get pods,svc -n tenant-b`.  
**Actual result shown in evidence**: Displays `pod/web-68d995574f-dvd9m` in `1/1 Running` state and `service/web` with ClusterIP `10.96.182.98` on port `80/TCP`.

![Verify Pods and Services in tenant-b](EVIDENCE/task%201%20Check%20pods%20and%20service%20in%20tenant-b.png)

---

## Task 2 — Observe the Default-Open Risk

By default, pods in one namespace can reach pods in another. Prove it: launch a test probe pod in `tenant-a` and connect directly to `tenant-b`'s web service.

### Step 2.1: Retrieve Tenant B Service ClusterIP
**What was performed**: Extracted the service IP of Tenant B using `kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'`.  
**Actual result shown in evidence**: Output prints ClusterIP `10.96.182.98`.

![Get Tenant B Service IP](EVIDENCE/task%202%20Get%20tenant-b's%20service%20IP.png)

### Step 2.2: Perform Cross-Tenant Network Probe (Tenant A $\rightarrow$ Tenant B)
**What was performed**: Launched an interactive curl probe inside `tenant-a` targeting `http://10.96.182.98`.  
**Actual result shown in evidence**: Output returns `HTTP 200` followed by `pod "probe" deleted`, proving unrestricted inter-namespace reachability.

![Test Cross-Tenant Reachability](EVIDENCE/task%202%20Test%20from%20tenant-a%20%E2%86%92%20tenant-b.png)

> **Security Note**: `HTTP 200` demonstrates that Kubernetes namespaces provide logical separation for API objects but do **NOT** isolate network traffic by default.

---

## Task 3 — Contain the Noisy Neighbour (Resource Quotas)

Isolation also includes resource management. Apply a `ResourceQuota` so one tenant cannot exhaust shared node CPU, memory, or pod limits.

### Step 3.1: Apply ResourceQuota to Tenant A
**What was performed**: Applied a `ResourceQuota` manifest named `tenant-a-quota` in `tenant-a` setting hard limits for pods (5), CPU requests (1), memory requests (1Gi), CPU limits (2), and memory limits (2Gi).  
**Actual result shown in evidence**: Output confirms `resourcequota/tenant-a-quota created`.

![Apply Resource Quota](EVIDENCE/task%203%20Resource%20Quota.png)

### Step 3.2: Verify Enforced Resource Quota
**What was performed**: Checked active quota limits using `kubectl get resourcequota -n tenant-a`.  
**Actual result shown in evidence**: Displays `tenant-a-quota` with `REQUEST pods: 1/5, requests.cpu: 0/1, requests.memory: 0/1Gi` and `LIMIT limits.cpu: 0/2, limits.memory: 0/2Gi`.

![Verify Resource Quota](EVIDENCE/task%203%20Verify%20the%20quota.png)

---

# Session B (Week 4) — Network & Storage Isolation

## Task 4 — Default-Deny Network Isolation

Apply a default-deny ingress policy to `tenant-b`, adhering to the segmentation principle: deny by default, permit by exception.

### Step 4.1: Apply Default-Deny Ingress NetworkPolicy to Tenant B
**What was performed**: Applied a `NetworkPolicy` named `default-deny-ingress` in `tenant-b` selecting all pods (`podSelector: {}`) with `policyTypes: [Ingress]`.  
**Actual result shown in evidence**: Output confirms `networkpolicy.networking.k8s.io/default-deny-ingress created`.

![Apply Default Deny NetworkPolicy](EVIDENCE/session%20b%20task%204.png)

### Step 4.2: Re-run Cross-Tenant Probe After Policy Application
**What was performed**: Re-launched the curl probe from `tenant-a` targeting `tenant-b`'s IP (`10.96.182.98`) and inspected probe logs via `kubectl logs -n tenant-a probe`.  
**Actual result shown in evidence**: Output returns `HTTP 000` (connection timed out and dropped by Calico CNI).

![Re-run Cross-Tenant Probe After Policy](EVIDENCE/session%20b%20task%204.2.png)

*Before vs After Comparison*:
- **Before NetworkPolicy (Task 2)**: `HTTP 200` (Cross-tenant access allowed).
- **After NetworkPolicy (Task 4)**: `HTTP 000` (Cross-tenant access blocked by Calico CNI).

---

## Task 5 — Storage & Secret Isolation

Each tenant stores a secret. Prove that a service account in `tenant-a` cannot read `tenant-b`'s secret — storage and secret isolation enforced by Kubernetes RBAC.

### Step 5.1: Create Secrets in Tenant A and Tenant B
**What was performed**: Created generic secret `data` in `tenant-a` (`value=SECRET_A`) and `data` in `tenant-b` (`value=SECRET_B`).  
**Actual result shown in evidence**: Output shows `secret/data created` in `tenant-a` and `secret/data created` in `tenant-b`.

![Create Tenant Secrets](EVIDENCE/session%20b%20task%205%20Create%20a%20secret%20in%20each%20tenant.png)

### Step 5.2: Create Scoped ServiceAccount and RBAC Rules in Tenant A
**What was performed**: Created ServiceAccount `app-a`, Role `reader` (granting `get` permission on `secrets`), and RoleBinding `rb` strictly inside `tenant-a`.  
**Actual result shown in evidence**: Output confirms creation of `serviceaccount/app-a`, `role.rbac.authorization.k8s.io/reader`, and `rolebinding.rbac.authorization.k8s.io/rb`.

![Create ServiceAccount and RBAC Rules](EVIDENCE/session%20b%20task%205%20A%20service%20account%20scoped%20to%20tenant-a%20only.png)

### Step 5.3: Verify Authorization Boundaries (`kubectl auth can-i`)
**What was performed**: Set `SA=system:serviceaccount:tenant-a:app-a` and evaluated access using `kubectl auth can-i get secrets` against `tenant-a` and `tenant-b`.  
**Actual result shown in evidence**: Impersonating `app-a` yields `yes` for `tenant-a` secrets, and `no` for `tenant-b` secrets.

![Verify RBAC Secret Access](EVIDENCE/session%20b%20task%205%20Check%20tenant-a%20permission%20and%20test%20access%20to%20tenant-b.png)

---

## Task 6 — Data Remanence & Secure Deletion

Demonstrate file system unlinking versus a secure zero-fill wipe inside a Docker container volume (`ccse-vol`).

### Step 6.1: Demonstrate Standard File Deletion (File System Unlinking)
**What was performed**: Wrote sensitive data `SENSITIVE-PATIENT-RECORD` to `/data/phi.txt`, synced storage, executed `rm /data/phi.txt`, and ran a block scan using `grep -a SENSITIVE /data/*`.  
**Actual result shown in evidence**: Scan completed with output `scan-done`, illustrating standard unlinking without overwriting disk bytes.

![Standard Deletion Scan](EVIDENCE/session%20b%20task%206%20.png)

### Step 6.2: Demonstrate Secure Wipe (Zero-Fill Overwrite via `dd`)
**What was performed**: Wrote data to `/data/phi2.txt`, executed `dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc` to overwrite data blocks with zeroes prior to deletion.  
**Actual result shown in evidence**: Terminal displays `1+0 records in`, `1+0 records out`, `1024 bytes (1.0KB) copied, 0.000122 seconds, 8.0MB/s`, followed by `wiped`.

![Secure Wipe Output](EVIDENCE/session%20b%20task%206.2.png)

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
