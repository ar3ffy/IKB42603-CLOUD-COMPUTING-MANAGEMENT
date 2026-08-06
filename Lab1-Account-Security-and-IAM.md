# Lab Report: Cloud Account Security, Identity & Access Management (IAM) & Kubernetes RBAC

**Course**: IKB42603 Cloud Computing Security Essentials  
**Institution**: UniKL MIIT — Prof. Dr. Shahrulniza Musa  
**Topic**: Identity Governance and Least Privilege — LocalStack IAM & Kubernetes RBAC  
**Student/User Identity**: Rafissah (`CloudAdmin_Rafissah` / `Analyst_Rafissah`)  

---

## Course & Assessment Mapping

- **Course Learning Outcome (CLO)**: CLO2 — Construct secure cloud operations that safeguard data integrity
- **Lecture Topics**: Weeks 1–2 (Fundamentals, Security Architecture) · Weeks 5 & 7 (Access Control, Identity)
- **Value / Skill Clusters**: VBE3 (Integrity) · SC8 (Integrated Problem-Solving)
- **Assessment**: Lab Report (Screenshots + CLI Output + Short Answers)

---

## Executive Summary

This lab report documents the step-by-step implementation of cloud account security, Identity and Access Management (IAM) governance using LocalStack, and fine-grained Role-Based Access Control (RBAC) enforcement using Kubernetes (`kind`). The objective is to replace dangerous root account reliance with scoped IAM entities (users, groups, policies, access keys) and validate strict authorization boundaries across Kubernetes namespaces.

---

# Session A (Week 1) — Cloud Identity with LocalStack

## One-Time Environment Setup

To isolate cloud testing without incurring costs or risking live AWS infrastructure, LocalStack was deployed locally via Docker.

```bash
# 1. Confirm Docker installation
docker --version

# 2. Start LocalStack container
docker run -d --name localstack -p 4566:4566 localstack/localstack

# 3. Confirm health status
curl http://localhost:4566/_localstack/health

# 4. Configure AWS CLI dummy credentials and endpoint alias
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

# Define endpoint helper variable
EP='--endpoint-url=http://localhost:4566'

# Verify operating identity
aws $EP sts get-caller-identity
```

---

## Task 1 — Map the Cloud Identity Landscape

The core building blocks of cloud identity and governance are mapped below:

| Concept | AWS Term | Purpose |
| :--- | :--- | :--- |
| **All-powerful owner** | Root user | The initial identity created with complete, unrestricted administrative permissions over all account resources and billing. Should be secured and avoided for daily tasks. |
| **Human/app identity** | IAM User | An identity created within AWS that represents a specific person or application, configured with specific long-term security credentials (passwords, access keys). |
| **Permission bundle** | IAM Policy | A formal JSON document that defines explicit permissions (Allow/Deny statements specifying actions, resources, and optional conditions). |
| **Collection of users** | IAM Group | A logical container for IAM users that allows administrators to attach policies once and manage permissions collectively across multiple identities. |
| **Temporary identity** | IAM Role | An identity with specific permissions that can be temporarily assumed by human users, applications, or AWS services using short-lived STS credentials. |

---

## Task 2 — Create a Least-Privilege Admin (Stop Using Root)

Operating as the root account presents severe security risks. A dedicated administrative group and user were created so daily administrative tasks do not rely on root credentials.

### Step 2.1: Create Admin Group & Attach AdministratorAccess Policy
```bash
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```
![Create Admin Group](LAB%201/Create%20admin%20group.png)  
![Attach Administrator Policy](LAB%201/Attach%20the%20administrator%20policy%20to%20the%20group.png)

### Step 2.2: Create Personal Admin User
```bash
aws $EP iam create-user --user-name CloudAdmin_Rafissah
```
![Create Personal Admin User](LAB%201/Step%204.%20create%20personal%20admin%20user.png)

### Step 2.3: Add User to Admins Group
```bash
aws $EP iam add-user-to-group --group-name Admins \
  --user-name CloudAdmin_Rafissah
```
![Add User to Admins Group](LAB%201/Step%205.%20add%20your%20user%20to%20the%20admins%20group.png)

### Step 2.4: Verify Group Membership
```bash
aws $EP iam get-group --group-name Admins
```
![Verify Group Membership](LAB%201/Step%206.%20verify%20the%20group%20membership.png)

**Security Tip**: Attaching policies to groups rather than individual users ensures manageable, consistent, and scalable privilege auditing across the organization.

---

## Task 3 — Enforce Least Privilege with a Scoped Policy

To prevent privilege creep and adhere to the principle of least privilege, a dedicated read-only analyst identity was provisioned.

### Step 3.1: Create Read-Only Analyst User
```bash
aws $EP iam create-user --user-name Analyst_Rafissah
```
![Create Analyst User](LAB%201/Step%207.%20create%20a%20read%20only%20analyst%20user%20(task%203).png)

### Step 3.2: Attach Scoped Read-Only Policy (AmazonS3ReadOnlyAccess)
```bash
aws $EP iam attach-user-policy --user-name Analyst_Rafissah \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```
![Attach Read Only Policy](LAB%201/Step%208.%20attach%20a%20read%20only%20policy%20to%20the%20analyst%20user.png)

### Step 3.3: Verify Attached Policies
```bash
aws $EP iam list-attached-user-policies --user-name Analyst_Rafissah
```
![Verify Attached Policy](LAB%201/Step%209.%20verity%20the%20attached%20policy.png)

### Blast-Radius Reduction Explanation
> **Question**: If the Analyst account were stolen, why is the damage limited compared to a stolen admin account? Connect your answer to blast-radius reduction.
>
> **Answer**: 
> "Blast radius" refers to the maximum potential extent of damage that can occur when an identity or system component is compromised. Because `Analyst_Rafissah` is governed by `AmazonS3ReadOnlyAccess`, an attacker possessing these credentials can only inspect S3 storage objects. The attacker cannot delete S3 buckets, tamper with data, spin up costly EC2 instances, alter security groups, or escalate privileges. In contrast, compromising an administrative account grants full control (`AdministratorAccess`), enabling total infrastructure destruction or data exfiltration. Scoping privileges drastically reduces the blast radius to read-only S3 operations.

---

## Task 4 — Credential Hygiene & Access Keys

Programmatic access relies on access key pairs. Managing lifecycle and rotating active credentials prevents security leaks.

### Step 4.1: Create Access Key for Analyst
```bash
aws $EP iam create-access-key --user-name Analyst_Rafissah
```
![Create Access Key](LAB%201/Step%2010.%20Create%20an%20access%20key%20for%20the%20analyst.png)

### Step 4.2: List Access Keys
```bash
aws $EP iam list-access-keys --user-name Analyst_Rafissah
```
![List Access Keys](LAB%201/Step%2011.%20%20list%20the%20access%20keys.png)

### Step 4.3: Deactivate Access Key (Key Rotation)
```bash
aws $EP iam update-access-key --user-name Analyst_Rafissah \
  --access-key-id LKIAQAAAAAAAPX56RNHV --status Inactive
```
![Deactivate Access Key](LAB%201/Step%2012.%20Deactive%20the%20access%20key%20(rotation%20key).png)

---

# Session B (Week 2) — Enforced Access Control with Kubernetes RBAC

## Setup — Create a Local Kubernetes Cluster

While LocalStack demonstrates IAM policy creation, Kubernetes RBAC actively enforces real-time API request authorization.

```bash
# Create local Kind cluster
kind create cluster --name ccse-lab1

# Verify cluster status
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```
![Create Kind Cluster](LAB%201/Session%20B.%20Step%201.%20create%20the%20kubernetes%20cluster.png)  
![Verify Kubernetes Cluster](LAB%201/Session%20B.%20Step%202.%20verify%20the%20kubernetes%20cluster.png)

---

## Task 5 — Separate Environments with Namespaces

Namespaces create logical isolation boundaries within a Kubernetes cluster to partition dev and production workloads.

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```
![Create Namespaces](LAB%201/Session%20B.%20Step%203.%20create%20the%20namespaces.png)

---

## Task 6 — Define a Role and Bind It (Least Privilege)

RBAC pairs permission definitions (`Role`) with assignment objects (`RoleBinding`).

### Step 6.1: Create Developer ServiceAccount
```bash
kubectl create serviceaccount dev-user -n dev
```
![Create ServiceAccount](LAB%201/Session%20B.%20Step%204.%20create%20a%20service%20account.png)

### Step 6.2: Create Scoped Pod-Reader Role
```bash
kubectl create role pod-reader -n dev \
  --verb=get,list,watch --resource=pods
```
![Create Pod-Reader Role](LAB%201/Session%20B.%20Step%205.%20create%20a%20role%20(read%20pods%20only).png)

### Step 6.3: Bind Role to ServiceAccount
```bash
kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader --serviceaccount=dev:dev-user
```
![Create RoleBinding](LAB%201/Session%20B.%20Step%206.%20bind%20the%20role%20to%20the%20service%20account.png)

---

## Task 7 — Test That Access Control Works

Using `kubectl auth can-i`, authorization limits were tested against `system:serviceaccount:dev:dev-user`.

```bash
# Define SA variable
SA=system:serviceaccount:dev:dev-user
echo $SA
```
![Define SA Variable](LAB%201/Session%20B.%20Step%207.%20define%20the%20service%20account%20variable.png)

### Test 1: Reading pods in dev (Allowed)
```bash
kubectl auth can-i list pods -n dev --as=$SA
# Result: yes
```
![Test 1 YES](LAB%201/Session%20B.%20Step%208.%20test%201%20%20(yes).png)

### Test 2: Deleting pods in dev (Denied)
```bash
kubectl auth can-i delete pods -n dev --as=$SA
# Result: no
```
![Test 2 NO](LAB%201/Session%20B.%20Step%209.%20test%202%20(no).png)

### Test 3: Reading pods in prod (Denied)
```bash
kubectl auth can-i list pods -n prod --as=$SA
# Result: no
```
![Test 3 NO](LAB%201/Session%20B.%20Step%2010.%20Test%203%20%20(no).png)

---

### Authentication vs. Authorization Analysis

> **Question**: Relate the three `can-i` results to authentication versus authorization: which step is the service account passing, and which step is blocking the delete and the prod access?
>
> **Analysis**:
> 1. **Authentication (Identity Verification)**: Passed in all three commands. Kubernetes successfully authenticated the entity `system:serviceaccount:dev:dev-user` via `--as=$SA`.
> 2. **Authorization (Permission Check)**:
>    - **Test 1 (`list pods -n dev`)**: Authorization succeeds (`yes`) because the `dev-user-binding` grants `pod-reader` role (`get`, `list`, `watch` on `pods`) specifically inside the `dev` namespace.
>    - **Test 2 (`delete pods -n dev`)**: Authorization fails (`no`) because the `delete` verb is absent from the `pod-reader` role definition.
>    - **Test 3 (`list pods -n prod`)**: Authorization fails (`no`) because `Role` and `RoleBinding` are namespace-scoped to `dev`. No RBAC rule authorizes this ServiceAccount in the `prod` namespace.

---

# Deliverables & Assessment

## 1. Screenshots Summary Checklist

- [x] Output of `sts get-caller-identity` / environment verification
- [x] Output of `get-group Admins` showing `CloudAdmin_Rafissah` as a member
- [x] Output of `list-attached-user-policies` showing `AmazonS3ReadOnlyAccess` for `Analyst_Rafissah`
- [x] The three `kubectl auth can-i` test results (`yes` / `no` / `no`)

---

## 2. Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?
> **Answer**:  
> Attaching policies to IAM Groups rather than individual users provides three critical operational and security advantages:
> 1. **Scalability and Manageability**: Updating permissions for an entire team requires modifying a single group policy, automatically applying changes to all group members rather than requiring individual edits per user.
> 2. **Auditability and Consistency**: Ensures all individuals performing a role maintain identical, predictable permissions, reducing configuration drift and accidental over-privileging.
> 3. **Simplified Lifecycles**: When employees join, switch teams, or leave, administrators simply add or remove them from groups without manually rebuilding individual policy attachments.

### Q2. What is the difference between an IAM User and an IAM Role?
> **Answer**:  
> - **IAM User**: A permanent identity created for a specific person or service. It has long-term credentials (passwords or static access key pairs) associated directly with it.
> - **IAM Role**: An identity with permissions that is **not** tied to a single person or permanent static credentials. Instead, it is dynamically assumed by authorized users, applications, or cloud services (e.g., EC2, Lambda) to obtain temporary security tokens via AWS Security Token Service (STS).

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.
> **Answer**:  
> - **Least Privilege**: Granting an identity only the minimum set of permissions necessary to execute its specific tasks. In Task 3, `Analyst_Rafissah` was assigned `AmazonS3ReadOnlyAccess` rather than administrator or write access.
> - **Blast Radius Reduction**: If `Analyst_Rafissah`'s access credentials are leaked or stolen, the attacker is strictly limited to inspecting S3 data. They cannot overwrite data, delete resources, alter IAM permissions, or deploy unauthorized workloads, effectively containing the potential damage of a credential breach.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?
> **Answer**:  
> - **Role**: Defines a collection of permission rules (Verbs: `get`, `list`, `watch` on Resources: `pods`, `services`) within a specific namespace. It specifies **WHAT** actions are allowed, but contains no references to users or identities.
> - **RoleBinding**: Connects (binds) a `Role` to one or more subjects (Users, Groups, or `ServiceAccounts`). It specifies **WHO** receives the permissions granted by the `Role` within that namespace.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?
> **Answer**:  
> - **Cause of Failure**: The `pod-reader` Role and `dev-user-binding` RoleBinding were configured exclusively within the `dev` namespace (`-n dev`). In Kubernetes, Roles and RoleBindings are namespace-scoped. No RoleBinding exists in `prod` for `dev-user`.
> - **Security Principles**: 
>   1. **Default Deny / Implicit Deny**: In RBAC models, any action not explicitly allowed by a binding is denied by default.
>   2. **Compartmentalization & Least Privilege**: Namespace boundaries isolate environments so dev access cannot leak into production.

---

## 3. Verification Command Output

Output of `kubectl get rolebinding dev-user-binding -n dev -o yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-08-05T17:22:15Z"
  name: dev-user-binding
  namespace: dev
  resourceVersion: "1247"
  uid: 731e4d9b-0471-4887-bcf1-28ecd7cd0959
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```
![Last Verification](LAB%201/Session%20B.%20Last%20verification.png)

---

## Security Best-Practices Checklist

- [x] **Root user is not used for daily tasks** (a dedicated admin identity `CloudAdmin_Rafissah` exists).
- [x] **Permissions are granted via groups/roles**, not directly to individual users.
- [x] **At least one least-privilege (read-only) identity** (`Analyst_Rafissah`) was created and tested.
- [x] **Access keys were listed and a rotation (deactivate)** was demonstrated.
- [x] **Kubernetes RBAC blocks an unauthorized action** (delete verb / cross-namespace access to `prod`).

---

## Cleanup & Teardown

```bash
# Remove local Kubernetes cluster
kind delete cluster --name ccse-lab1

# Stop and remove LocalStack container
docker stop localstack && docker rm localstack
```
