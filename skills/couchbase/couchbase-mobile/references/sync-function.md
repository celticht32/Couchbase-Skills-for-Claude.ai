# Sync function

The sync function is a JavaScript function that runs on Sync Gateway / App Services for every document write. It determines:
- Which **channels** the document belongs to (who can read it)
- Which **users or roles** have read/write access to it
- Whether to **reject** the document entirely

## Basic structure

```javascript
function(doc, oldDoc, meta) {
    // doc     — the new document being written (or {_deleted: true} for deletes)
    // oldDoc  — the previous version (null on create)
    // meta    — metadata: {id, xattrs}

    // 1. Validate (throw to reject)
    if (!doc.type) {
        throw({forbidden: "Documents must have a type field"});
    }

    // 2. Route to channels
    channel("public");                          // all users
    channel("user." + doc.owner_id);           // owner only
    channel("region." + doc.region);           // regional channel

    // 3. Grant access (optional — usually done separately)
    access(doc.owner_id, "user." + doc.owner_id);
}
```

## Channel patterns

### Public channel
All authenticated users can read.
```javascript
channel("public");
```

### User-private channel
Only the owner sees their documents.
```javascript
channel("user." + doc.user_id);
access(doc.user_id, "user." + doc.user_id);   // grant user access to their channel
```

### Group/team channel
Members of a group share documents.
```javascript
if (doc.team_id) {
    channel("team." + doc.team_id);
    // Grant access to group members elsewhere (in a user profile doc or roles)
}
```

### Role-based channel
Admins see everything; users see their own.
```javascript
channel("user." + doc.user_id);   // owner
channel("admin");                  // all admins
requireRole("admin");              // only admins can write (optional)
```

### Tenant isolation
Multi-tenant SaaS: strict tenant separation.
```javascript
if (!doc.tenant_id) throw({forbidden: "tenant_id required"});
channel("tenant." + doc.tenant_id);
requireUser(doc.tenant_id + "::" + doc.owner);  // user must belong to this tenant
```

## Access control functions

| Function | What it does |
|---|---|
| `channel(name)` | Routes this document to the named channel |
| `access(user, channel)` | Grants a user access to a channel |
| `role(user, role)` | Assigns a role to a user |
| `requireUser(user)` | Rejects write if not from this user |
| `requireRole(role)` | Rejects write if user doesn't have this role |
| `requireAccess(channel)` | Rejects write if user doesn't have access to channel |
| `throw({forbidden: "msg"})` | Rejects write with a 403 and message |
| `throw({unauthorized: "msg"})` | Rejects write with a 401 |

## Document validation

Validate in the sync function to reject bad writes before they reach the database:

```javascript
function(doc, oldDoc, meta) {
    // Reject deletes of certain document types
    if (doc._deleted && oldDoc && oldDoc.type === "order") {
        throw({forbidden: "Orders cannot be deleted"});
    }

    // Require type field
    if (!doc._deleted && !doc.type) {
        throw({forbidden: "type field required"});
    }

    // Prevent field changes
    if (oldDoc && oldDoc.owner_id !== doc.owner_id) {
        throw({forbidden: "owner_id is immutable"});
    }

    // Route to channel after validation passes
    channel(doc.type + "." + doc.owner_id);
}
```

## Granting channel access

Access grants can be embedded in a document (e.g. a user profile doc):

```javascript
// When a user profile doc is written, grant the user access to their channel
function(doc, oldDoc, meta) {
    if (doc.type === "user_profile") {
        // Grant this user access to their personal channel
        access(doc.username, "user." + doc.username);

        // Grant access to team channels
        if (doc.teams) {
            doc.teams.forEach(function(team) {
                access(doc.username, "team." + team);
            });
        }

        channel("user." + doc.username);   // route the profile doc itself
    }
}
```

This pattern means: when you create or update a user profile, the sync function automatically grants the user access to the right channels. You don't need to call the admin API to manage access separately.

## Performance rules

- The sync function runs synchronously on every write. Keep it under 1ms.
- No I/O, no external calls, no complex computation.
- Avoid calling `access()` with wildcard or dynamically computed channels in tight loops — each `access()` call adds to the user's channel grant list, which grows the replication state.
- For large-scale systems (>1M users), keep channel names short — they're stored in user records and transmitted in replication messages.

## Capella App Services sync function

In Capella App Services, the sync function is configured per collection via the Capella UI or REST API, not a config file. The JavaScript syntax is identical to self-managed Sync Gateway.

```
Capella UI → App Services → Your App Service → App Endpoints → Edit → Access and Validation
```

You can also use the Capella API to update the sync function programmatically (useful for CI/CD pipelines).
