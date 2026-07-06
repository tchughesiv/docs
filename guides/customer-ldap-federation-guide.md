# Integrating Enterprise LDAP with OSAC via External Identity Provider

**Last Updated**: 2026-07-06
**Audience**: Cloud administrators, tenant operators
**Status**: Draft

---

## Overview

OSAC uses a central Keycloak instance with an internal `osac` realm for
authentication and authorization. This realm is managed by the OSAC platform
and is **not available for direct tenant configuration**.

If your organization uses an LDAP directory (389 Directory Server, Active
Directory, OpenLDAP, Red Hat Directory Server) for user management, you can
integrate it with OSAC by placing an **OIDC-capable Identity Provider (IdP)**
in front of your LDAP and federating it with the OSAC realm.

This guide covers:

1. Why direct LDAP configuration in the OSAC realm is not supported
2. How to set up your own IdP with LDAP federation
3. How to register your IdP with OSAC for tenant-scoped authentication

### What This Guide Does NOT Cover

- Configuring the OSAC realm itself (managed by the platform)
- The internal LDAP integration architecture (see
  [osac-ldap-keycloak-setup-guide.md](osac-ldap-keycloak-setup-guide.md))

---

## Why Direct LDAP in the OSAC Realm Is Not Supported

The `osac` realm is the platform's internal identity boundary. It manages
service accounts, client credentials, and organization membership for all
tenants. Allowing tenants to configure LDAP federation directly in this
realm would:

1. **Break tenant isolation** — LDAP federation is realm-wide; all
   organizations would share the same LDAP provider, and username collisions
   across tenants could cause authentication failures or credential leakage
   to the wrong directory.

2. **Expose the internal realm** — Tenants would need admin access to the
   OSAC Keycloak instance, which is not operationally acceptable.

3. **Prevent password boundary enforcement** — In a multi-tenant environment,
   user passwords must never leave the tenant's own infrastructure. With
   direct LDAP federation, Keycloak sends the password to the LDAP server
   during every authentication — if multiple LDAPs are configured in one
   realm, passwords could be sent to the wrong tenant's directory.

Instead, OSAC uses **Keycloak Organizations with OIDC IdP brokering**:
each tenant provides (or deploys) their own OIDC Identity Provider, and the
OSAC realm brokers authentication to it. Passwords never enter the OSAC
platform — only signed tokens are exchanged.

```
┌──────────────────────────────────────────────────────────────┐
│                    OSAC Platform                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              OSAC Keycloak (osac realm)                │   │
│  │              Organizations: ENABLED                    │   │
│  │                                                       │   │
│  │  ┌─────────────┐           ┌─────────────┐           │   │
│  │  │ Org: Acme   │           │ Org: Globex │           │   │
│  │  │ IdP: oidc   │           │ IdP: oidc   │           │   │
│  │  └──────┬──────┘           └──────┬──────┘           │   │
│  └─────────┼─────────────────────────┼───────────────────┘   │
│            │ OIDC tokens only        │ OIDC tokens only       │
└────────────┼─────────────────────────┼────────────────────────┘
             │                         │
             ▼                         ▼
   ┌─────────────────┐       ┌─────────────────┐
   │  Acme's IdP     │       │  Globex's IdP   │
   │  (Keycloak /    │       │  (Azure AD /    │
   │   Okta / ADFS)  │       │   Okta / ...)   │
   │       │         │       │       │         │
   │       ▼         │       │       ▼         │
   │  Acme's LDAP    │       │  Globex's LDAP  │
   └─────────────────┘       └─────────────────┘

   Passwords stay here        Passwords stay here
```

---

## Prerequisites

Before starting, you need:

| Requirement | Description |
|-------------|-------------|
| LDAP directory | A running LDAP server (389-ds, Active Directory, OpenLDAP, RHDS) accessible from your IdP |
| OIDC-capable IdP | Keycloak, Azure AD/Entra ID, Okta, ADFS, PingFederate, Authentik, or any OIDC-compliant provider |
| Email domain | A verified email domain for your organization (e.g., `acme-corp.example.com`) |
| Network access | Your IdP must be reachable via HTTPS from the OSAC Keycloak instance |
| TLS certificate | A valid TLS certificate on your IdP (self-signed is acceptable if the CA is provided to OSAC) |

**If you don't have an IdP yet**, see [Option B: Deploy a Keycloak Instance
as Your IdP](#option-b-deploy-a-keycloak-instance-as-your-idp) below.

---

## Option A: Register an Existing OIDC Identity Provider

If your organization already operates an OIDC-capable IdP (Azure AD, Okta,
ADFS, PingFederate, or your own Keycloak), you only need to:

1. Create an application/client registration in your IdP for OSAC
2. Provide the connection details to the OSAC administrator

### Step 1: Create an Application Registration in Your IdP

Create a new OIDC client/application in your IdP with the following settings:

| Setting | Value |
|---------|-------|
| **Client type** | Confidential (with client secret) |
| **Grant type** | Authorization Code |
| **Redirect URI** | `https://<osac-keycloak-host>/realms/osac/broker/<idp-alias>/endpoint` |
| **Scopes** | `openid`, `email`, `profile` |
| **Token format** | JWT (signed, not opaque) |

> **Note:** The exact redirect URI will be provided by the OSAC administrator
> after your organization is registered. The `<idp-alias>` is the Identity
> Provider alias assigned by the OSAC administrator (e.g., `acme-idp`).

Record the following values — you will provide them to the OSAC administrator:

| Value | Example |
|-------|---------|
| **OIDC Discovery URL** | `https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration` |
| **Client ID** | `a1b2c3d4-e5f6-...` |
| **Client Secret** | `secret-value` |
| **Email domain(s)** | `acme-corp.example.com` |

### Step 2: Ensure LDAP Federation in Your IdP

Your IdP must be configured to authenticate users against your LDAP directory.
This is specific to your IdP:

| IdP | LDAP Integration Method |
|-----|------------------------|
| **Azure AD / Entra ID** | Azure AD Connect or Cloud Sync from on-premises AD |
| **Okta** | Okta AD Agent or LDAP Agent |
| **Keycloak** | User Storage Federation (LDAP provider) |
| **ADFS** | Native Active Directory integration |
| **PingFederate** | LDAP Data Store |
| **Authentik** | LDAP Source |

The key requirement is that users authenticating through your IdP are validated
against your LDAP directory, and the resulting OIDC tokens include the user's
email address as a claim.

### Step 3: Provide Details to the OSAC Administrator

Send the following to the OSAC platform administrator:

```yaml
organization_name: "acme-corp"
email_domains:
  - "acme-corp.example.com"
idp_type: "oidc"
oidc_discovery_url: "https://your-idp.example.com/.well-known/openid-configuration"
client_id: "<your-client-id>"
client_secret: "<your-client-secret>"
tls_ca_certificate: |
  -----BEGIN CERTIFICATE-----
  <only needed if your IdP uses a private/self-signed CA>
  -----END CERTIFICATE-----
```

The OSAC administrator will:

1. Create an Organization for your tenant in the `osac` realm
2. Register your IdP as an OIDC Identity Provider
3. Link the IdP to your organization with domain-based routing
4. Provide you the exact redirect URI to configure in your IdP

### What Happens at Login

After registration, users with your email domain are automatically routed
to your IdP:

1. User opens an OSAC application (CLI, Rancher, ArgoCD, etc.)
2. Redirected to the OSAC Keycloak login page — **email field only**
3. User enters their email (e.g., `alice@acme-corp.example.com`)
4. Keycloak matches the domain to your organization and redirects to your IdP
5. User authenticates against your IdP (which validates against your LDAP)
6. Redirected back to OSAC with a JWT containing `"organization": ["acme-corp"]`
7. User is logged in with tenant-scoped access

Your password is never seen by the OSAC platform — only a signed token
is exchanged.

---

## Option B: Deploy a Keycloak Instance as Your IdP

> **Note:** This option has not been validated end-to-end. The steps below
> are based on Keycloak documentation and the proven architecture from
> Option A. If you follow this path, please report any issues to the OSAC
> team so we can refine this guide.

If your organization only has a bare LDAP directory without an OIDC-capable
IdP, you can deploy a lightweight Keycloak instance to bridge the gap.

This Keycloak instance is **yours** — you manage it, and it runs in your
infrastructure (or in a namespace you control). It serves as an OIDC frontend
for your LDAP directory.

### Step 1: Deploy Keycloak

Deploy a Keycloak instance using one of these methods:

**On OpenShift (recommended):**

```bash
# Install the Keycloak operator (RHBK)
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhbk-operator
  namespace: my-tenant-keycloak
spec:
  channel: stable-v26
  name: rhbk-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
EOF

# Wait for operator, approve install plan, then create Keycloak instance
# (see Keycloak Operator documentation for full steps)
```

**Container (podman/docker):**

```bash
podman run -d --name tenant-keycloak \
  -p 8443:8443 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=<admin-password> \
  -e KC_HOSTNAME=keycloak.acme-corp.example.com \
  -e KC_HTTPS_CERTIFICATE_FILE=/opt/keycloak/conf/tls.crt \
  -e KC_HTTPS_CERTIFICATE_KEY_FILE=/opt/keycloak/conf/tls.key \
  -v /path/to/certs:/opt/keycloak/conf:Z \
  quay.io/keycloak/keycloak:26.2 start
```

> **Important:** For production use, configure an external database
> (PostgreSQL recommended), TLS certificates, and proper hostname settings.

### Step 2: Create a Realm

Create a realm for your organization. This realm is separate from the OSAC
`osac` realm — it is your own identity boundary.

```bash
KC_URL="https://keycloak.acme-corp.example.com"

KC_TOKEN=$(curl -ks "${KC_URL}/realms/master/protocol/openid-connect/token" \
    -d 'client_id=admin-cli' \
    -d 'grant_type=password' \
    -d 'username=admin' \
    -d "password=<admin-password>" | jq -r '.access_token')

curl -sk -X POST "${KC_URL}/admin/realms" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "realm": "acme-idp",
  "enabled": true,
  "loginWithEmailAllowed": true
}'
```

### Step 3: Configure LDAP Federation

Register your LDAP server as a User Storage Provider in your realm:

```bash
curl -sk -X POST "${KC_URL}/admin/realms/acme-idp/components" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "name": "acme-ldap",
  "providerId": "ldap",
  "providerType": "org.keycloak.storage.UserStorageProvider",
  "config": {
    "editMode":              ["READ_ONLY"],
    "vendor":                ["rhds"],
    "connectionUrl":         ["ldap://<your-ldap-host>:389"],
    "bindDn":                ["cn=Directory Manager"],
    "bindCredential":        ["<bind-password>"],
    "usersDn":               ["ou=users,dc=acme,dc=example,dc=com"],
    "usernameLDAPAttribute": ["uid"],
    "rdnLDAPAttribute":      ["uid"],
    "uuidLDAPAttribute":     ["nsuniqueid"],
    "userObjectClasses":     ["inetOrgPerson"],
    "searchScope":           ["2"],
    "importEnabled":         ["true"],
    "syncRegistrations":     ["false"],
    "trustEmail":            ["true"],
    "enabled":               ["true"],
    "priority":              ["0"]
  }
}'
```

**Adapt the following values for your environment:**

| Setting | 389-ds / RHDS | Active Directory | OpenLDAP |
|---------|--------------|------------------|----------|
| `vendor` | `rhds` | `ad` | `other` |
| `connectionUrl` | `ldap://host:389` | `ldap://dc.corp:389` | `ldap://host:389` |
| `bindDn` | `cn=Directory Manager` | `CN=svc-keycloak,OU=Service,DC=corp` | `cn=admin,dc=example,dc=com` |
| `usersDn` | `ou=users,dc=...` | `OU=Users,DC=corp,DC=example,DC=com` | `ou=people,dc=...` |
| `usernameLDAPAttribute` | `uid` | `sAMAccountName` | `uid` |
| `uuidLDAPAttribute` | `nsuniqueid` | `objectGUID` | `entryUUID` |
| `userObjectClasses` | `inetOrgPerson` | `person, organizationalPerson, user` | `inetOrgPerson` |

> **Production note:** Use `ldaps://` (port 636) with TLS certificates for
> production deployments. Cleartext LDAP (port 389) should only be used in
> development/lab environments.

### Step 4: Create the OSAC Broker Client

Create a confidential OIDC client that the OSAC Keycloak will use to
authenticate users through your realm:

```bash
curl -sk -X POST "${KC_URL}/admin/realms/acme-idp/clients" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "clientId": "osac-broker",
  "enabled": true,
  "protocol": "openid-connect",
  "publicClient": false,
  "secret": "<generate-a-strong-secret>",
  "standardFlowEnabled": true,
  "directAccessGrantsEnabled": false,
  "redirectUris": [
    "https://<osac-keycloak-host>/realms/osac/broker/<idp-alias>/endpoint*"
  ],
  "defaultClientScopes": ["openid", "email", "profile"]
}'
```

> **Note:** The `redirectUris` value must match the broker endpoint that the
> OSAC administrator provides. Replace `<idp-alias>` with the Identity Provider
> alias assigned by the OSAC administrator (e.g., `acme-idp`).

### Step 5: Verify LDAP Authentication

The `osac-broker` client uses the Authorization Code flow and does not
support direct password grants. To verify that LDAP authentication works,
create a temporary test client with direct access grants enabled:

```bash
# Create a temporary test client
curl -sk -X POST "${KC_URL}/admin/realms/acme-idp/clients" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "clientId": "ldap-test",
  "enabled": true,
  "publicClient": true,
  "directAccessGrantsEnabled": true,
  "standardFlowEnabled": false
}'

# Test LDAP authentication via the test client
curl -ks "${KC_URL}/realms/acme-idp/protocol/openid-connect/token" \
    -d 'client_id=ldap-test' \
    -d 'grant_type=password' \
    -d 'username=alice' \
    -d 'password=<alice-ldap-password>' | jq .
```

Expected response:

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 300
}
```

If you receive `"error": "invalid_grant"`, verify:
- LDAP connectivity from the Keycloak pod/container
- Bind DN credentials
- User DN and search scope
- User exists in LDAP with the correct `objectClass`

After verification, delete the test client:

```bash
TEST_CLIENT_ID=$(curl -ks -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/acme-idp/clients?clientId=ldap-test" | jq -r '.[0].id')

curl -sk -X DELETE "${KC_URL}/admin/realms/acme-idp/clients/${TEST_CLIENT_ID}" \
    -H "Authorization: Bearer ${KC_TOKEN}"
```

### Step 6: Provide Details to the OSAC Administrator

Once your Keycloak + LDAP setup is working, provide the OSAC administrator
with the details listed in [Option A, Step 3](#step-3-provide-details-to-the-osac-administrator):

```yaml
organization_name: "acme-corp"
email_domains:
  - "acme-corp.example.com"
idp_type: "oidc"
oidc_discovery_url: "https://keycloak.acme-corp.example.com/realms/acme-idp/.well-known/openid-configuration"
client_id: "osac-broker"
client_secret: "<your-secret>"
tls_ca_certificate: |
  -----BEGIN CERTIFICATE-----
  <your CA certificate if using private/self-signed TLS>
  -----END CERTIFICATE-----
```

---

## OSAC Administrator: Registering a Tenant's IdP

This section is for OSAC platform administrators who receive tenant IdP
details and need to register them in the `osac` realm.

### Step 1: Create the Organization

```bash
KC_URL="https://<osac-keycloak-host>"
KC_TOKEN="<admin-token>"

curl -sk -X POST "${KC_URL}/admin/realms/osac/organizations" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "name": "acme-corp",
  "domains": [{"name": "acme-corp.example.com", "verified": true}]
}'
```

### Step 2: Register the IdP

```bash
curl -sk -X POST "${KC_URL}/admin/realms/osac/identity-provider/instances" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "alias": "acme-idp",
  "displayName": "Acme Corp",
  "providerId": "oidc",
  "enabled": true,
  "hideOnLogin": true,
  "config": {
    "issuer": "https://keycloak.acme-corp.example.com/realms/acme-idp",
    "authorizationUrl": "https://keycloak.acme-corp.example.com/realms/acme-idp/protocol/openid-connect/auth",
    "tokenUrl": "https://keycloak.acme-corp.example.com/realms/acme-idp/protocol/openid-connect/token",
    "clientId": "osac-broker",
    "clientSecret": "<tenant-provided-secret>",
    "defaultScope": "openid email profile",
    "syncMode": "FORCE"
  }
}'
```

> **Important:** Always set `hideOnLogin: true` to prevent tenant names from
> appearing on the login page. Routing is handled by email domain matching,
> not by visible IdP buttons.

### Step 3: Link the IdP to the Organization

```bash
ORG_ID=$(curl -sk -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac/organizations?search=acme-corp" | jq -r '.[0].id')

curl -sk -X POST "${KC_URL}/admin/realms/osac/organizations/${ORG_ID}/identity-providers" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"alias": "acme-idp"}'
```

### Step 4: Verify the Integration

1. Open an OSAC application (or the Keycloak account console)
2. Enter a test user's email from the tenant's domain
3. Confirm you are redirected to the tenant's IdP login page
4. Authenticate and verify the JWT contains the `organization` claim:

```json
{
  "preferred_username": "alice",
  "email": "alice@acme-corp.example.com",
  "organization": ["acme-corp"]
}
```

### Ensure the Browser Flow Uses Organization Routing

The `osac` realm must use a browser flow that supports Organization
Identity-First Login. If not already configured:

```bash
# Verify the browser flow
curl -sk -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac" | jq '.browserFlow'
```

The flow should include the `organization` authenticator (Identity-First
Login). See the [internal setup guide](osac-ldap-keycloak-setup-guide.md#7-configure-the-browser-flow)
for flow configuration details.

---

## TLS Requirements

### Tenant IdP → OSAC Keycloak

The OSAC Keycloak instance must trust your IdP's TLS certificate to perform
the back-channel token exchange. If your IdP uses a certificate signed by a
well-known public CA, no additional configuration is needed.

If your IdP uses a private or self-signed CA:

1. Provide the CA certificate to the OSAC administrator
2. The OSAC administrator adds it to Keycloak's trust store via
   `KC_TRUSTSTORE_PATHS`

### Tenant IdP → LDAP

Your IdP's connection to your LDAP server should use LDAPS (port 636) in
production. This is entirely within your infrastructure and does not
involve the OSAC platform.

---

## Supported Identity Providers

Any OIDC-compliant IdP should work with OSAC. **None of the options below
have been tested end-to-end with OSAC yet.** They are listed as supported
based on each product's documented OIDC and LDAP capabilities.

### IdPs with Native LDAP Authentication

These IdPs can authenticate users directly against an LDAP directory and
issue OIDC tokens:

| IdP | LDAP Authentication Method | Reference |
|-----|---------------------------|-----------|
| **Keycloak** (community or RHBK) | User Storage Federation | [RHBK User Federation](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html/server_administration_guide/user-storage-federation) |
| **Okta** | Okta AD Agent / LDAP Agent | [Okta LDAP Agent](https://help.okta.com/en-us/content/topics/directory/ldap-agent-main.htm) |
| **ADFS** | Native Active Directory | [ADFS Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/ad-fs-overview) |
| **PingFederate** | LDAP Data Store | [PingFederate LDAP](https://docs.pingidentity.com/pingfederate/latest/administrators_reference_guide/pf_c_ldapDataStores.html) |
| **Authentik** (open source) | LDAP Source | [Authentik LDAP](https://docs.goauthentik.io/docs/sources/ldap/) |

### IdPs with Directory Sync (Not Direct LDAP Auth)

These platforms synchronize user data from on-premises directories but do
not perform LDAP bind authentication directly. They can still serve as
OIDC IdPs for OSAC after syncing your directory:

| IdP | Sync Method | Reference |
|-----|------------|-----------|
| **Azure AD / Entra ID** | Azure AD Connect (syncs AD users to cloud) | [Azure AD Connect](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect) |
| **Google Workspace** | Google Cloud Directory Sync | [GCDS Overview](https://support.google.com/a/answer/106368) |

---

## Frequently Asked Questions

### Can I configure LDAP directly in the OSAC Keycloak realm?

No. The `osac` realm is managed by the OSAC platform and does not support
direct tenant modifications. Place an OIDC IdP in front of your LDAP and
register it with OSAC as described in this guide.

### What if I have multiple email domains?

You can register multiple domains for a single organization. All domains
will route to the same IdP:

```json
{
  "name": "acme-corp",
  "domains": [
    {"name": "acme-corp.example.com", "verified": true},
    {"name": "acme.io", "verified": true}
  ]
}
```

### What happens if a user's email domain is not registered?

Authentication will fail at the domain-matching step. The user will see
an error on the Keycloak login page. Contact the OSAC administrator to
register the domain.

### Can two tenants share an LDAP server?

Yes, as long as each tenant has its own IdP (or its own realm in a shared
Keycloak instance). The isolation is at the IdP level, not the LDAP level.
Each IdP can point to the same LDAP server with different base DNs or
search filters.

### Do users need to change their workflow?

No. Users authenticate with the same credentials they use today. The only
visible change is that the login page shows an email field first, then
redirects to their organization's IdP for password entry.

### What about CLI access?

CLI tools (e.g., `osac login`) support the Device Authorization Flow:

1. CLI displays a URL and a one-time code
2. User opens the URL in a browser and enters the code
3. Browser redirects to the tenant's IdP for authentication
4. CLI receives a token after the browser flow completes

The user's password is entered in the browser and sent only to the
tenant's IdP — the CLI and the OSAC platform never see it.

---

## References

- [Keycloak Organizations — Managing IdPs](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html/server_administration_guide/managing_organizations)
- [Keycloak User Storage Federation](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html/server_administration_guide/user-storage-federation)
- [Keycloak OIDC Identity Brokering](https://www.keycloak.org/docs/latest/server_admin/#_identity_broker_oidc)
- [Internal: LDAP + Keycloak Setup Guide](osac-ldap-keycloak-setup-guide.md) (engineering reference)
- [Internal: Multi-Tenant LDAP Isolation Research](multi-tenant-ldap-isolation-research.md)
