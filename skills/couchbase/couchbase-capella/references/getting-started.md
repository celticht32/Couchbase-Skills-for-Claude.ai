# Getting started with Capella

## Account and organization setup

1. Sign up at cloud.couchbase.com. Your account is tied to an **organization** — the top-level billing and management unit.
2. Create a **project** — a logical group of clusters (e.g. "production", "development"). Projects don't have billing implications; they're just organizational.
3. Create a **cluster** inside the project.

## Free tier

Capella's free tier gives you a single-node cluster with:
- 1 bucket, 1 GB storage, 1 GB RAM
- No credit card required
- Expires after 30 days (can be renewed)
- Connection via public internet (no VPC peering on free tier)

Free tier is for development and evaluation only. It has no SLA and is not suitable for production.

## Creating a cluster

In the Capella UI: **Clusters → Create Cluster**

Key decisions:
- **Cloud provider and region:** AWS, GCP, or Azure. Pick the region closest to your application. For production, match the region your application runs in to minimize latency.
- **Compute tier:** determines CPU, RAM, and storage per node. See `couchbase-sizing/references/capella.md` for tier selection guidance.
- **Node count:** 3 nodes minimum for production (high availability). 1 node is allowed for development.
- **Services:** choose which services each node runs (Data, Query, Index, Search, Eventing, Analytics). For small clusters, all services on all nodes. For large clusters, dedicated nodes per service.
- **Storage:** provisioned IOPS storage for production; general-purpose SSD for development.

## Cluster deployment time

Expect 5-15 minutes for a new cluster to be fully provisioned and ready for connections. The cluster status shows "deploying" → "rebalancing" → "healthy."

## After cluster creation checklist

1. **Create a bucket** — Clusters → your cluster → Buckets → Add Bucket
2. **Create database credentials** — Clusters → your cluster → Connect → Database Access → Create Credentials
3. **Configure allowed CIDRs** — Clusters → your cluster → Connect → Allowed IP Addresses → Add Allowed IP
4. **Get the connection string** — Clusters → your cluster → Connect → SDKs → copy the `couchbases://` string
5. **Test connectivity** — paste the connection string into your SDK code with the database credentials

## Connection string format

```
couchbases://cb.<cluster-id>.cloud.couchbase.com
```

The `couchbases://` prefix means TLS is required (port 11207 for KV, 18091 for management). Never use `couchbase://` (plaintext) with Capella.

## SDK connection example

```python
from couchbase.cluster import Cluster
from couchbase.auth import PasswordAuthenticator
from couchbase.options import ClusterOptions, ClusterTimeoutOptions
from datetime import timedelta

cluster = Cluster(
    "couchbases://cb.your-cluster-id.cloud.couchbase.com",
    ClusterOptions(
        PasswordAuthenticator("db-username", "db-password"),
        timeout_options=ClusterTimeoutOptions(kv_timeout=timedelta(seconds=10))
    )
)
cluster.wait_until_ready(timedelta(seconds=30))
```

Use the **database credentials** here — not your Capella UI login.
