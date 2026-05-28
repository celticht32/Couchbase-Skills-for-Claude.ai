# RBAC design

## Couchbase RBAC model

Couchbase uses role-based access control with two assignment mechanisms:
- **Direct role assignment:** roles attached directly to a user
- **Group membership:** user belongs to a group; group has roles

Use groups for anything beyond one-off service accounts. Groups make auditing and rotation much easier.

## Core roles for common use cases

**Application service accounts (read-write):**
```
data_reader[bucket]
data_writer[bucket]
query_select[bucket]
query_insert[bucket]
query_update[bucket]
query_delete[bucket]
```
Scope to the specific bucket(s) the service needs. Never grant `*` (all buckets) unless the service genuinely needs all buckets.

**Read-only analytics / reporting:**
```
data_reader[bucket]
query_select[bucket]
analytics_reader[bucket]
```

**Index management (CI/CD pipelines):**
```
query_manage_index[bucket]
```
Does not grant data access — an index-management service account can create/drop indexes without being able to read document contents.

**DBA / ops team:**
```
cluster_admin
```
Full cluster management, no data access. Use for operational tasks.

**Full data access (break-glass only):**
```
data_reader[*]
data_writer[*]
query_select[*]
```
Restrict to a named individual, require MFA, and log all access.

## Least-privilege design process

1. **List the operations the service performs.** SELECT on these collections. INSERT/UPDATE on those. Never DELETE.
2. **Map operations to the minimum roles.** Use `admin_user_list_roles` to browse available roles and their descriptions.
3. **Scope to specific buckets and scopes.** `query_select[orders]` not `query_select[*]`.
4. **Create a group**, assign the roles to the group, add the service account user to the group.
5. **Verify** with `admin_user_check_permissions` — confirm the service account has exactly what it needs and nothing more.

## Group structure for teams

```
Groups:
  app-orders-rw       → data_reader[orders], data_writer[orders], query_select[orders], query_insert[orders], query_update[orders]
  app-orders-ro       → data_reader[orders], query_select[orders]
  analytics-team      → analytics_reader[*], query_select[*], data_reader[*]
  dba-ops             → cluster_admin
  index-deploy        → query_manage_index[orders], query_manage_index[inventory]
  backup-agent        → backup_admin

Users:
  orders-service      → group: app-orders-rw
  reporting-user      → group: analytics-team
  alice (DBA)         → group: dba-ops
  ci-pipeline         → group: index-deploy
  cbbackupmgr-agent   → group: backup-agent
```

## Password policy

```python
admin_cluster_set_password_policy(
    minLength=12,
    enforceUppercase=True,
    enforceLowercase=True,
    enforceDigits=True,
    enforceSpecialChars=True,
    expiryDays=90,        # 0 = no expiry
    cluster="prod"
)
```

For service accounts: use generated secrets (32+ random characters), never rotate on a schedule (rotate only on compromise), store in a secrets manager (Vault, AWS Secrets Manager, 1Password).

For human accounts: enforce complexity + 90-day rotation. Use LDAP/SSO (external auth) to inherit your organization's existing password policy and MFA requirements.

## Account lockout (8.x)

```python
admin_user_update_lockout_settings(
    enabled=True,
    threshold=5,        # failed attempts before lockout
    duration=300,       # lockout duration in seconds (5 min)
    cluster="prod"
)
```

Check/unlock a locked user:
```python
admin_user_get_user(domain="local", username="alice", cluster="prod")
# Check "locked" field

admin_user_unlock(domain="local", username="alice", cluster="prod")
```

Lockout applies to local users only — external (LDAP/SAML) user lockout is managed in the external directory.

## RBAC audit

Periodically review all users and their effective permissions:

1. `admin_user_list_users` — enumerate all local users
2. For each user, check their direct roles + group memberships
3. `admin_user_check_permissions` — verify effective permissions match what's expected
4. Remove stale service accounts (departed employees, decommissioned services)
5. Review group memberships — groups tend to accumulate members over time

Set a calendar reminder: quarterly RBAC audit is the minimum for SOC2 compliance.
