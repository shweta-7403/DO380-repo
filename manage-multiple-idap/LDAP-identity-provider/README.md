# 🗂️ OpenShift LDAP Identity Provider (IDP)

A detailed guide to configure an **LDAP Identity Provider** on **OpenShift 4.x**. Ideal for integrating OpenShift authentication with your enterprise directory (e.g., Microsoft Active Directory, OpenLDAP). 🧑‍💻✨

> **Scope:** Configure LDAP IDP, manage bind credentials, map LDAP groups to OpenShift groups, test authentication, and apply RBAC.

---

## 🗺️ Table of Contents

* [Why LDAP?](#-why-ldap)
* [Prerequisites](#-prerequisites)
* [Quick Start (TL;DR)](#-quick-start-tldr)
* [Step-by-Step Setup](#-step-by-step-setup)

  * [1) Prepare LDAP/AD server info](#1-prepare-ldapad-server-info)
  * [2) Create Bind Secret](#2-create-bind-secret)
  * [3) Configure OAuth with LDAP](#3-configure-oauth-with-ldap)
  * [4) Validate login](#4-validate-login)
* [Group Sync & RBAC](#-group-sync--rbac)
* [Security Notes](#-security-notes)
* [Troubleshooting](#-troubleshooting)
* [Remove LDAP IDP](#-remove-ldap-idp)
* [FAQ](#-faq)

---

## 💡 Why LDAP?

LDAP/AD IDP allows OpenShift to integrate with **existing enterprise directory services**:

* Centralized **user management** and password policies.
* Users log in with their **corporate credentials**.
* Easier **RBAC via group sync**.
* Secure, scalable, and production-ready (unlike htpasswd).

---

## ✅ Prerequisites

* OpenShift **4.x** cluster admin access (`cluster-admin`).
* `oc` CLI configured.
* An accessible **LDAP or Active Directory server**.
* LDAP service account credentials (bind DN + password).

Example details to collect:

* **LDAP URL**: `ldap://ldap.example.com:389` or `ldaps://ldap.example.com:636`
* **Bind DN**: `cn=ldap-reader,ou=ServiceAccounts,dc=example,dc=com`
* **Bind Password**: (store securely in secret)
* **User Search Base DN**: `ou=Users,dc=example,dc=com`
* **User Attribute**: `uid` or `sAMAccountName` (AD)
* **Group Search Base DN**: `ou=Groups,dc=example,dc=com`

---

## ⚡ Quick Start (TL;DR)

```bash
# 1) Create bind password secret
oc create secret generic ldap-bind-secret \
  --from-literal=bindPassword='SuperSecret123' \
  -n openshift-config

# 2) Apply OAuth config with LDAP provider
cat <<'YAML' | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ldap
    mappingMethod: claim
    type: LDAP
    ldap:
      url: "ldap://ldap.example.com:389/ou=Users,dc=example,dc=com?uid"
      bindDN: "cn=ldap-reader,ou=ServiceAccounts,dc=example,dc=com"
      bindPassword:
        name: ldap-bind-secret
      insecure: true
      attributes:
        id: ["dn"]
        preferredUsername: ["uid"]
        name: ["cn"]
        email: ["mail"]
YAML

# 3) Test login with LDAP user
oc login -u alice -p password https://api.<cluster-domain>:6443
```

---

## 🧭 Step-by-Step Setup

### 1) Prepare LDAP/AD server info

Gather:

* Hostname, port (`389` for LDAP, `636` for LDAPS).
* Base DN for users and groups.
* Bind DN with read/search permissions.
* Attribute used as username (`uid` for LDAP, `sAMAccountName` for AD).

---

### 2) Create Bind Secret

```bash
oc create secret generic ldap-bind-secret \
  --from-literal=bindPassword='SuperSecret123' \
  -n openshift-config
```

> 🔐 Store secrets securely. Use a password vault in production.

---

### 3) Configure OAuth with LDAP

Create or patch OAuth:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ldap-provider                 # Visible on login page
    mappingMethod: claim
    type: LDAP
    ldap:
      url: "ldap://ldap.example.com:389/ou=Users,dc=example,dc=com?sAMAccountName"
      bindDN: "cn=ldap-reader,ou=ServiceAccounts,dc=example,dc=com"
      bindPassword:
        name: ldap-bind-secret
      insecure: true                    # Use 'false' with LDAPS
      attributes:
        id: ["dn"]
        preferredUsername: ["sAMAccountName"]
        name: ["cn"]
        email: ["mail"]
```

Apply:

```bash
oc apply -f oauth-ldap.yaml
```

---

### 4) Validate login

* Log out of the OpenShift console.
* Choose provider **ldap-provider** on login.
* Login with corporate user (e.g., `alice`).

CLI test:

```bash
oc login -u alice -p 'Password1' https://api.<cluster-domain>:6443
oc whoami
```

---

## 👥 Group Sync & RBAC

Map LDAP/AD groups to OpenShift groups.

1. Create an LDAP group sync config:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://ldap.example.com:389
bindDN: "cn=ldap-reader,ou=ServiceAccounts,dc=example,dc=com"
bindPassword: "SuperSecret123"
groupUIDNameMapping:
  "cn=ocp-admins,ou=Groups,dc=example,dc=com": ocp-admins
  "cn=dev-team,ou=Groups,dc=example,dc=com": developers
```

2. Run group sync:

```bash
oc adm groups sync --sync-config=ldap-sync.yaml --confirm
```

3. Apply RBAC:

```bash
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
oc adm policy add-role-to-group edit developers -n my-project
```

---

## 🔒 Security Notes

* Always prefer **LDAPS (636)** with `insecure: false`.
* Limit bind account to **read-only** access.
* Rotate bind password periodically.
* Avoid storing passwords in Git—use secrets.
* Apply least-privilege RBAC via group sync, not individuals.

---

## 🛠️ Troubleshooting

**Login option not visible?**

* Confirm IDP is listed:

  ```bash
  oc get oauth cluster -o yaml | yq '.spec.identityProviders'
  ```

**Bind/auth errors?**

* Check Secret name matches `bindPassword.name`.
* Verify DN syntax matches LDAP server.

**Group sync fails?**

* Test LDAP query manually with `ldapsearch`.

**Certificates issues with LDAPS?**

* Import CA cert into OpenShift trusted bundle:

  ```bash
  oc create configmap ldap-ca --from-file=ca.crt=ldap-ca.pem -n openshift-config
  ```

  Add to OAuth:

  ```yaml
  ca:
    name: ldap-ca
  ```

**Stuck?**

* Use `kubeadmin` to log in.

---

## 🧼 Remove LDAP IDP

1. Edit OAuth config and remove the LDAP provider entry.
2. Delete secret if no longer used:

```bash
oc -n openshift-config delete secret ldap-bind-secret
```

---

## ❓ FAQ

**Q: Can I configure multiple LDAP servers?**
A: Yes, add multiple `identityProviders` entries in OAuth.

**Q: Can I use both htpasswd and LDAP?**
A: Yes, multiple IDPs are supported.

**Q: How do I change LDAP search base?**
A: Update `url` field in OAuth config.

**Q: Does OpenShift enforce password expiry?**
A: Password policy is managed by LDAP/AD, not OpenShift.

**Q: Can I sync only specific groups?**
A: Yes, define mappings in `LDAPSyncConfig`.

---

### 🏁 Done!

You now have **LDAP authentication integrated with OpenShift**. Corporate users can log in using their existing credentials. 🎉

> For production, use **LDAPS**, configure **group sync**, and apply RBAC via groups instead of individuals. ✅

