
# Configuring IdP-Initiated SSO in Keycloak with SAML 2.0 for OIDC Clients

## Overview
Single Sign-On (SSO) is a cornerstone of modern identity and access management strategies. In scenarios where an organization uses a centralized Identity Provider (IdP) and still wants to delegate application-level authentication through OpenID Connect (OIDC), Keycloak can act as an **identity broker**—bridging SAML-based identity assertions from an external IdP to OIDC-based clients.

This guide outlines how to configure **IdP-Initiated SSO using SAML 2.0**, with Keycloak acting as a SAML proxy that federates authentication to **two OIDC clients** behind the scenes.

---

## Implementation Architecture
![SSO Architecture Diagram](https://github.com/Tinsae-Tadesse/IdP-Initiated-SSO/blob/main/Architecture.png?raw=true)

---

## Implementation Steps

### 1. Create a SAML Proxy Client in Keycloak

In Keycloak, a **SAML client** acts as the Assertion Consumer Service (ACS) endpoint. This client will receive and validate SAML assertions from your external IdP.

- Go to **Clients > Create**.
- **Client ID**: `saml-proxy-client` (or any name).
- **Client Protocol**: `saml`.
- **Force POST Binding**: Enabled.

Save and configure signature and encryption settings as per your IdP requirements.

**With these settings, Keycloak’s default ACS endpoint will handle SAML assertions for this client.**
Keycloak's default ACS endpoint is at:
```
https://<keycloak-domain>/auth/realms/<realm-name>/broker/<idp-alias>/endpoint
```
For example, an actual ACS endpoint will look like:
```
https://example.com/auth/realms/DEFAULT/broker/external-idp/endpoint
```

---

### 2. Set Up the Identity Provider in Keycloak

This step allows Keycloak to **trust the external IdP** and receive SAML assertions.

- Go to **Identity Providers > Add provider > SAML v2.0**.
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
  - Add **Handle Existing Account**
  - Add **Review Profile** (optional), in my case i removed this flow since i don't want my users to make changes to their profile passed from the IdP 

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
RewriteRule ^/realms/<realm-name>/broker/<idp-alias>/endpoint$ \\
  /realms/<realm-name>/protocol/openid-connect/auth?client_id=my-oidc-client-app&response_type=code&scope=openid&redirect_uri=https://my-app.example.com/callback \\
  [R=302,L]
```
For example:
```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} POST
RewriteRule ^/realms/DEFAULT/broker/external-idp/endpoint$ \\
  /realms/DEFAULT/protocol/openid-connect/auth?client_id=my-oidc-client-app&response_type=code&scope=openid&redirect_uri=https://my-oidc-client-app.example.com/ui/ \\
  [R=302,L]
```

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
