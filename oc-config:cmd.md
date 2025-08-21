# 🔧 OpenShift `oc config` Commands — Complete Guide  

The `oc config` command in OpenShift helps you manage your **kubeconfig file**.  
This file stores information about **clusters, users, and contexts**, which define how you connect and interact with OpenShift clusters.  

This guide covers all important `oc config` commands with **use cases** and **examples**.  

---

## 📌 1. Context Management  

### ▶️ `oc config current-context`  
**Description:** Displays the current context in use.  
**Use case:** Verify which cluster/project you are connected to before running commands.  

```bash
oc config current-context
# Output: mycluster-admin/dev-project
```

---

### ▶️ `oc config use-context <context-name>`  
**Description:** Switch to a different context.  
**Use case:** Move between clusters/namespaces easily.  

```bash
oc config use-context mycluster-admin/prod-project
```

---

### ▶️ `oc config get-contexts`  
**Description:** List all available contexts.  
**Use case:** See all cluster+user+namespace combinations defined.  

```bash
oc config get-contexts
```

---

### ▶️ `oc config rename-context <old> <new>`  
**Description:** Rename an existing context.  
**Use case:** Clean up confusing context names.  

```bash
oc config rename-context mycluster-admin/dev mycluster-dev
```

---

### ▶️ `oc config set-context <context-name>`  
**Description:** Create or modify a context (cluster + user + namespace).  
**Use case:** Define reusable contexts for quick switching.  

```bash
oc config set-context dev-context   --cluster=mycluster   --namespace=dev-project   --user=dev-user
```

---

## 📌 2. Cluster Management  

### ▶️ `oc config get-clusters`  
**Description:** List all clusters defined in kubeconfig.  
**Use case:** Check which clusters are stored.  

```bash
oc config get-clusters
```

---

### ▶️ `oc config set-cluster <name>`  
**Description:** Add or modify a cluster entry.  
**Use case:** Configure a new cluster endpoint.  

```bash
oc config set-cluster mycluster   --server=https://api.openshift.example.com:6443   --certificate-authority=/etc/kubernetes/ca.crt
```

---

### ▶️ `oc config delete-cluster <name>`  
**Description:** Remove a cluster entry.  
**Use case:** Clean up old or unused clusters.  

```bash
oc config delete-cluster oldcluster
```

---

## 📌 3. User Management  

### ▶️ `oc config get-users`  
**Description:** List all users defined in kubeconfig.  
**Use case:** Check which credentials are available.  

```bash
oc config get-users
```

---

### ▶️ `oc config set-credentials <name>`  
**Description:** Add or modify user credentials.  
**Use case:** Define a user with token or client certs.  

```bash
oc config set-credentials dev-user   --token=sha256~abc123xyz
```

---

### ▶️ `oc config delete-user <name>`  
**Description:** Delete a user entry.  
**Use case:** Remove stale or invalid credentials.  

```bash
oc config delete-user old-user
```

---

## 📌 4. Configuration Values  

### ▶️ `oc config set`  
**Description:** Set an individual value in kubeconfig.  
**Use case:** Update specific kubeconfig fields.  

```bash
oc config set clusters.mycluster.server https://new-api.example.com:6443
```

---

### ▶️ `oc config unset`  
**Description:** Remove an individual kubeconfig value.  
**Use case:** Reset a specific field.  

```bash
oc config unset users.dev-user.client-key-data
```

---

### ▶️ `oc config view`  
**Description:** Display merged kubeconfig settings.  
**Use case:** Debug configurations, especially with multiple kubeconfig files.  

```bash
oc config view --minify --flatten
```

---

## 📌 5. Advanced Admin Commands  

### ▶️ `oc config refresh-ca-bundle`  
**Description:** Update the CA bundle from the API server.  
**Use case:** Fix trust issues when certificates are updated.  

```bash
oc config refresh-ca-bundle
```

---

### ▶️ `oc config new-admin-kubeconfig`  
**Description:** Generate a fresh admin kubeconfig.  
**Use case:** Post-installation setup or recovery.  

```bash
oc config new-admin-kubeconfig --user=cluster-admin
```

---

### ▶️ `oc config new-kubelet-bootstrap-kubeconfig`  
**Description:** Generate a bootstrap kubeconfig for kubelet.  
**Use case:** Used during node bootstrapping.  

```bash
oc config new-kubelet-bootstrap-kubeconfig --token=abc123xyz
```

---

## ✅ Pro Tips  

1. Always check **`current-context`** before applying resources to avoid mistakes.  
2. Use **separate contexts** for `dev`, `stage`, and `prod` to prevent accidental deployments.  
3. Combine with **`KUBECONFIG` env var** for managing multiple kubeconfig files:  

```bash
export KUBECONFIG=~/.kube/config:~/.kube/dev-config
```

---

# 📚 Summary  

`oc config` is your toolbox for managing OpenShift/Kubernetes access.  
It helps you securely and efficiently manage **clusters, users, and contexts** in your daily DevOps workflow.  

---
