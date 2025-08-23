# 🔐 Secure GitOps Implementation with ArgoCD & OpenShift

This repository demonstrates an **end-to-end GitOps workflow** implemented on **Red Hat OpenShift** using **ArgoCD** with a strong focus on **security and compliance**.  

---

## 🚀 Project Overview
- Installed and configured the **OpenShift GitOps (ArgoCD) Operator**
- Enabled **TLS encryption** and integrated `cluster-root-ca-bundle` ConfigMap  
- Applied **RBAC policies** (added `opcadmin` with `cluster-admin`)  
- Configured ArgoCD to mount ConfigMap as a **trusted volume**  
- Integrated with **GitLab repository** for GitOps workflow  
- Automated deployment of manifests (`operator.yaml`, `console.yaml`) to OpenShift cluster  
- Enforced **policy compliance** using the Compliance Operator  

---

## 📂 Repository Contents
- `manifests/operator.yaml` → GitOps operator setup  
- `manifests/console.yaml` → OpenShift console customization  
- `config/cluster-root-ca-bundle.yaml` → ConfigMap with trusted root certificates  
- `policies/rbac-policy.yaml` → Role/RoleBinding for secure access control  

---

## 🛠️ Tools & Skills
- ArgoCD  
- GitOps  
- OpenShift  
- TLS & ConfigMap Integration  
- RBAC (Role-Based Access Control)  
- GitLab CI/CD  
- Kubernetes Operators  
- Compliance Operator  

---

## 📸 Screenshots
(Add your OpenShift & ArgoCD screenshots here for better presentation)

---

## 🔗 How to Use
1. Clone the repo:
   ```bash
   git clone https://github.com/shweta-7403/DO380-repo/Gitops-argocd.git
   cd Gitops-argocd

