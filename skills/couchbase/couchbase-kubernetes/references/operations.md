# Operations

## Rolling upgrades

To upgrade Couchbase Server version via the operator:

1. Update `spec.image` in the `CouchbaseCluster` resource:
```bash
kubectl patch couchbasecluster couchbase-prod \
  --namespace couchbase-prod \
  --type merge \
  -p '{"spec":{"image":"couchbase/server:8.0.1"}}'
```

2. The operator detects the change and starts a rolling upgrade automatically:
   - Takes one node offline (graceful failover)
   - Updates the pod image
   - Brings the node back online and rebalances
   - Repeats for each node

3. Monitor progress:
```bash
kubectl describe couchbasecluster couchbase-prod -n couchbase-prod
# Watch the "Status" section for current upgrade progress

kubectl get events -n couchbase-prod --sort-by='.lastTimestamp'
# Shows operator actions in sequence
```

**Important:** before changing `spec.image`, confirm the CAO version supports the target Couchbase Server version. Check the compatibility matrix at `docs.couchbase.com/operator`.

## Scaling

Scale a node pool by updating `spec.servers[].size`:

```bash
# Scale data nodes from 3 to 5
kubectl patch couchbasecluster couchbase-prod \
  --namespace couchbase-prod \
  --type json \
  -p '[{"op":"replace","path":"/spec/servers/0/size","value":5}]'
```

The operator adds the new nodes, adds them to the cluster, and triggers a rebalance automatically. Scale-in (reducing size) gracefully fails over the excess nodes before removing them.

## Expanding persistent volumes

If your storage class supports `allowVolumeExpansion: true`, you can expand PVs online:

```bash
# Update the VolumeClaimTemplate size in the CouchbaseCluster CRD
kubectl patch couchbasecluster couchbase-prod \
  --namespace couchbase-prod \
  --type json \
  -p '[{"op":"replace","path":"/spec/volumeClaimTemplates/0/spec/resources/requests/storage","value":"800Gi"}]'
```

The operator detects the change and resizes the PVCs on each node sequentially. The cluster stays online.

**Limitation:** you can only expand, not shrink. PV expansion is also limited by what your cloud provider supports (EBS: can expand at any time; some providers require stopping the instance).

## Prometheus monitoring

CAO exposes Couchbase metrics via the standard Prometheus scrape endpoint on each pod. To configure Prometheus to scrape them:

```yaml
# ServiceMonitor (for Prometheus Operator / kube-prometheus-stack)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: couchbase-metrics
  namespace: couchbase-prod
  labels:
    release: prometheus    # matches your Prometheus Operator's serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: couchbase
  endpoints:
    - port: prometheus
      path: /metrics
      scheme: https
      tlsConfig:
        insecureSkipVerify: false
        caFile: /etc/prometheus/couchbase-ca.crt
      basicAuth:
        username:
          name: couchbase-operator-credentials
          key: username
        password:
          name: couchbase-operator-credentials
          key: password
      interval: 30s
```

Enable the Prometheus port in `CouchbaseCluster`:
```yaml
spec:
  monitoring:
    prometheus:
      enabled: true
      image: couchbase/exporter:1.0.5
```

## Log access

Couchbase logs are written to the pods' volume mounts. Access them:

```bash
# Get logs from a specific pod
kubectl exec -n couchbase-prod couchbase-prod-data-nodes-0 \
  -- cat /opt/couchbase/var/lib/couchbase/logs/error.log | tail -100

# Or stream via kubectl logs (captures stdout — limited for Couchbase)
kubectl logs -n couchbase-prod couchbase-prod-data-nodes-0 --tail=100
```

For production log aggregation, deploy Fluent Bit or Filebeat as a DaemonSet to tail `/opt/couchbase/var/lib/couchbase/logs/` from each Couchbase pod's volume mount. See `couchbase-observability/references/log-aggregation.md` for shipper config.

## Troubleshooting operator issues

**Cluster stuck in "Progressing" state:**
```bash
kubectl describe couchbasecluster couchbase-prod -n couchbase-prod
# Look at "Conditions" and "Events" sections — operator emits detailed messages
```

**Pod in CrashLoopBackOff:**
```bash
kubectl logs -n couchbase-prod couchbase-prod-data-nodes-0 --previous
# --previous shows logs from the crashed container instance
```

**PVC not binding:**
```bash
kubectl get pvc -n couchbase-prod
# STATUS should be "Bound" — if "Pending", check storage class and AZ availability
kubectl describe pvc <pvc-name> -n couchbase-prod
```

**Rebalance stuck after node add:**
The operator triggers rebalance automatically. If it's stuck, check the operator logs:
```bash
kubectl logs -n couchbase-operator deployment/couchbase-operator | grep -i rebalance | tail -20
```

## Hibernation (cost saving for non-production)

CAO supports cluster hibernation — scales all Couchbase pods to zero while preserving PVs:

```bash
kubectl patch couchbasecluster couchbase-staging \
  --namespace couchbase-staging \
  --type merge \
  -p '{"spec":{"hibernate":true}}'
```

Resume:
```bash
kubectl patch couchbasecluster couchbase-staging \
  --namespace couchbase-staging \
  --type merge \
  -p '{"spec":{"hibernate":false}}'
```

Useful for dev/staging clusters that don't need to run 24/7. Data is preserved on the PVs.
