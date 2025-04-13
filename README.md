# 🛡️ Configuring IdP-Initiated SSO for Keycloak as a SAML 2.0 Identity Broker for OIDC Clients

## 📌 Overview

Single Sign-On (SSO) is essential for modern identity and access management. In cases where an organization relies on a centralized SAML-based Identity Provider (IdP) but still wants to authenticate users in OIDC-based applications, **Keycloak** can act as an **identity broker**—bridging SAML assertions to OIDC clients.

This guide explains how to configure **IdP-Initiated SSO using SAML 2.0**, with Keycloak serving as a SAML proxy that federates authentication to **OIDC clients** behind the scenes.

### ✅ Assumptions

- You have a SAML 2.0-compliant Identity Provider.
- Keycloak is used as the intermediary (identity broker) between the IdP and OIDC applications.
- One or more OIDC clients are already configured in Keycloak (e.g., `oidc-client-a`, `oidc-client-b`).
- You want to enable **IdP-initiated login** and **Single Logout (SLO)**.

---

## 🧱 High-Level Architecture

![SSO Architecture Diagram](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/Architecture.png?raw=true)

---

## 🔧 Step-by-Step Implementation

### 1. 🧩 Create a SAML Proxy Client in Keycloak

This client will act as the **Assertion Consumer Service (ACS)** endpoint to receive SAML assertions from the IdP.

- Go to **Clients > Create** in the Keycloak admin console.
- Set:
  - **Client ID**: `saml-proxy-client` (or any name).
  - **Client Protocol**: `SAML`.
  - **Valid Redirect URIs**: URL(s) for your OIDC client app(s).
  - **Assertion Consumer Service POST Binding URL**: `/sso/login` (or any endpoint you’ll use).
  - Enable **Force POST Binding**.

> 💡 Configure signature and encryption settings to match your IdP requirements.

**Default ACS URL (used by Keycloak for SAML assertions):**
```
https://<keycloak-domain>/auth/realms/<realm-name>/broker/<idp-alias>/endpoint
```
**Example:**
```
https://example.com/auth/realms/my-realm/broker/external-idp/endpoint
```

---

### 2. 🔗 Configure the Identity Provider in Keycloak

This sets up trust between Keycloak and the external SAML IdP.

- Go to **Identity Providers > Add provider > SAML v2.0**.
- Set:
  - **Alias**: `external-idp`.
  - **IdP-Initiated SSO URL Name**: e.g., `external-idp-saml-my-client-app`.
  - **Import Metadata**: from the IdP’s metadata URL or XML file.
- Ensure **Single Sign-On Service URL** is populated automatically.
- Set **Single Logout Service URL** for SLO support.
- Enable **Want AuthnRequests Signed**, if required by the IdP.
- Configure attribute mappers for fields like `email`, `username`, etc.

---

### 3. 🧠 Customize the First Login Authentication Flow

This ensures correct user provisioning and linking during the first login from the IdP.

- Navigate to **Authentication > Flows > First Broker Login**.
- Click **Copy**, and rename it (e.g., `First Broker Login - Custom`).
- Edit the flow to add or configure:
  - `Create User If Unique` – Automatically create users if they don’t exist.
  - `Handle Existing Account` – Prevent duplicate accounts and guide linking.
  - Optional authenticators (based on your needs):
    - `Confirm Link Existing Account` – *Disabled* (no user confirmation).
    - `Verify Existing Account By Email` – *Disabled*.
    - `Verify Existing Account By Re-authentication` – *Disabled*.
    - `Review Profile` – *Disabled*.

> 🧭 Apply this custom flow to the IdP under **First Login Flow** in the IdP settings.

---

### 4. 🌐 Modify the Browser Flow for Seamless Redirection

This forces unauthenticated users to be redirected directly to the IdP.

- Go to **Authentication > Flows > Browser**.
- Click **Copy**, rename to `Browser - Auto Redirect`.
- Add the **Identity Provider Redirector** authenticator.
- Set the **Default Identity Provider** to the IdP alias created in Step 2.
- Go to **Bindings > Browser Flow**, and select your new flow as the default.

---

### 5. ⚙️ Add Apache HTTPD Rewrite Rule

This rule intercepts SAML assertion POSTs and forwards them to the target OIDC client.

Add the following to your Apache configuration:

```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} POST
RewriteRule ^/sso/login$ https://oidc-client-a.example.com/ [R=302,L]
```

> 🔁 Adjust the target URL as needed for your specific OIDC client(s).

---

## 🧪 Final Verification

- Trigger login from the IdP’s app launcher.
- Ensure Keycloak accepts and processes the SAML assertion.
- Confirm that users are redirected to the correct OIDC client.
- Verify user provisioning or account linking.
- Test Single Logout (SLO) to ensure proper session termination.

---

## ✅ Conclusion

Keycloak simplifies the process of bridging legacy SAML identity systems with modern OIDC applications. By combining SAML proxying, custom authentication flows, and reverse proxy rewrites, you can create a seamless and secure IdP-initiated SSO experience for your users.

---
