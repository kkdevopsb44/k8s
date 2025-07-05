# Kubernetes RBAC — Hands‑On Guide

Role‑Based Access Control (RBAC) in Kubernetes is a powerful method for regulating access to the Kubernetes API based on the roles of individual users or groups.

---

## Key Components

| Component | Scope | Purpose |
|-----------|-------|---------|
| **Role** | Namespace‑scoped | Grants permissions (e.g., `pods`, `deployments`) **within a single namespace**. |
| **ClusterRole** | Cluster‑wide | Same as Role, but applies across **all namespaces** and to cluster‑scoped resources. |
| **RoleBinding** | Namespace‑scoped | **Binds** a Role to a user/group/service‑account **within one namespace**. |
| **ClusterRoleBinding** | Cluster‑wide | **Binds** a ClusterRole to a user/group/service‑account **across the entire cluster**. |

---

# Practical Walk‑Through

> Tested on **Windows 11** workstation + **AWS EKS** cluster.

---

## Step 1 — Install `kubectl` (Windows)

Official docs → <https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/>

```powershell
# v1.31.0
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"

# or v1.32.0
curl.exe -LO "https://dl.k8s.io/release/v1.32.0/bin/windows/amd64/kubectl.exe"
```

Add the folder that contains **`kubectl.exe`**  
(e.g., `C:\Users\gpras\OneDrive\Desktop\rbac-practise`)  
to **System Properties → Environment Variables → Path**.

---

## Step 2 — Install AWS CLI

Docs → <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>

```powershell
# Download & run the 64‑bit MSI
https://awscli.amazonaws.com/AWSCLIV2.msi
```

---

## Step 3 — Create an IAM User + Keys

1. Sign in: <https://640168417733.signin.aws.amazon.com/console>  
2. User name: **`kkfunda`**  
3. Password: **`Vision@2030`** (example)  
4. Generate **Access Key ID (AK)** and **Secret Access Key (SK)**.

```
AK: <your-access-key>
SK: <your-secret-key>
```

---

## Step 4 — Configure AWS CLI + Connect to EKS

```bash
aws configure
# Access Key ID:  <AK>
# Secret Access Key: <SK>
# Default region name: ap-south-1
# Default output format: table
```

Verify access:

```bash
aws sts get-caller-identity
aws eks list-clusters
aws eks update-kubeconfig --name EKS-Demo123 --region ap-southeast-1
```

---

## Step 5 — (If Needed) Attach Read‑Only EKS Policy

```bash
aws sts get-caller-identity
aws iam attach-user-policy   --user-name kkfunda   --policy-arn arn:aws:iam::aws:policy/AmazonEKSReadOnlyAccess
```

---

## Step 6 — Map the IAM User in `aws-auth` ConfigMap

From a Linux bastion (or wherever you have cluster‑admin `kubectl`):

```bash
kubectl edit -n kube-system configmap/aws-auth
```

Add the user:

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::418295704475:role/EKS-worker-role-class
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::418295704475:user/kkfunda
      username: kkfunda
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```

Save & exit. The IAM user **`kkfunda`** can now authenticate to the cluster.

---

# RBAC Objects

Below are two sets of manifests:

* **Namespace‑scoped read‑only access** (`Role` + `RoleBinding`)
* **Cluster‑admin access** (`ClusterRole` + `ClusterRoleBinding`)

---

## 1 — Role & RoleBinding (Namespace = `test-ns`)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-ns
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: test-ns
subjects:
- kind: User
  name: kkfunda
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f pod-reader-rb.yaml
```

---

## 2 — ClusterRole & ClusterRoleBinding (Full Admin)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fullaccesscr
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fullaccess
subjects:
- kind: User
  name: kkfunda
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: fullaccesscr
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f cluster-admin.yaml
```

---

# Verification Commands

```bash
# Check namespace‑scoped permissions
kubectl auth can-i list pods --as kkfunda -n test-ns
# → yes

# Check cluster‑wide permissions
kubectl auth can-i list nodes --as kkfunda
# → yes  (only if ClusterRoleBinding is applied)
```

If both return **`yes`**, RBAC is configured correctly.

---

## Troubleshooting Checklist

| Symptom | What to Check |
|---------|---------------|
| `Forbidden` errors | ‑ Correct Role/ClusterRole?  <br>‑ RoleBinding/ClusterRoleBinding subject name matches IAM user in `aws-auth`. |
| `iam authenticator: access denied` | ‑ User/role missing from `aws-auth` ConfigMap. |
| Uses but cannot list cluster resources | ‑ Has only namespace RoleBinding; need ClusterRoleBinding. |

---

**Happy clustering!** 🐳⚙️
