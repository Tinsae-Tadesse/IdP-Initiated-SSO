# 🛡️ Configuring IdP-Initiated SSO with Keycloak as a SAML 2.0 Broker for OIDC Clients

## 📌 Overview

Single Sign-On (SSO) is critical for modern identity management. If your organization uses a centralized SAML-based Identity Provider (IdP) but wants to authenticate users in OIDC-based applications, **Keycloak** can serve as an **identity broker**—bridging SAML assertions to OIDC clients.

This guide walks you through setting up **IdP-Initiated SSO with SAML 2.0**, where Keycloak acts as a SAML proxy to federate authentication into **OIDC clients**.

### ✅ Prerequisites

- A SAML 2.0-compliant Identity Provider is available.
- Keycloak is configured as the identity broker.
- One or more OIDC clients (e.g., `oidc-client-a`, `oidc-client-b`) are configured in Keycloak.
- IdP-Initiated login and Single Logout (SLO) are required.
- [Apache HTTP Server](https://httpd.apache.org/) is used as the web server.

---

## 🧱 High-Level Architecture

![SSO Architecture Diagram](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/assets/Architecture.png?raw=true)

---

## 🔧 Step-by-Step Setup

### 1. 🔗 Configure the External Identity Provider in Keycloak

Create trust between Keycloak and your external SAML IdP:

- Navigate to **Identity Providers > Add provider > SAML v2.0**.
- Set:
  - **Alias**: `external-saml-idp`
  - **Import Metadata** from the IdP’s metadata URL or XML file.
- Verify the **Single Sign-On Service URL** is populated.
- Set the **Single Logout Service URL** (for SLO).
- Enable **Want AuthnRequests Signed** if required.
- Map attributes like `email`, `username`, etc.

#### Example Screenshots:

![Identity Provider Configuration - 1](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/assets/idp-config-1.jpg?raw=true)

![Identity Provider Configuration - 2](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/assets/idp-config-2.jpg?raw=true)

---

### 2. 🧹 Create a SAML Proxy Client

This acts as the **Assertion Consumer Service (ACS)** endpoint.

- Go to **Clients > Create**.
- Configure:
  - **Client ID**: `idpSAMLProxyClient`
  - **Client Protocol**: `SAML`
  - **IdP-Initiated SSO URL Name**: `idpSAMLProxyClient`
  - **ACS POST Binding URL**: `/sso/oidc-client-a/login`
  - Enable **Force POST Binding**.
- Configure signing/encryption based on your IdP’s needs.

> 💡 **Important:**
> - The `IdP-Initiated SSO URL Name` defines the public URL Keycloak exposes for IdP-initiated logins:
>   ```
>   https://<keycloak-domain>/realms/<realm-name>/broker/<idp-alias>/endpoint/clients/<sso-url-name>
>   ```
> - Example:
>   ```
>   https://example.com/realms/my-realm/broker/external-saml-idp/endpoint/clients/idpSAMLProxyClient
>   ```
> - If you omit `IdP-Initiated SSO URL Name`, Keycloak falls back to the default ACS URL:
>   ```
>   https://<keycloak-domain>/realms/<realm-name>/broker/<idp-alias>/endpoint
>   ```

#### Example Screenshots:

![SAML Client Configuration - 1](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/assets/saml-client-config-1.jpg?raw=true)

![SAML Client Configuration - 2](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/assets/saml-client-config-2.jpg?raw=true)

---

### 3. 🧐 Customize First Login Authentication Flow

Control user provisioning and account linking:

- Go to **Authentication > Flows > First Broker Login**.
- **Copy** the flow and rename it (e.g., `First Broker Login Flow - IDP`).
- Update steps:
  - Enable `Create User If Unique`.
  - Enable `Handle Existing Account`.
  - Disable unnecessary steps like `Confirm Link Existing Account`, `Verify Existing Account By Email`, and `Review Profile`.

> 🛍️ Apply this new flow in your IdP settings under **First Login Flow**.

---

### 4. 🌐 Modify the Browser Flow for Seamless Redirection

Automatically redirect unauthenticated users to the IdP:

- Go to **Authentication > Flows > Browser**.
- **Copy** the flow, rename to `Browser Flow - IDP`.
- Add the **Identity Provider Redirector**.
- Set the **Default Identity Provider** to your IdP alias.
- Under **Bindings > Browser Flow**, set this as the default.

---

### 5. ⚙️ Add an Apache HTTPD Rewrite Rule

Forward SAML assertion POSTs to your OIDC client:

```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} POST
RewriteRule ^/sso/oidc-client-a/login$ https://oidc-client-a.example.com/ [R=302,L]
```

> 🔁 Update the rule according to your OIDC client’s URL.

---

## 💪 Final Verification

- Trigger login via the IdP’s app launcher.
- Validate that Keycloak processes the SAML assertion correctly.
- Ensure users are redirected to the appropriate OIDC application.
- Confirm user provisioning and account linking behavior.
- Test Single Logout (SLO) functionality.

---

## ✅ Conclusion

By bridging SAML IdPs with modern OIDC applications through Keycloak, you can create a seamless and secure IdP-Initiated SSO experience backed by the smart use of SAML proxying, custom authentication flows, and HTTPD rewrites.

---
