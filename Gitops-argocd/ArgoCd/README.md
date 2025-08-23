# 🔐 Secure GitOps Implementation with ArgoCD & OpenShift

This repository demonstrates an **end-to-end GitOps workflow** implemented on **Red Hat OpenShift** using **ArgoCD** with a strong focus on **security and compliance**.  

---

## 🚀 Project Overview
1. **Installed ArgoCD Operator** (`openshift-gitops` namespace).  
2. **Edited ArgoCD CR** (`oc edit argocd openshift-gitops -n openshift-gitops`) to:
   - Add TLS settings  
   - Mount `cluster-root-ca-bundle` ConfigMap as a trusted volume  
   - Apply RBAC for `opcadmin` role with cluster-admin permissions  
3. **Created ConfigMap** for trusted CA certificates.  
4. **Integrated GitLab repo** with ArgoCD Application for continuous deployment.  
5. Modified and pushed changes (`operator.yaml`, `console.yaml`) to GitLab repo.  
6. Validated security & compliance with **Compliance Operator**.  


---

## 📂 Repository Contents
- `argocd-rbac.yaml` → RBAC roles & bindings  
- `cluster-root-ca-bundle-configmap.yaml` → Trusted CA bundle  
- `argocd-edit.yaml` → Snippet to add ConfigMap volume in ArgoCD CR  
- `application.yaml` → Example ArgoCD Application pointing to GitLab repo  
- `operator.yaml` & `console.yaml` → Modified deployment manifests  

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

