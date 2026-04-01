Kubernetes Multi-User RBAC with Istio

Table of Contents

Overview
Project Structure
Prerequisites
Setup Instructions

Cluster-Wide Roles
Namespaces
Roles and RoleBindings
Service Accounts
Istio Setup

Usage
Troubleshooting
Best Practices
License

Overview <a name="overview"></a>
This project provides a structured approach to setting up Role-Based Access Control (RBAC) in Kubernetes for multiple users, along with integrating Istio for enhanced security and observability. Each user is restricted to specific namespaces and roles, ensuring secure and isolated access.

Project Structure <a name="project-structure"></a>
text
Copy

RBAC_MULTI_USER/
├── cluster/
│   ├── clusterroles/
│   │   ├── devops-engineer-clusterrole.yaml
│   │   └── monitoring-clusterrole.yaml
│   └── clusterrolebindings/
│       ├── devops-engineer-clusterrolebinding.yaml
│       └── monitoring-clusterrolebinding.yaml
├── dev/
│   ├── limitranges/
│   │   └── dev-limits.yaml
│   ├── resourcequotas/
│   │   └── dev-quota.yaml
│   ├── rolebindings/
│   │   ├── developer-rolebinding.yaml
│   │   ├── pipeline-rolebinding.yaml
│   │   └── viewer-rolebinding.yaml
│   ├── roles/
│   │   ├── developer-role.yaml
│   │   ├── pipeline-role.yaml
│   │   └── viewer-role.yaml
│   └── serviceaccounts/
│       ├── pipeline-sa.yaml
│       └── vijay-sa.yaml
├── prod/
│   ├── limitranges/
│   │   └── prod-limits.yaml
│   ├── resourcequotas/
│   │   └── prod-quota.yaml
│   ├── rolebindings/
│   │   ├── db-admin-rolebinding.yaml
│   │   ├── pipeline-rolebinding.yaml
│   │   └── viewer-rolebinding.yaml
│   ├── roles/
│   │   ├── db-admin-role.yaml
│   │   ├── pipeline-role.yaml
│   │   └── viewer-role.yaml
│   └── serviceaccounts/
│       └── pipeline-sa.yaml
├── istio-1.29.1/
│   └── (Istio configuration files)
├── certificates/
│   ├── devuser.crt
│   ├── devuser.csr
│   ├── devuser.key
│   ├── vijay.crt
│   ├── vijay.csr
│   ├── vijay.key
│   ├── vipin.crt
│   ├── vipin.csr
│   └── vipin.key
└── README.md




Prerequisites <a name="prerequisites"></a>

A running Kubernetes cluster (e.g., kind, minikube, or kubeadm).
kubectl installed and configured.
openssl installed on your machine.
Root or sudo access to create Linux users and manage files.
Istio installed (version 1.29.1 or compatible).

Setup Instructions <a name="setup-instructions"></a>
Cluster-Wide Roles <a name="cluster-wide-roles"></a>
Cluster-wide roles define permissions that apply across all namespaces.
Monitoring ClusterRole
yaml
Copy

# cluster/clusterroles/monitoring-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes"]
  verbs: ["get", "list", "watch"]



DevOps Engineer ClusterRole
yaml
Copy

# cluster/clusterroles/devops-engineer-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: devops-engineer
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]



Apply these roles:
bash
Copy

kubectl apply -f cluster/clusterroles/



ClusterRoleBindings
yaml
Copy

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



Apply these bindings:
bash
Copy

kubectl apply -f cluster/clusterrolebindings/




Namespaces <a name="namespaces"></a>
Define namespaces for dev and prod.
Dev Namespace
yaml
Copy

# dev/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev



Prod Namespace
yaml
Copy

# prod/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod



Apply these namespaces:
bash
Copy

kubectl apply -f dev/namespace.yaml
kubectl apply -f prod/namespace.yaml




Roles and RoleBindings <a name="roles-and-rolebindings"></a>
Dev Namespace Roles and RoleBindings
Developer Role
yaml
Copy

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



Developer RoleBinding
yaml
Copy

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



Apply these roles and bindings:
bash
Copy

kubectl apply -f dev/roles/
kubectl apply -f dev/rolebindings/



Prod Namespace Roles and RoleBindings
DB Admin Role
yaml
Copy

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



DB Admin RoleBinding
yaml
Copy

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



Apply these roles and bindings:
bash
Copy

kubectl apply -f prod/roles/
kubectl apply -f prod/rolebindings/




Service Accounts <a name="service-accounts"></a>
Service accounts are used for automation and CI/CD pipelines.
Pipeline Service Account in Dev
yaml
Copy

# dev/serviceaccounts/pipeline-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline
  namespace: dev



Pipeline Service Account in Prod
yaml
Copy

# prod/serviceaccounts/pipeline-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline
  namespace: prod



Apply these service accounts:
bash
Copy

kubectl apply -f dev/serviceaccounts/
kubectl apply -f prod/serviceaccounts/




Resource Quotas and Limit Ranges <a name="resource-quotas-and-limit-ranges"></a>
Dev Resource Quota
yaml
Copy

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



Dev Limit Range
yaml
Copy

# dev/limitranges/dev-limits.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - default:
      cpu: 500m
      memory: 1Gi
    defaultRequest:
      cpu: 200m
      memory: 512Mi
    type: Container



Apply these quotas and limits:
bash
Copy

kubectl apply -f dev/resourcequotas/
kubectl apply -f dev/limitranges/
kubectl apply -f prod/resourcequotas/
kubectl apply -f prod/limitranges/




Istio Setup <a name="istio-setup"></a>
Istio provides service mesh capabilities for observability, security, and traffic management.
Install Istio
bash
Copy

curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y



Enable Istio Sidecar Injection
bash
Copy

kubectl label namespace dev istio-injection=enabled
kubectl label namespace prod istio-injection=enabled




Usage <a name="usage"></a>


Access Cluster as a User:

Use the respective kubeconfig file for each user.
Example for vipin:
bash
Copy

export KUBECONFIG=/path/to/vipin-kubeconfig
kubectl get pods -n dev






Deploy Applications:

Deploy your applications in the respective namespaces.
Example:
bash
Copy

kubectl apply -f your-app.yaml -n dev






Monitor with Istio:

Use Istio tools like Kiali for observability.
bash
Copy

istioctl dashboard kiali





Troubleshooting <a name="troubleshooting"></a>
Issue 1: Permission Denied

Cause: Incorrect RBAC rules or missing roles/rolebindings.
Solution: Verify roles and rolebindings are correctly applied.
bash
Copy

kubectl get roles,rolebindings -n dev
kubectl describe rolebinding developer-binding -n dev




Issue 2: Unable to Access Namespace

Cause: Incorrect context or kubeconfig file.
Solution: Verify the context and kubeconfig file.
bash
Copy

kubectl config get-contexts
kubectl config view




Issue 3: Istio Sidecar Not Injected

Cause: Namespace not labeled for Istio sidecar injection.
Solution: Label the namespace for Istio sidecar injection.
bash
Copy

kubectl label namespace dev istio-injection=enabled




Issue 4: Certificate Errors

Cause: Missing or incorrect ca.crt file.
Solution: Ensure ca.crt is correctly referenced in the kubeconfig file.
bash
Copy

grep 'certificate-authority' ~/.kube/config





Best Practices <a name="best-practices"></a>

Least Privilege: Grant the minimum permissions necessary for each user or service account.
Regular Audits: Regularly review and audit roles, rolebindings, and permissions.
Use Service Accounts for Automation: Avoid using user credentials for automation; use service accounts instead.
Monitor and Log: Use Istio and Kubernetes audit logs to monitor and log access and activities.
Rotate Certificates: Regularly rotate certificates and keys to maintain security.

License <a name="license"></a>
This project is licensed under the MIT License.
