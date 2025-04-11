
# 🛡️ Configuring IdP-Initiated SSO for Keycloak as a SAML 2.0 Identity Broker for OIDC Clients

## 📌 Overview
Single Sign-On (SSO) is a cornerstone of modern identity and access management strategies. In scenarios where an organization uses a centralized Identity Provider (IdP) and still wants to delegate application-level authentication through OpenID Connect (OIDC), Keycloak can act as an **identity broker**—bridging SAML-based identity assertions from an external IdP to OIDC-based clients.

This guide outlines how to configure **IdP-Initiated SSO using SAML 2.0**, with Keycloak acting as a SAML proxy that federates authentication to **two OIDC clients** behind the scenes.

Assumtions:

- You have a SAML 2.0-compliant Identity Provider.
- You use Keycloak as an intermediary (broker) between the IdP and your OIDC-based applications.
- One or more OIDC clients are configured in Keycloak (like oidc-client-a, oidc-client-b, etc.).
- You want to enable IdP-initiated login and single logout (SLO).

---

## 🧱 High-Level Implementation Architecture
![SSO Architecture Diagram](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/Architecture.png?raw=true)

---

## 🔧 Implementation Steps

### 1. 🧩 Create a SAML Proxy Client in Keycloak

In Keycloak, a **SAML client** acts as the Assertion Consumer Service (ACS) endpoint. This client will receive and validate SAML assertions from your external IdP.

- Navigate to **Clients > Create**.
- Set the **Client ID** as `saml-proxy-client` (or any name).
- Set the **Client Protocol** as `SAML`.
- Configure **Valid Redirect URIs** with a proper URL to the OIDC client application.
- Set **Assertion Consumer Service POST Binding URL** to `/sso/login` (or any url) to receive and process SAML assertions sent from the identity provider using HTTP POST binding through this endpoint.
- Enable **Force POST Binding**.

Save and configure signature and encryption settings as per your IdP requirements.

**With these settings, Keycloak’s default Assertion Consumer Service (ACS) URL will handle SAML assertions for this client.**
The default ACS URL used by keycloak is:
```
https://<keycloak-domain>/auth/realms/<realm-name>/broker/<idp-alias>/endpoint
```
An actual ACS URL will look like the one below:
```
https://example.com/auth/realms/my-realm/broker/external-idp/endpoint
```

---

### 2. 🔗 Set Up the Identity Provider in Keycloak

This step allows Keycloak to **trust the external IdP** and receive SAML assertions.

- Go to **Identity Providers > Add provider > SAML v2.0**.
- Set an **Alias** as `external-idp`.
- Set an **IdP-Initiated SSO URL Name** to a unique name (e.g., `external-idp-saml-my-client-app`) to support IdP-initiated flows.
- **Import the metadata from URL or XML file of the IdP**: Use the IdP’s entity descriptor metadata URL or upload the XML file pointing to the SAML IdP.
- **Single Sign-On Service URL** is auto-filled from the metadata URL of XML file.
- Configure the **Single Logout Service URL** to support Single Logout (SLO).
- Enable **Want AuthnRequests Signed** depending on the external IdP configuration.

Adjust mappers to link SAML attributes like `email`, `username`, etc and Save.

---

### 3. 🧠 Customize the First Login Authentication Flow

The goal of this step is to automatically **provision new users** and associate existing users correctly when logging in via the external IdP.

- Go to **Authentication > Flows > First Broker Login**.
- Click **Copy** to duplicate the `First Broker Login` flow and name it as `First Broker Login - Custom`.
- Edit the new flow:
  - Add **Create User If Unique** to tell the authenticator to create a new local Keycloak account and link it with the identity provider, if account with the same email or username does not exist.
  - Add **Handle Existing Account** to show the user an error message if there is an existing Keycloak account and the user will need to be linked to the identity provider through account management.
  - Add **Confirm Link Existing Account** to let the users confirm that they want to link their identity provider account with their existing Keycloak account. This authenticator is `disabled`, in my case, as i do not want users to see this confirmation page.
  - Add **Verify Existing Account By Email** to sends an email to users to confirm that they want to link the identity provider with their Keycloak account. This authenticator is `disabled`, in my case, as i do not want users to confirm linking by email.
  - Add **Verify Existing Account By Re-authentication** to display a login screen for users to re-authenticate and link their Keycloak account with the identity provider. This authenticator is `disabled`, in my case, as i do not want users to confirm linking by email.
  - Add **Review Profile** to let end users to confirm their profile information. This authenticator is `disabled`, in my case, as i do not want users to confirm their profile information.

Apply this flow to the new IdP created in [Step 2](#2-set-up-the-identity-provider-in-keycloak) under **First Login Flow** settings.

---

### 4. 🌐 Modify the Browser Authentication Flow for Seamless SSO

The purpose is to **redirect users directly to the IdP login page** instead of showing Keycloak’s login screen.

- Go to **Authentication > Flows > Browser**
- Click **Copy** to duplicate the `Browser` flow and name it as `Browser - Auto Redirect`.
- Edit the new flow:
  - Add the **Identity Provider Redirector** option to redirect users to the IdP's SSO page.
  - Set **Default Identity Provider** to the alias of the IdP created in [Step 2](#2-set-up-the-identity-provider-in-keycloak). 

- Set the new flow as default browser flow in **Bindings > Browser Flow**.

This ensures that unauthenticated users are automatically redirected to the external IdP.

---

### 5. ⚙️ Add Apache HTTPD Rewrite Rule

The whole purpose of this step is to accept SAML assertion POSTs and redirect them internally to a valid OIDC client, using a rewrite rule.

Use the below Apache HTTPD rewrite rule to achieve this:

```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} POST
RewriteRule ^/sso/login$ https://oidc-client-a.example.com/ [R=302,L]
```

---

## Final Verification

- Trigger login via the IdP’s app launcher.
- Keycloak accepts SAML and redirects to OIDC client.
- Verify user creation or account linking.
- Confirm logout flows work as expected.

---

## ✅ Conclusion

Keycloak makes it easy to bridge SAML-based identity systems with modern OIDC clients. With a few configuration steps; and by combining SAML proxying, custom authentication flows, and a smart reverse proxy setup you can enable a seamless, secure IdP-initiated SSO experience for your apps.

---
