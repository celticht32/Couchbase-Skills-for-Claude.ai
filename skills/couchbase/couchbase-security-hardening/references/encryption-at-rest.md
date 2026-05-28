# Encryption at rest (DARE and KMIP)

## What DARE covers

Data At Rest Encryption (DARE) encrypts data files on disk — vBucket data files, index files, and log files. It does not encrypt data in transit (that's TLS) or data in RAM.

DARE is Couchbase Enterprise Edition only. Not available in Community Edition or on nodes that don't have EE licensing.

## Enabling DARE

DARE is enabled per-bucket via `admin_encryption_update_settings`. It requires a running, healthy cluster with EE licensing.

```python
admin_encryption_get_settings(cluster="prod")
# Returns current DARE status: enabled/disabled, key type (internal/KMIP)

admin_encryption_update_settings(
    enabled=True,
    keyType="internal",   # "internal" (Couchbase-managed keys) or "kmip" (external KMIP server)
    cluster="prod"
)
```

**Important:** Enabling DARE on an existing bucket triggers a data re-encryption process. During this process:
- The cluster must remain healthy (don't rebalance, don't add nodes)
- I/O load increases significantly on affected nodes
- Time to complete depends on data volume; plan a maintenance window for large buckets

## Key management options

### Internal key management (simpler)

Couchbase generates and manages encryption keys internally. Keys are stored in the Couchbase keystore on each node. This is acceptable for:
- Environments where the threat model doesn't require external key custody
- Deployments without a compliance requirement for KMIP

Risk: if an attacker gains access to both the data files AND the keystore on the same node, they can decrypt the data.

### KMIP (external key management)

KMIP (Key Management Interoperability Protocol) stores encryption keys on an external key management server (Thales, Fortanix, HashiCorp Vault with KMIP plugin, AWS CloudHSM, etc.).

Use KMIP when:
- Compliance requires separation of key custody from data custody (PCI-DSS requirement 3.6, HIPAA)
- Your organization has an existing KMS infrastructure
- You need hardware security module (HSM) key protection

```python
admin_kmip_add_key_server(
    host="kmip.example.com",
    port=5696,
    certificate="-----BEGIN CERTIFICATE-----\n...",
    keyPath="/path/to/client.key",
    certPath="/path/to/client.crt",
    cluster="prod"
)

admin_encryption_update_settings(
    enabled=True,
    keyType="kmip",
    kmipServerId="<id from add_key_server>",
    cluster="prod"
)
```

KMIP requires mTLS between Couchbase and the key server. The Couchbase node presents a client certificate; the KMIP server presents a server certificate. Both sides must trust each other's CA.

## Key rotation

For internal keys:
```python
admin_encryption_rotate_data_key(cluster="prod")
```
Rotates the active encryption key. Old data is re-encrypted with the new key in the background.

For KMIP keys: key rotation is managed in the external KMS. After rotating the key in the KMS, trigger a re-encryption in Couchbase to apply the new key version.

**Key rotation frequency:** annually at minimum; quarterly for PCI-DSS scope. Set a calendar reminder — key rotation is easy to forget.

## Backup encryption

DARE encrypts data at rest on cluster nodes. Backup files written by cbbackupmgr are separate — they need their own encryption:

```bash
cbbackupmgr backup \
    --archive /backup/couchbase-archive \
    --repo prod-cluster \
    --cluster couchbase://10.0.0.1 \
    --username Administrator \
    --password <pass> \
    --encrypted \
    --passphrase "$(vault kv get -field=passphrase secret/couchbase/backup)"
```

Store the backup passphrase in a secrets manager. A backup encrypted with a lost passphrase is unrecoverable.

## DARE and Capella

Capella encrypts all data at rest by default (AES-256) using cloud provider managed keys (AWS KMS, Azure Key Vault, GCP KMS). No configuration required. Customer-managed keys (BYOK) are available in Capella Enterprise tier.
