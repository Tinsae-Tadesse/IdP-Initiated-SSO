
# Configuring IdP-Initiated SSO in Keycloak with SAML 2.0 for OIDC Clients

Single Sign-On (SSO) is a cornerstone of modern identity and access management strategies. In scenarios where an organization uses a centralized Identity Provider (IdP) and still wants to delegate application-level authentication through OpenID Connect (OIDC), Keycloak can act as an **identity broker**—bridging SAML-based identity assertions from an external IdP to OIDC-based clients.

This guide outlines how to configure **IdP-Initiated SSO using SAML 2.0**, with Keycloak acting as a SAML proxy that federates authentication to **two OIDC clients** behind the scenes.

---

## Architecture Overview
![SSO Architecture Diagram](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/Architecture.png?raw=true)

---

## Step-by-Step Implementation

### 1. Create a SAML Proxy Client in Keycloak

In Keycloak, a **SAML client** acts as the Assertion Consumer Service (ACS) endpoint. This client will receive and validate SAML assertions from your external IdP.

- Go to **Clients > Create**.
- **Client ID**: `saml-proxy-client` (or any name).
- **Client Protocol**: `saml`.
- **Client SAML Endpoint**: Use Keycloak’s default ACS endpoint:
  ```
  https://<keycloak-domain>/auth/realms/<realm-name>/broker/<idp-alias>/endpoint
  ```
- **Force POST Binding**: Enabled.
- **Allow IDP-Initiated SSO**: Ensure Keycloak accepts unsolicited assertions.

Save and configure signature and encryption settings as per your IdP requirements.

---

### 2. Set Up the Identity Provider in Keycloak

This step allows Keycloak to **trust the external IdP** and receive SAML assertions.

- Go to **Identity Providers > Add provider > SAML**.
- **Alias**: `external-idp`
- **Import from URL or XML**: Use the IdP’s metadata URL or upload the XML.
- **Single Sign-On Service URL**: Auto-filled from metadata.
- **Single Logout Service URL**: Optional, but ideal for SLO.
- **Want AuthnRequests Signed**: Optional; depends on IdP configuration.

Adjust mappers to link SAML attributes like `email`, `username`, etc.

---

### 3. Customize the First Login Flow

To automatically **provision new users** or match existing users when logging in via the IdP:

- Go to **Authentication > Flows > First Broker Login**.
- Click **Copy** to duplicate the default flow.
- Edit the new flow:
  - Add **Create User If Unique**
  - Add **Review Profile** (optional)
  - Add **Automatically Set Existing User**

Apply this flow to your IdP under **First Login Flow** settings.

---

### 4. Modify the Browser Authentication Flow

To **redirect users to the IdP login page**:

- Go to **Authentication > Flows > Browser**
- Copy the default flow and modify it:
  - Replace **Username Password Form** with **Identity Provider Redirector**

Set the new flow as default in **Bindings > Browser Flow**.

---

### 5. Apache HTTPD Rewrite Rule

Handle POSTs from IdP-Initiated SSO by rewriting to the OIDC client endpoint:

```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} POST
RewriteRule ^/auth/realms/your-realm/broker/external-idp/endpoint$ \
    /auth/realms/your-realm/protocol/openid-connect/auth \
    [R=302,L]
```

Ensure the rule is context-aware to prevent redirect loops.

---

## Final Verification

- Trigger login via the IdP’s app launcher.
- Keycloak accepts SAML and redirects to OIDC client.
- Verify user creation or account linking.
- Confirm logout flows work as expected.

---

## Conclusion

Keycloak makes it easy to bridge SAML-based identity systems with modern OIDC clients. With a few configuration steps, you can enable a seamless, secure IdP-initiated SSO experience for your apps.

---
