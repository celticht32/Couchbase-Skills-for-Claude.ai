# Capella credentials

## Two credential types — don't confuse them

| Type | Where created | What it's for | In application code? |
|---|---|---|---|
| **Capella UI credentials** | cloud.couchbase.com account | Managing Capella (clusters, billing, networking) | Never |
| **Database credentials** | Clusters → Connect → Database Access | Application connections to the cluster | Yes |

Your Capella UI email + password authenticates you to the Capella management plane. It has no access to the cluster's data plane. Applications authenticate with database credentials, which are scoped to specific buckets and roles.

## Creating database credentials

**Clusters → your cluster → Connect → Database Access → Create Credentials**

Fields:
- **Username:** any string. Convention: `<service-name>-<env>` (e.g. `orders-service-prod`)
- **Password:** auto-generated strong password, or provide your own (min 8 chars, mix of upper/lower/digit/special)
- **Bucket access:** select which buckets and what permissions (Read/Write, Read Only, or custom)

Generate the password once and store it in your secrets manager (Vault, AWS Secrets Manager, etc.). Capella won't show it again.

## Scoping credentials

Give each service its own credentials with minimum necessary access:

| Service | Buckets | Access |
|---|---|---|
| `orders-service-prod` | orders | Read/Write |
| `reporting-service-prod` | orders, inventory | Read Only |
| `analytics-pipeline-prod` | * (all) | Read Only |

Never use a single set of credentials shared across all services. If one service is compromised, all data is exposed.

## Rotating credentials

1. Create new database credentials with the same bucket/role configuration
2. Update your application to use the new credentials (rolling deploy or secrets manager update)
3. Verify the application is running with the new credentials
4. Delete the old credentials

There's no in-place password rotation — delete and recreate. Plan for this in your rotation procedure.

## Capella API keys (for automation)

For automated cluster management (Terraform, CI/CD, the `cb-analytics-capella` MCP tools):

**Organization → API Keys → Create API Key**

API keys have organization-level roles:
- `organizationOwner` — full management access
- `projectCreator` — can create projects
- `organizationMember` — read-only org access

API keys are separate from database credentials. They're for the management plane (create/delete clusters, manage backups) not for data access.

Store API keys in a secrets manager. Rotate annually at minimum.

## Connection string reference

```
# Standard Capella connection
couchbases://cb.<cluster-id>.cloud.couchbase.com

# With PrivateLink (private endpoint hostname varies by cloud/region)
couchbases://cb.<cluster-id>.private.cloud.couchbase.com
```

The cluster ID is visible in the URL when you're on the cluster page in the Capella UI, and in the cluster details via the API.

## SDK credential configuration

```python
# Python
PasswordAuthenticator("db-username", "db-password")

# Java
PasswordAuthenticator.create("db-username", "db-password")

# Node.js
new PasswordAuthenticator("db-username", "db-password")
```

Always load credentials from environment variables or a secrets manager — never hardcode them:

```python
import os
auth = PasswordAuthenticator(
    os.environ["CB_USERNAME"],
    os.environ["CB_PASSWORD"]
)
```
