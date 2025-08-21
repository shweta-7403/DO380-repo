# 🔑 OpenShift `oc auth` Commands — Complete Guide  

The `oc auth` command in OpenShift provides ways to **check, verify, and manage authentication and authorization** for users in the cluster.  
It is mainly used for **RBAC (Role-Based Access Control)** related checks and understanding what a user can or cannot do.  

This guide explains all key `oc auth` subcommands with **use cases** and **examples**.  

---

## 📌 1. Check User Permissions  

### ▶️ `oc auth can-i <verb> <resource>`  
**Description:** Checks if the current user (or a specified user) can perform an action on a resource.  
**Use case:** Useful for verifying RBAC permissions before attempting an action.  

```bash
# Check if the current user can create pods in the current namespace
oc auth can-i create pods

# Check if the current user can delete deployments in 'dev-project'
oc auth can-i delete deployments --namespace=dev-project

# Check for another user
oc auth can-i get pods --as=developer
```

✅ **Real use case:** Before giving developers access, cluster admins often verify if the permissions are set correctly using `can-i`.  

---

### ▶️ `oc auth can-i --list`  
**Description:** Lists all actions the user is allowed or denied on resources.  
**Use case:** Get a complete view of what the user can do in a namespace.  

```bash
oc auth can-i --list
oc auth can-i --list --namespace=prod-project
```

✅ **Real use case:** Useful during audits and troubleshooting when a user reports **“I cannot access XYZ resource.”**  

---

### ▶️ `oc auth can-i <verb> <resource> --all-namespaces`  
**Description:** Checks permissions across all namespaces.  
**Use case:** Verify cluster-wide permissions.  

```bash
oc auth can-i get pods --all-namespaces
```

✅ **Real use case:** Cluster admins verify if a service account has global read permissions across all namespaces.  

---

## 📌 2. Using `--as` for Impersonation  

### ▶️ `oc auth can-i <verb> <resource> --as <user>`  
**Description:** Test what actions another user would be able to do.  
**Use case:** Debug and validate RBAC for different users without switching accounts.  

```bash
# Check if 'alice' can create services
oc auth can-i create services --as=alice

# Check if 'system:serviceaccount:dev:build-bot' can create pods
oc auth can-i create pods --as=system:serviceaccount:dev:build-bot
```

✅ **Real use case:** Helps admins validate access for **service accounts** and **end-users**.  

---

## 📌 3. Group-Based Permission Checks  

### ▶️ `oc auth can-i <verb> <resource> --as-group <group>`  
**Description:** Test what actions a group of users can perform.  
**Use case:** Validate group-level RBAC assignments.  

```bash
# Check if members of 'developers' group can create routes
oc auth can-i create route --as-group=developers
```

✅ **Real use case:** When permissions are granted via groups, this helps confirm group-level access.  

---

## 📌 4. Resource and Verb Variations  

You can check permissions with more granularity:  

```bash
# Can the user update a specific resource type?
oc auth can-i update configmap

# Can the user patch a secret in the dev namespace?
oc auth can-i patch secret --namespace=dev

# Can the user list nodes (cluster-wide resource)?
oc auth can-i list nodes
```

✅ **Real use case:** Verify if DevOps engineers have just enough access to do operational tasks without cluster-admin rights.  

---

## 📌 5. Example Scenarios  

### 🔹 Scenario 1: Developer reports "I can’t deploy apps"  
```bash
oc auth can-i create deployment --as=developer --namespace=dev
```  
If this returns **“no”**, check RBAC roles.  

---

### 🔹 Scenario 2: ServiceAccount for CI/CD pipeline  
```bash
oc auth can-i create imagestream --as=system:serviceaccount:ci:jenkins
```  
Ensures Jenkins has permissions to push images.  

---

### 🔹 Scenario 3: Security Audit  
```bash
oc auth can-i --list --as=alice --namespace=prod
```  
Lists everything Alice can do in production.  

---

## ✅ Pro Tips  

1. `oc auth can-i` is the **RBAC debugging Swiss Army knife**.  
2. Always combine with `--as` and `--as-group` for testing impersonation.  
3. Use `--list` to quickly audit permissions.  
4. Remember: **Cluster-scoped resources** (like `nodes`, `namespaces`) need cluster roles.  

---

# 📚 Summary  

`oc auth` is essential for **testing and validating RBAC permissions** in OpenShift.  
It helps admins, developers, and auditors confirm whether a user or service account has the correct access rights.  

---
