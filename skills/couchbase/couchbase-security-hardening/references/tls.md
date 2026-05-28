# TLS configuration

## TLS in Couchbase — what it covers

Couchbase has three distinct TLS surfaces:

1. **Client-to-cluster:** SDK connections, REST API, and MCP server connections from clients to the cluster. Controlled by the cluster's TLS settings and certificate.
2. **Node-to-node:** internal cluster traffic between nodes (rebalance, replication, admin). Controlled by the node-to-node encryption setting.
3. **XDCR:** replication traffic between clusters. Controlled per-reference (see `couchbase-xdcr/references/configuration.md`).

All three should be TLS-encrypted in production.

## Enabling and enforcing TLS

Via `admin_cluster_update_security_settings`:

```python
admin_cluster_update_security_settings(
    tlsMinVersion="tlsv1.2",        # Minimum TLS version: "tlsv1", "tlsv1.1", "tlsv1.2", "tlsv1.3"
    tlsCipherSuites=["TLS_AES_256_GCM_SHA384", "TLS_CHACHA20_POLY1305_SHA256"],  # optional allowlist
    disableUIOverHTTP=True,         # Redirect UI to HTTPS only
    disableUIOverHTTPSa=False,
    cluster="prod"
)
```

**Enforce TLS-only for client connections** (disables plaintext port 11210, keeps only 11207):

```python
admin_cluster_update_security_settings(
    clusterEncryptionLevel="strict",  # "control" | "all" | "strict"
    cluster="prod"
)
```

Encryption levels:
- `control` — management API (8091) uses TLS; data traffic may be plaintext
- `all` — all cluster traffic uses TLS; plaintext ports still open but unused
- `strict` — plaintext ports disabled; only TLS connections accepted (recommended for production)

**Warning:** setting `strict` before all clients and SDKs are configured for TLS will break those clients immediately. Migrate clients first, then enforce strict.

## Certificates

Couchbase uses a per-cluster CA certificate. Each node has a node certificate signed by the cluster CA.

**Viewing current certificates:**
```python
admin_cluster_get_security_settings(cluster="prod")
# Returns current TLS settings including certificate chain info
```

**Certificate requirements:**
- SAN (Subject Alternative Name) must include the node's IP address and/or hostname
- CA certificate must be installed on all clients that verify server certificates
- Certificate expiry monitoring is critical — an expired cert causes all TLS connections to fail

**Rotating certificates:**
1. Generate new CA (if rotating root) or new node certs signed by existing CA
2. Upload new certificates via the Couchbase UI or REST API (`/controller/uploadClusterCA`, `/node/controller/reloadCertificate`)
3. Verify new cert is served before removing old one
4. No cluster restart required for certificate rotation on 7.0+

## Node-to-node encryption

```python
admin_cluster_update_security_settings(
    nodeToNodeEncryption="all",  # "disabled" | "control" | "all"
    cluster="prod"
)
```

Enable node-to-node encryption before enabling client-side strict mode — internal traffic is a larger attack surface than external in many environments.

## mTLS (mutual TLS)

mTLS requires clients to present a certificate that the cluster trusts. Provides certificate-based authentication in addition to (or instead of) password authentication.

Configure via the cluster's client certificate settings (Couchbase UI → Security → Client Certificate). You specify:
- Which field in the certificate to map to a Couchbase username (e.g. CN or SAN)
- The path expression to extract the username

On the SDK side, configure the client certificate and key in the connection string or cluster options. See `couchbase-app-integration/references/connection-management.md` for SDK-specific code.

## TLS 1.3

Couchbase 7.1+ supports TLS 1.3. It's faster (0-RTT handshake) and more secure than 1.2. If all clients support 1.3, set `tlsMinVersion: "tlsv1.3"`. Check SDK version compatibility — older SDKs may not support TLS 1.3.
