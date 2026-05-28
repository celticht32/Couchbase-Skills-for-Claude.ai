# Network hardening

## Couchbase port reference

These are the ports that need to be open between components. Everything else should be blocked.

### Client to cluster

| Port | Protocol | Service | Notes |
|---|---|---|---|
| 8091 | HTTP | Management REST API, UI | Block from public internet; internal only |
| 18091 | HTTPS | Management REST API, UI (TLS) | Use this in production; block 8091 |
| 8093 | HTTP | Query service (N1QL) | Internal only |
| 18093 | HTTPS | Query service (TLS) | Use in production |
| 8094 | HTTP | FTS service | Internal only |
| 18094 | HTTPS | FTS service (TLS) | Use in production |
| 8095 | HTTP | Analytics service | Internal only |
| 18095 | HTTPS | Analytics service (TLS) | Use in production |
| 8096 | HTTP | Eventing service | Internal only |
| 18096 | HTTPS | Eventing service (TLS) | Use in production |
| 11210 | TCP | Data service (KV, SDK) | Plaintext — block when TLS strict mode enabled |
| 11207 | TCP+TLS | Data service (KV, SDK, TLS) | Use in production |

### Node to node (intra-cluster)

All nodes must be able to reach each other on:

| Port | Notes |
|---|---|
| 4369 | Erlang Port Mapper Daemon (EPMD) |
| 8091–8096 | Service ports (same as client) |
| 11209–11211 | KV (internal) |
| 21100–21299 | Data transfer (rebalance) |

Restrict intra-cluster traffic to the cluster's private subnet. Nodes should not be reachable on these ports from outside the cluster network.

### XDCR between clusters

| Port | Notes |
|---|---|
| 8091 / 18091 | Management API (for reference setup) |
| 11210 / 11207 | Data replication |

For XDCR over the public internet or across cloud VPCs, use 11207 (TLS) and set the XDCR reference `secure: "full"`.

## Firewall rule design

**Inbound to cluster nodes (from application tier):**
```
Allow TCP 18091 from app-security-group
Allow TCP 18093 from app-security-group
Allow TCP 18094 from app-security-group (if FTS used by app)
Allow TCP 11207 from app-security-group
Deny all other inbound
```

**Inbound to cluster nodes (from cluster nodes — intra-cluster):**
```
Allow all TCP from cluster-security-group
```

**Inbound to cluster nodes (from DBA/ops):**
```
Allow TCP 18091 from bastion/vpn-security-group
```

**Outbound from cluster nodes:**
```
Allow TCP to cluster-security-group (intra-cluster)
Allow TCP 18091 to remote-cluster-security-group (XDCR management)
Allow TCP 11207 to remote-cluster-security-group (XDCR data)
Allow TCP to KMIP server (if DARE with KMIP)
Allow TCP to LDAP server (if external auth)
Deny all other outbound (apply principle of least privilege)
```

## Capella network isolation

For Capella clusters:

1. **Allowed CIDRs:** configure the IP allowlist (Clusters → Connect → Allowed IPs). Only add your application's IP ranges, VPN exit IPs, and bastion IPs. Do not add `0.0.0.0/0`.
2. **Private networking:** Capella supports AWS PrivateLink and Azure Private Endpoint (Enterprise tier). Use private networking to keep traffic off the public internet.
3. **App Services:** if using Capella App Services (Sync Gateway), configure its allowed origins separately from the cluster's allowed CIDRs.

## Restricting Couchbase UI access

By default, the Couchbase UI is accessible on port 8091/18091 from any host that can reach those ports. In production:

1. Restrict 18091 to the DBA/ops security group only (no app servers need the UI)
2. Enable HTTPS-only (`disableUIOverHTTP: true`)
3. Consider putting the UI behind a VPN requirement

## Security groups on cloud deployments

Avoid over-permissive security groups:
- Don't use "allow all" intra-cluster rules if you can enumerate the specific ports
- Tag security groups clearly: `couchbase-cluster-sg`, `couchbase-client-sg`, `couchbase-xdcr-sg`
- Review security group rules quarterly — they accumulate stale rules over time (same as RBAC audit)
