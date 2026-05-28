# CouchbaseCluster CRD

The `CouchbaseCluster` resource is the primary definition of your cluster. A minimal annotated production example:

```yaml
apiVersion: couchbase.com/v2
kind: CouchbaseCluster
metadata:
  name: couchbase-prod
  namespace: couchbase-prod
spec:
  # Admin credentials secret
  security:
    adminSecret: couchbase-operator-credentials
    rbac:
      managed: true    # operator manages CouchbaseUser/CouchbaseGroup CRDs

  # TLS (optional but recommended for production)
  networking:
    tls:
      static:
        serverSecret: couchbase-server-tls
        operatorSecret: couchbase-operator-tls
    exposeAdminConsole: true
    adminConsoleServices:
      - data

  # Couchbase Server image
  image: couchbase/server:8.0.1

  # Anti-affinity: prevent multiple Couchbase pods on the same k8s node
  antiAffinity: true

  # Server groups = AZ awareness
  serverGroups:
    - us-east-1a
    - us-east-1b
    - us-east-1c

  # Node topology
  servers:
    # Data nodes (3 nodes, one per AZ)
    - name: data-nodes
      size: 3
      services:
        - data
      serverGroups:
        - us-east-1a
        - us-east-1b
        - us-east-1c
      resources:
        requests:
          cpu: "4"
          memory: "16Gi"
        limits:
          cpu: "4"
          memory: "16Gi"
      volumeMounts:
        default: couchbase-data    # maps to VolumeClaimTemplate below
        data: couchbase-data
      pod:
        spec:
          nodeSelector:
            node-role: couchbase-data

    # Index + Query nodes (shared, 2 nodes)
    - name: index-query-nodes
      size: 2
      services:
        - index
        - query
      resources:
        requests:
          cpu: "8"
          memory: "32Gi"
        limits:
          cpu: "8"
          memory: "32Gi"
      volumeMounts:
        default: couchbase-index
        index: couchbase-index

  # Persistent volume templates
  volumeClaimTemplates:
    - metadata:
        name: couchbase-data
      spec:
        storageClassName: couchbase-data
        resources:
          requests:
            storage: 500Gi
        accessModes:
          - ReadWriteOnce

    - metadata:
        name: couchbase-index
      spec:
        storageClassName: couchbase-data
        resources:
          requests:
            storage: 200Gi
        accessModes:
          - ReadWriteOnce
```

## Server groups and AZ awareness

`spec.serverGroups` declares the available groups. `spec.servers[].serverGroups` lists which groups a node pool can be scheduled into. CAO uses pod anti-affinity to distribute pods across the declared groups.

For this to work, your Kubernetes nodes must have the topology label that CAO uses. By default CAO reads `topology.kubernetes.io/zone`. If your nodes don't have this label, add it or configure CAO to use a different label via `spec.serverGroups[].nodeAffinitySelector`.

**Minimum for HA:** at least 3 groups with at least 1 data node per group, and replica count ≥ 1. This ensures any single AZ failure leaves all data accessible via replicas.

## Service breakdown

Declare which services run on which node pools. Common production patterns:

**Small cluster (3 nodes, all services):**
```yaml
servers:
  - name: all-services
    size: 3
    services: [data, query, index, fts, eventing, analytics, backup]
```

**Medium cluster (separate data from query/index):**
```yaml
servers:
  - name: data
    size: 3
    services: [data]
  - name: query-index
    size: 2
    services: [query, index]
```

**Large cluster (dedicated per service):**
```yaml
servers:
  - name: data
    size: 5
    services: [data]
  - name: index
    size: 3
    services: [index]
  - name: query
    size: 3
    services: [query]
  - name: fts
    size: 2
    services: [fts]
  - name: analytics
    size: 2
    services: [analytics]
```

## Resource requests and limits

**Always set both requests and limits to the same value** (Guaranteed QoS class). If limits > requests, the pod can burst but Kubernetes may OOM-kill it under node pressure. Couchbase is sensitive to OOM kills — it's a database.

Memory sizing guidance: set the pod memory limit to match the Couchbase service memory quotas plus ~2 GB overhead for the OS and Couchbase processes outside of managed memory.

## Storage volume mapping

Each node pool can have multiple volume mounts mapped to named claim templates:

| Mount name | What it holds | Typical size |
|---|---|---|
| `data` | KV data files (vBuckets) | 2-3× working set size |
| `index` | GSI index files | 1.5× index RAM quota |
| `analytics` | Analytics service data | Depends on dataset |
| `eventing` | Eventing metadata | Small — 10-20 GB |
| `logs` | Couchbase logs | 20-50 GB |
| `default` | Any service not specifically mapped | Match the primary service |

Use separate volumes for `data` and `logs` in production. Logs filling the data volume is a known failure mode.

## In-place upgrades (spec.upgradeStrategy)

```yaml
spec:
  upgradeStrategy: InPlace    # or RollingUpgrade (default)
```

- `RollingUpgrade` (default): CAO gracefully fails over each node, upgrades it, adds it back, rebalances. Slower but safer.
- `InPlace`: upgrades nodes without full rebalance cycles. Faster but requires the cluster to be healthy. Only for minor version bumps within the same major version.
