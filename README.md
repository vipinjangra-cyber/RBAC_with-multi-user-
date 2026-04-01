# 🔐 Kubernetes Multi-User RBAC with Istio

> A production-ready setup for Role-Based Access Control (RBAC) in Kubernetes with Istio service mesh integration — enabling secure, isolated, and observable multi-user environments.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Cluster-Wide Roles](#1-cluster-wide-roles)
  - [2. Namespaces](#2-namespaces)
  - [3. Roles & RoleBindings](#3-roles--rolebindings)
  - [4. Service Accounts](#4-service-accounts)
  - [5. Resource Quotas & Limit Ranges](#5-resource-quotas--limit-ranges)
  - [6. Istio Setup](#6-istio-setup)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [License](#license)

---

## 🧭 Overview

This project provides a **structured, scalable approach** to managing multi-user access in Kubernetes using RBAC, combined with **Istio** for advanced service mesh capabilities including mTLS, traffic management, and observability.

### Key Features

| Feature | Description |
|---|---|
| 🔒 **RBAC** | Fine-grained access control per user and namespace |
| 🌐 **Istio Integration** | mTLS, traffic policies, and observability via Kiali |
| 🏗️ **Multi-Namespace** | Isolated `dev` and `prod` environments |
| ⚙️ **Service Accounts** | Dedicated accounts for CI/CD pipelines |
| 📊 **Resource Governance** | Quotas and limit ranges per namespace |
| 🔑 **Certificate-Based Auth** | X.509 certificates for user authentication |

---

## 📁 Project Structure

```
RBAC_MULTI_USER/
├── cluster/
│   ├── clusterroles/
│   │   ├── devops-engineer-clusterrole.yaml
│   │   └── monitoring-clusterrole.yaml
│   └── clusterrolebindings/
│       ├── devops-engineer-clusterrolebinding.yaml
│       └── monitoring-clusterrolebinding.yaml
│
├── dev/
│   ├── limitranges/
│   │   └── dev-limits.yaml
│   ├── resourcequotas/
│   │   └── dev-quota.yaml
│   ├── roles/
│   │   ├── developer-role.yaml
│   │   ├── pipeline-role.yaml
│   │   └── viewer-role.yaml
│   ├── rolebindings/
│   │   ├── developer-rolebinding.yaml
│   │   ├── pipeline-rolebinding.yaml
│   │   └── viewer-rolebinding.yaml
│   └── serviceaccounts/
│       ├── pipeline-sa.yaml
│       └── vijay-sa.yaml
│
├── prod/
│   ├── limitranges/
│   │   └── prod-limits.yaml
│   ├── resourcequotas/
│   │   └── prod-quota.yaml
│   ├── roles/
│   │   ├── db-admin-role.yaml
│   │   ├── pipeline-role.yaml
│   │   └── viewer-role.yaml
│   ├── rolebindings/
│   │   ├── db-admin-rolebinding.yaml
│   │   ├── pipeline-rolebinding.yaml
│   │   └── viewer-rolebinding.yaml
│   └── serviceaccounts/
│       └── pipeline-sa.yaml
│
├── istio-1.29.1/
│   └── (Istio configuration files)
│
├── certificates/
│   ├── devuser.{crt,csr,key}
│   ├── vijay.{crt,csr,key}
│   └── vipin.{crt,csr,key}
│
└── README.md
```

### User → Role Mapping

| User | Namespace | Role | Access Level |
|---|---|---|---|
| `vipin` | `dev` | `developer` | Full CRUD on pods, services, deployments |
| `devuser` | `prod` | `db-admin` | Full CRUD including secrets |
| `vijay` | `dev` | Custom SA | Via service account |
| Monitoring Group | Cluster-wide | `monitoring` | Read-only on pods, services, nodes |
| DevOps Engineer | Cluster-wide | `devops-engineer` | Full cluster access |

---

## ✅ Prerequisites

Ensure the following are available before proceeding:

- [ ] A running Kubernetes cluster (`kind`, `minikube`, or `kubeadm`)
- [ ] `kubectl` installed and configured
- [ ] `openssl` installed for certificate generation
- [ ] Root/sudo access for Linux user management
- [ ] Istio v1.29.1 (or compatible) installed

---

## 🛠️ Setup Instructions

### 1. Cluster-Wide Roles

Cluster roles apply across **all namespaces** and are used for platform-level personas.

#### Monitoring ClusterRole

```yaml
# cluster/clusterroles/monitoring-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes"]
  verbs: ["get", "list", "watch"]
```

#### DevOps Engineer ClusterRole

```yaml
# cluster/clusterroles/devops-engineer-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: devops-engineer
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
```

#### Monitoring ClusterRoleBinding

```yaml
# cluster/clusterrolebindings/monitoring-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-binding
subjects:
- kind: Group
  name: monitoring-group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: monitoring
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Apply all cluster-level resources
kubectl apply -f cluster/clusterroles/
kubectl apply -f cluster/clusterrolebindings/
```

---

### 2. Namespaces

```yaml
# dev/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```yaml
# prod/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

```bash
kubectl apply -f dev/namespace.yaml
kubectl apply -f prod/namespace.yaml
```

---

### 3. Roles & RoleBindings

#### Dev Namespace

**Developer Role** — grants CRUD access to core workload resources:

```yaml
# dev/roles/developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "deployments"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

**Developer RoleBinding** — binds user `vipin` to the developer role:

```yaml
# dev/rolebindings/developer-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
subjects:
- kind: User
  name: vipin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dev/roles/
kubectl apply -f dev/rolebindings/
```

#### Prod Namespace

**DB Admin Role** — includes access to `secrets` for database credential management:

```yaml
# prod/roles/db-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: db-admin
rules:
- apiGroups: [""]
  resources: ["pods", "services", "deployments", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

**DB Admin RoleBinding** — binds user `devuser` to the db-admin role:

```yaml
# prod/rolebindings/db-admin-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: db-admin-binding
  namespace: prod
subjects:
- kind: User
  name: devuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: db-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f prod/roles/
kubectl apply -f prod/rolebindings/
```

---

### 4. Service Accounts

Service accounts are used for **CI/CD pipelines and automation** — avoiding the use of personal user credentials.

```yaml
# dev/serviceaccounts/pipeline-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline
  namespace: dev
```

```yaml
# prod/serviceaccounts/pipeline-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline
  namespace: prod
```

```bash
kubectl apply -f dev/serviceaccounts/
kubectl apply -f prod/serviceaccounts/
```

---

### 5. Resource Quotas & Limit Ranges

Resource quotas prevent any single namespace from consuming excessive cluster resources.

#### Dev Namespace

```yaml
# dev/resourcequotas/dev-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

```yaml
# dev/limitranges/dev-limits.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 1Gi
    defaultRequest:
      cpu: 200m
      memory: 512Mi
```

```bash
kubectl apply -f dev/resourcequotas/
kubectl apply -f dev/limitranges/
kubectl apply -f prod/resourcequotas/
kubectl apply -f prod/limitranges/
```

---

### 6. Istio Setup

Istio adds a **service mesh layer** providing mutual TLS, traffic policies, and rich observability.

#### Install Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.29.1
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```

#### Enable Sidecar Injection

```bash
kubectl label namespace dev istio-injection=enabled
kubectl label namespace prod istio-injection=enabled
```

> **Note:** Pods must be restarted after labeling for the Istio sidecar to be injected into existing workloads.

```bash
# Restart existing deployments to pick up sidecar injection
kubectl rollout restart deployment -n dev
kubectl rollout restart deployment -n prod
```

---

## 🚀 Usage

### Switching User Contexts

Each user has their own kubeconfig file backed by an X.509 certificate. Set the context before running any commands:

```bash
# Switch to vipin's context (dev namespace)
export KUBECONFIG=/path/to/vipin-kubeconfig
kubectl get pods -n dev

# Switch to devuser's context (prod namespace)
export KUBECONFIG=/path/to/devuser-kubeconfig
kubectl get pods -n prod
```

### Deploying Applications

```bash
# Deploy to dev
kubectl apply -f your-app.yaml -n dev

# Deploy to prod
kubectl apply -f your-app.yaml -n prod
```

### Observability with Istio

```bash
# Open Kiali dashboard (service mesh topology)
istioctl dashboard kiali

# Open Grafana (metrics)
istioctl dashboard grafana

# Open Jaeger (distributed tracing)
istioctl dashboard jaeger
```

---

## 🔧 Troubleshooting

### ❌ Permission Denied

**Cause:** Incorrect RBAC rules or missing roles/rolebindings.

```bash
# Inspect roles and bindings in the dev namespace
kubectl get roles,rolebindings -n dev

# Describe a specific binding
kubectl describe rolebinding developer-binding -n dev

# Test permissions for a specific user
kubectl auth can-i get pods -n dev --as=vipin
```

---

### ❌ Unable to Access Namespace

**Cause:** Wrong Kubernetes context or kubeconfig file.

```bash
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config view
```

---

### ❌ Istio Sidecar Not Injected

**Cause:** Namespace missing the Istio injection label.

```bash
# Verify labels
kubectl get namespace dev --show-labels

# Re-apply the label if missing
kubectl label namespace dev istio-injection=enabled --overwrite

# Restart pods to trigger injection
kubectl rollout restart deployment -n dev
```

---

### ❌ Certificate Errors

**Cause:** Missing or incorrect `ca.crt` in the kubeconfig file.

```bash
# Verify the CA certificate reference
grep 'certificate-authority' ~/.kube/config

# Check certificate validity
openssl x509 -in certificates/vipin.crt -text -noout | grep -E "Subject|Validity"
```

---

## 💡 Best Practices

| Practice | Description |
|---|---|
| **Least Privilege** | Grant only the minimum permissions required — avoid wildcard verbs in production roles |
| **Regular Audits** | Periodically review roles, bindings, and service account permissions |
| **Use Service Accounts** | Never use personal user credentials for automated workloads or pipelines |
| **Rotate Certificates** | Rotate X.509 certificates regularly and keep private keys secure |
| **Enable Audit Logging** | Use Kubernetes audit logs alongside Istio telemetry for full traceability |
| **Namespace Isolation** | Use `NetworkPolicy` in addition to RBAC for stronger workload isolation |
| **Avoid ClusterAdmin** | Restrict `cluster-admin` to break-glass scenarios only |

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

---

<p align="center">
  Made with ❤️ for secure Kubernetes multi-tenancy
</p>