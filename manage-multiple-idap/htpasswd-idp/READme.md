# 🔐 OpenShift HTPasswd Identity Provider (IDP)

A complete, copy‑paste friendly guide to configure an **HTPasswd** Identity Provider on **OpenShift 4.x**. Perfect for lab, POC, or small teams. 🧑‍💻✨

> **Scope:** Create/manage the htpasswd user file, store it as a Secret, plug it into the cluster `OAuth` config, test logins, manage users/groups, and troubleshoot.

---

## 🗺️ Table of Contents

* [Why HTPasswd?](#-why-htpasswd)
* [Prerequisites](#-prerequisites)
* [Quick Start (TL;DR)](#-quick-start-tldr)
* [Step-by-Step Setup](#-step-by-step-setup)

  * [1) Create the htpasswd file](#1-create-the-htpasswd-file)
  * [2) Create/Update the Secret](#2-createupdate-the-secret)
  * [3) Configure the OAuth IDP](#3-configure-the-oauth-idp)
  * [4) Validate login](#4-validate-login)
* [User & Password Management](#-user--password-management)
* [Groups & RBAC](#-groups--rbac)
* [Security Notes & Best Practices](#-security-notes--best-practices)
* [Troubleshooting](#-troubleshooting)
* [Rotate / Update Passwords](#-rotate--update-passwords)
* [Remove HTPasswd IDP](#-remove-htpasswd-idp)
* [FAQ](#-faq)

---

## 💡 Why HTPasswd?

HTPasswd IDP is a **simple, file-backed** authentication method. Useful when:

* You **don’t** have an external IdP (LDAP, SSO, OAuth, etc.).
* You need **quick local users** for labs or short-term access.
* You want **full control** over a small set of usernames/passwords.

> ⚠️ Not recommended for large/production orgs—prefer enterprise IdPs.

---

## ✅ Prerequisites

* OpenShift **4.x** cluster admin access (`cluster-admin`).
* `oc` CLI configured and logged in.
* A **Linux / macOS** shell with `htpasswd` tool available (from `httpd-tools`), or a container alternative.

Install on RHEL/CentOS/Fedora:

```bash
sudo dnf install -y httpd-tools
```

> No root access? Use containerized `htpasswd`:

```bash
podman run --rm -it registry.access.redhat.com/ubi9/ubi bash -lc \
  'dnf -y install httpd-tools && htpasswd -B -c users.htpasswd alice'
```

---

## ⚡ Quick Start (TL;DR)

```bash
# 1) Create local htpasswd file with first user (creates file)
htpasswd -B -c users.htpasswd alice

# 2) Add more users (no -c)
htpasswd -B users.htpasswd bob

# 3) Create secret in openshift-config
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd -n openshift-config

# 4) Configure OAuth to use HTPasswd
cat <<'YAML' | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: local-htpasswd
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
YAML

# 5) Test: log out/in via web console → choose "local-htpasswd"
```

---

## 🧭 Step-by-Step Setup

### 1) Create the htpasswd file

Create the **first** user and the file:

```bash
htpasswd -B -c users.htpasswd alice
```

* `-c` creates the file (use **only once**).
* `-B` uses bcrypt (recommended).

Add more users:

```bash
htpasswd -B users.htpasswd bob
htpasswd -B users.htpasswd charlie
```

> 🔐 **Strong passwords** recommended. You’ll be prompted interactively.

---

### 2) Create/Update the Secret

Store the file as a **Secret** in `openshift-config` (required by OAuth):

```bash
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config
```

Updating later? Use **apply** pattern:

```bash
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

> 💾 Keep a **backup** of your `users.htpasswd` in a secure place (e.g., encrypted vault).

---

### 3) Configure the OAuth IDP

Patch or apply the cluster OAuth to include the HTPasswd provider:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: local-htpasswd            # Name visible on the login page
    mappingMethod: claim            # How identities map to users
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret         # Secret we created in openshift-config
```

Apply:

```bash
oc apply -f oauth-htpasswd.yaml   # or use the heredoc method from TL;DR
```

> ⏳ Wait \~1–2 minutes for the oauth-operator to reconcile.

---

### 4) Validate login

* Log out of the web console.
* On the login page, select the provider **`local-htpasswd`**.
* Sign in as `alice` (or whichever user you created).

CLI test:

```bash
oc login https://api.<cluster-domain>:6443 -u alice -p '<password>'
whoami
oc whoami
```

---

## 👥 User & Password Management

Add a user:

```bash
htpasswd -B users.htpasswd dave
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

Change a user’s password (same command re-prompts):

```bash
htpasswd -B users.htpasswd alice
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

Remove a user:

```bash
htpasswd -D users.htpasswd bob
oc create secret generic htpass-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

> ♻️ After any change to the htpasswd file, **re-apply the Secret** so OAuth picks it up.

---

## 🧩 Groups & RBAC

Create a group and add users:

```bash
oc adm groups new platform-admins
oc adm groups add-users platform-admins alice charlie
```

Grant cluster admin to a group (careful!):

```bash
oc adm policy add-cluster-role-to-group cluster-admin platform-admins
```

Grant project-scoped roles:

```bash
oc adm policy add-role-to-user admin alice -n my-namespace
oc adm policy add-role-to-group edit developers -n my-namespace
```

List users/groups:

```bash
oc get users
oc get groups
```

---

## 🔒 Security Notes & Best Practices

* **Limit usage** to labs/POCs or small teams. Use enterprise IdPs for prod.
* Use **bcrypt** (`-B`), not MD5/crypt.
* Store `users.htpasswd` in a **private repo/vault**; avoid sharing in chat/tools.
* **Rotate** passwords periodically; remove ex-users promptly.
* Consider creating an **admin group** and avoid granting `cluster-admin` to individuals.
* Keep the original `kubeadmin` credentials only for **break-glass**; disable when comfortable with RBAC.

---

## 🛠️ Troubleshooting

**Login option not visible?**

* Confirm OAuth config includes your provider name:

  ```bash
  oc get oauth cluster -o yaml | yq '.spec.identityProviders'
  ```
* Ensure the Secret name/namespace is correct: `openshift-config`.

**Auth changes not taking effect?**

* Re-apply the Secret (apply pattern above).
* Check operator status/logs:

  ```bash
  oc get co oauth
  oc -n openshift-authentication get pods
  oc -n openshift-authentication logs deploy/oauth-openshift
  ```

**"Login failed" for a user?**

* Verify the username exists in `users.htpasswd`.
* Reset password with `htpasswd -B users.htpasswd <user>` and re-apply Secret.

**Multiple IDPs and wrong one selected by default?**

* Order in `spec.identityProviders` matters. Put the preferred provider **first**.

**Stuck or misconfigured?**

* Temporarily use `kubeadmin` to regain access.

---

## 🔁 Rotate / Update Passwords

1. Update the local file with new password(s):

   ```bash
   htpasswd -B users.htpasswd alice
   ```
2. Re-apply the Secret:

   ```bash
   oc create secret generic htpass-secret \
     --from-file=htpasswd=users.htpasswd \
     -n openshift-config --dry-run=client -o yaml | oc apply -f -
   ```
3. Ask users to **log out and back in**.

---

## 🧼 Remove HTPasswd IDP

1. Edit the OAuth and remove the HTPasswd entry from `spec.identityProviders`.
2. Apply the change and wait for reconciliation.
3. Optionally delete the Secret:

   ```bash
   oc -n openshift-config delete secret htpass-secret
   ```

> ⚠️ Ensure you still have **another working IDP** or `kubeadmin` before removal.

---

## ❓ FAQ

**Q: Where is the htpasswd file stored in the cluster?**
A: In a Secret (e.g., `htpass-secret`) in the `openshift-config` namespace.

**Q: Can I manage users from the web console?**
A: Authentication is file-backed—**update the file** and re-apply the Secret.

**Q: Can I use plaintext passwords?**
A: No. Use `htpasswd -B` which stores bcrypt hashes.

**Q: How do I make someone cluster-admin?**
A: Prefer a **group** → `oc adm policy add-cluster-role-to-group cluster-admin <group>`.

**Q: Does this enforce password complexity?**
A: Complexity is **not** enforced by OpenShift; enforce at creation time and via policy in your team.

---

### 🧾 Sample Files

**oauth-htpasswd.yaml**

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: local-htpasswd
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```

**Makefile (optional convenience)**

```makefile
HTPASSWD_FILE ?= users.htpasswd
SECRET_NAME   ?= htpass-secret
NAMESPACE     ?= openshift-config

.PHONY: init add apply secret
init:
	@htpasswd -B -c $(HTPASSWD_FILE) alice

add:
	@htpasswd -B $(HTPASSWD_FILE) $(USER)

secret:
	@oc create secret generic $(SECRET_NAME) \
	  --from-file=htpasswd=$(HTPASSWD_FILE) -n $(NAMESPACE) \
	  --dry-run=client -o yaml | oc apply -f -

apply:
	@oc apply -f oauth-htpasswd.yaml
```

---

### 🏁 You’re done!

You now have a working **HTPasswd IDP** on OpenShift—easy to manage, perfect for labs and quick collaborations. 🎉

