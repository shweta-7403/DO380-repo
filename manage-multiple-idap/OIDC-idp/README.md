# 🔓 OpenShift OpenID Connect (OIDC) Identity Provider (IDP)

A complete, GitHub‑ready guide to integrate **OpenShift 4.x** with any **OpenID Connect (OIDC)** provider (e.g., **Keycloak**, **Dex**, **Azure AD (Entra ID)**, **Okta**, **Google**). 🧑‍💻✨

> **Scope:** Prepare your OIDC app, configure OpenShift OAuth, map claims, secure with custom CA if needed, test login, sync groups/RBAC, and troubleshoot.

---

## 🗺️ Table of Contents

* [Why OIDC?](#-why-oidc)
* [Prerequisites](#-prerequisites)
* [Quick Start (TL;DR)](#-quick-start-tldr)
* [Step-by-Step Setup](#-step-by-step-setup)

  * [1) Create an OIDC app / client](#1-create-an-oidc-app--client)
  * [2) Create the client secret in OpenShift](#2-create-the-client-secret-in-openshift)
  * [3) Configure the OAuth with OIDC](#3-configure-the-oauth-with-oidc)
  * [4) Validate login](#4-validate-login)
* [Groups & RBAC](#-groups--rbac)
* [Examples (Keycloak, Entra ID, Okta)](#-examples-keycloak-entra-id-okta)
* [Security Notes & Best Practices](#-security-notes--best-practices)
* [Troubleshooting](#-troubleshooting)
* [Remove OIDC IDP](#-remove-oidc-idp)
* [FAQ](#-faq)

---

## 💡 Why OIDC?

OIDC is a modern, interoperable protocol built on OAuth 2.0. Benefits:

* **Single Sign‑On (SSO)** with your existing provider.
* **Centralized policies** (MFA, password rules, conditional access) managed by your IdP.
* **Standards‑based claims** mapping to OpenShift users and groups.
* **Scalability** and production readiness.

---

## ✅ Prerequisites

* OpenShift **4.x** cluster and **cluster‑admin** access.
* `oc` CLI configured.
* An OIDC provider where you can register an app/client (e.g., Keycloak realm/client or Entra ID App Registration).
* OIDC **Issuer URL** (supports discovery via `/.well-known/openid-configuration`).
* **Client ID** and **Client Secret** for the app.
* **Redirect URI** for OpenShift OAuth callback.

> 🔎 **Find your OpenShift OAuth callback URL:**

```bash
oc -n openshift-authentication get route oauth-openshift \
  -o jsonpath='https://{.spec.host}/oauth2callback/oidc'
# Example output:
# https://oauth-openshift.apps.ocp4.example.com/oauth2callback/oidc
```

Use that URL when registering your OIDC client. The last path segment (`oidc`) must match the **IDP `name`** you configure in OAuth.

---

## ⚡ Quick Start (TL;DR)

```bash
# 1) Secret for client credentials
oc create secret generic oidc-client-secret \
  -n openshift-config \
  --from-literal=clientSecret='REPLACE_ME'

# 2) (Optional) Custom CA bundle for your OIDC issuer
# oc create configmap oidc-ca --from-file=ca.crt=issuer-ca.pem -n openshift-config

# 3) Configure OAuth with the OpenID provider
cat <<'YAML' | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: oidc                          # appears on login page, must match callback path
    mappingMethod: claim
    type: OpenID
    openID:
      issuer: https://keycloak.example.com/realms/ocp
      clientID: openshift-console
      clientSecret:
        name: oidc-client-secret
      # ca:
      #   name: oidc-ca                  # uncomment if using a private CA
      claims:
        preferredUsername: ["preferred_username", "email", "upn"]
        name: ["name", "given_name"]
        email: ["email"]
        groups: ["groups"]
      # extraScopes extends the provider defaults
      extraScopes: ["openid", "email", "profile", "groups"]
YAML

# 4) Test login via console → choose "oidc"
```

---

## 🧭 Step-by-Step Setup

### 1) Create an OIDC app / client

In your IdP (Keycloak/Okta/Entra ID):

1. **Register an application/client** (confidential).
2. Set **Redirect URI** to your cluster callback (see command above), e.g.:
   `https://oauth-openshift.apps.ocp4.example.com/oauth2callback/oidc`
3. Enable **Standard Flow / Authorization Code**.
4. Configure **allowed scopes**: at minimum `openid`; commonly add `email`, `profile`, and `groups`.
5. Ensure desired **claims** are present in ID token or userinfo: `preferred_username`, `email`, `name`, optionally `groups`.
6. Note the **Issuer URL**, **Client ID**, and **Client Secret**.

> ℹ️ For Azure AD (Entra ID), groups may require **group claims** enabled for the app. For Keycloak, add a **Group Membership** protocol mapper if needed.

---

### 2) Create the client secret in OpenShift

Store the client secret in `openshift-config`:

```bash
oc create secret generic oidc-client-secret \
  -n openshift-config \
  --from-literal=clientSecret='REPLACE_ME'
```

If your issuer uses a **private CA**, add it:

```bash
oc create configmap oidc-ca --from-file=ca.crt=issuer-ca.pem -n openshift-config
```

---

### 3) Configure the OAuth with OIDC

Create or patch the cluster `OAuth`:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: oidc
    mappingMethod: claim
    type: OpenID
    openID:
      issuer: https://keycloak.example.com/realms/ocp
      clientID: openshift-console
      clientSecret:
        name: oidc-client-secret
      # ca:
      #   name: oidc-ca
      claims:
        preferredUsername: ["preferred_username", "email", "upn"]
        name: ["name", "given_name"]
        email: ["email"]
        groups: ["groups"]
      extraScopes: ["openid", "email", "profile", "groups"]
```

Apply it:

```bash
oc apply -f oauth-oidc.yaml
```

> ⏳ The oauth-operator reconciles in \~1–2 minutes. Revisit the login page afterwards.

---

### 4) Validate login

* Log out of the console and choose the **oidc** provider.
* Sign in with your IdP user. First login creates an OpenShift `User` object.

CLI test:

```bash
oc login https://api.<cluster-domain>:6443 -u ignored -p ignored --web
# (Browser flow opens to complete OIDC auth)
```

---

## 👥 Groups & RBAC

If your IdP emits `groups` claims, OpenShift will attach them to the `User` object. Common patterns:

```bash
# Grant cluster-admin to a specific IdP group (be careful!)
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins

# Project-scoped permissions
oc adm policy add-role-to-group edit developers -n my-namespace
```

> 🧩 When using Azure AD or Okta, ensure the app is configured to include **group claims** in tokens (by ID or name). For Keycloak, add a **Group Membership** mapper to ID token and/or userinfo.

---

## 🧪 Examples (Keycloak, Entra ID, Okta)

### 🔑 Keycloak (Realm: `ocp`, Client: `openshift-console`)

* Enable **Standard Flow**.
* Mappers: `preferred_username`, `email`, `name`, optional `groups` (Group Membership mapper).
* Issuer: `https://keycloak.example.com/realms/ocp`

```yaml
openID:
  issuer: https://keycloak.example.com/realms/ocp
  clientID: openshift-console
  clientSecret:
    name: oidc-client-secret
  claims:
    preferredUsername: ["preferred_username"]
    name: ["name"]
    email: ["email"]
    groups: ["groups"]
  extraScopes: ["openid", "email", "profile", "groups"]
```

### 🟦 Microsoft Entra ID (Azure AD)

* Issuer format: `https://login.microsoftonline.com/<TENANT_ID>/v2.0` (multi-tenant varies).
* App Registration → Authentication → add Redirect URI.
* Token configuration → add **ID tokens** and **Group claims**.

```yaml
openID:
  issuer: https://login.microsoftonline.com/<TENANT_ID>/v2.0
  clientID: <APP_CLIENT_ID>
  clientSecret:
    name: oidc-client-secret
  claims:
    preferredUsername: ["upn", "email"]
    name: ["name"]
    email: ["email"]
    groups: ["groups"]
  extraScopes: ["openid", "email", "profile"]
```

### 🟧 Okta

* Issuer: `https://<OKTA_DOMAIN>/oauth2/<AUTH_SERVER_ID>` (or `/oauth2/default`).
* Add Redirect URI, enable `groups` in claims (regex-based group filter if needed).

```yaml
openID:
  issuer: https://dev-123456.okta.com/oauth2/default
  clientID: <OKTA_CLIENT_ID>
  clientSecret:
    name: oidc-client-secret
  claims:
    preferredUsername: ["preferred_username", "email"]
    name: ["name"]
    email: ["email"]
    groups: ["groups"]
  extraScopes: ["openid", "email", "profile", "groups"]
```

---

## 🔒 Security Notes & Best Practices

* Prefer **HTTPS** issuer with a trusted CA; if private CA, mount via `ca` ConfigMap.
* Restrict app to **Authorization Code** flow; avoid implicit/hybrid.
* Limit scopes to what you need; include `groups` only if used.
* **Rotate** client secrets regularly (recreate Secret → `oc apply` OAuth unchanged).
* Use **groups‑based RBAC** rather than individual user grants.
* Keep `kubeadmin` as break‑glass until OIDC is verified and an admin group has cluster‑admin.

---

## 🛠️ Troubleshooting

**Login option missing?**

* Check OAuth config:

  ```bash
  oc get oauth cluster -o yaml | yq '.spec.identityProviders'
  ```
* Confirm the `name` matches your callback path segment and that the callback URI is registered in IdP.

**`invalid_client` or `unauthorized_client`**

* Client ID/secret mismatch or redirect URI not exact. Update IdP app to match.

**`invalid_request: redirect_uri`**

* Ensure redirect URI equals `https://<oauth-route>/oauth2callback/<idpName>`.

**Issuer / JWKS errors**

* Confirm `issuer` exactly matches discovery doc. Import CA if private.

**Groups not appearing**

* Enable groups in IdP (Azure AD: token configuration; Okta: claim mapping; Keycloak: Group mapper) and include `groups` in `extraScopes` if required by the IdP.

**Operator status/logs**

```bash
oc get co oauth
oc -n openshift-authentication get pods
oc -n openshift-authentication logs deploy/oauth-openshift
```

---

## 🧼 Remove OIDC IDP

1. Edit `OAuth` and remove the OpenID entry.
2. Delete related secrets/configmaps if unused:

```bash
oc -n openshift-config delete secret oidc-client-secret
# oc -n openshift-config delete configmap oidc-ca
```

---

## ❓ FAQ

**Q: Do I need to specify authorization/token endpoints?**
A: No—OpenShift uses **OIDC discovery** from the `issuer`.

**Q: Can I configure multiple OIDC providers?**
A: Yes, add multiple entries under `spec.identityProviders`.

**Q: Where are users stored?**
A: OpenShift creates `User` objects on first login; identities come from your IdP.

**Q: Can I force email as username?**
A: Map `claims.preferredUsername` to `["email"]`.

**Q: Can I use PKCE?**
A: OpenShift behaves as a confidential client (server‑side), using client secret; PKCE typically isn’t required.

---

### 🧾 Sample files

**oauth-oidc.yaml**

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: oidc
    mappingMethod: claim
    type: OpenID
    openID:
      issuer: https://keycloak.example.com/realms/ocp
      clientID: openshift-console
      clientSecret:
        name: oidc-client-secret
      claims:
        preferredUsername: ["preferred_username"]
        name: ["name"]
        email: ["email"]
        groups: ["groups"]
      extraScopes: ["openid", "email", "profile", "groups"]
```

**Makefile (optional)**

```makefile
OIDC_NAME     ?= oidc
CLIENT_SECRET ?= oidc-client-secret
NAMESPACE     ?= openshift-config

.PHONY: secret apply route-url
secret:
	@oc create secret generic $(CLIENT_SECRET) -n $(NAMESPACE) \
	  --from-literal=clientSecret=$$(read -p "Client Secret: " s; echo $$s) \
	  --dry-run=client -o yaml | oc apply -f -

route-url:
	@echo https://$$(oc -n openshift-authentication get route oauth-openshift \
	  -o jsonpath='{.spec.host}')/oauth2callback/$(OIDC_NAME)

apply:
	@oc apply -f oauth-oidc.yaml
```

---

### 🏁 You’re done!

OpenShift is now wired up to your **OIDC** provider—SSO with your corporate credentials, clean claims mapping, and group‑based RBAC. 🎉

> Want me to tailor this README for **Keycloak** or **Azure AD** with exact screenshots and claim mappers? Ping me and I’ll extend it. 💬

