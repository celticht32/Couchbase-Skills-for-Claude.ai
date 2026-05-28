# External authentication

## Why use external auth?

Local Couchbase users are fine for service accounts. For human operators, external auth (LDAP/AD/SAML) provides:
- Centralized identity management (onboard/offboard in one place)
- MFA enforcement at the IdP layer
- Consistent password policy across all systems
- Audit trail in the identity provider

## LDAP / Active Directory

Couchbase supports LDAP bind for authentication and LDAP group-to-Couchbase-group mapping.

**Configuration (via UI or REST — not currently exposed in MCP-Couchbase tools):**

Key settings:
- **LDAP host/port:** your directory server. Port 389 (plaintext) or 636 (LDAPS). Always use LDAPS in production.
- **Bind DN + password:** the service account Couchbase uses to query the directory (read-only LDAP bind account).
- **User search template:** how to find the user entry for a given username. Example: `uid=%u,ou=people,dc=example,dc=com` (static DN) or a search filter `(&(objectClass=person)(uid=%u))`.
- **Group search:** LDAP groups that map to Couchbase groups.

**Group mapping:** map an LDAP group DN to a Couchbase group name. Members of the LDAP group automatically get the Couchbase group's roles without creating individual Couchbase users.

```
LDAP group: cn=couchbase-dba,ou=groups,dc=example,dc=com
  ↓ maps to ↓
Couchbase group: dba-ops (roles: cluster_admin)
```

Users authenticate with their LDAP credentials. Couchbase validates against LDAP; if successful, looks up the user's LDAP group memberships and applies the corresponding Couchbase group roles.

**Testing LDAP configuration:**
Use the Couchbase UI's "Test Authentication" feature before relying on LDAP in production. Common failure modes:
- Wrong bind DN (LDAP returns "invalid credentials" for the service account)
- User search template doesn't match the directory's schema
- LDAPS certificate not trusted by the Couchbase node

## SAML (Couchbase 7.2+)

SAML 2.0 support for the Couchbase UI. Allows SSO from an IdP (Okta, Azure AD, PingFederate, etc.) for web console access.

SAML is UI-only — it doesn't apply to SDK connections or REST API calls. Use LDAP for those.

**Setup overview:**
1. Register Couchbase as a SAML service provider in your IdP. The SP entity ID and ACS URL come from the Couchbase SAML configuration UI.
2. Export IdP metadata XML from your IdP.
3. Import IdP metadata into Couchbase (Security → SAML).
4. Map SAML assertion attributes (group membership claim) to Couchbase groups.

**Common IdP attribute mappings:**
- Okta: use a custom attribute `couchbaseGroups` populated from Okta group assignments
- Azure AD: use group claims with the group object IDs mapped to Couchbase group names

## PAM (Linux PAM)

PAM authentication allows Couchbase to delegate authentication to the Linux host's PAM stack. Useful for:
- Integrating with SSSD/Kerberos/AD on Linux nodes without a full LDAP setup
- Maintaining a single Linux user database for both OS and Couchbase access

PAM applies to local Linux user accounts only. The Couchbase user record must exist (domain: `external`) with the same username as the Linux account. Authentication is delegated to PAM at login time.

Less common than LDAP for most enterprise deployments — use LDAP if your org has Active Directory.

## External users in Couchbase

Regardless of auth method (LDAP, SAML, PAM), external users in Couchbase:
- Exist in the `external` domain
- Have no stored password in Couchbase (authentication is delegated)
- Still need role assignments (direct or via group) in Couchbase
- Are created with `admin_user_upsert_user(domain="external", ...)` — no password field

```python
admin_user_upsert_user(
    domain="external",
    username="alice",           # must match the LDAP uid / SAML nameId / PAM username
    roles="",                   # roles come from group membership, not direct assignment
    full_name="Alice Smith",
    cluster="prod"
)
```

Then add Alice to a Couchbase group (which has the appropriate roles), or assign roles directly.
