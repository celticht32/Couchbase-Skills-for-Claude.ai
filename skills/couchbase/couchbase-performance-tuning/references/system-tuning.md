# System-level tuning

## Connection limits

Couchbase Data service accepts connections on port 11207 (TLS) and 11210 (plaintext). Each connection consumes a file descriptor and a small amount of memory.

**Default limits:**
- Per-node connection limit: set by OS `ulimit -n` and Couchbase's `max_connections` setting
- Default Couchbase `max_connections`: 65,000 per Data node

Signs you're hitting connection limits:
- SDK logs: `CONN_LIMIT_EXCEEDED` or `ECONNREFUSED` under load
- `curr_connections` per node is near `max_connections`
- New SDK connection attempts fail under load

**How to check:**
```python
admin_stats_nodes(cluster="prod")
# Look at: curr_connections, max_connections per node
```

**Fixes:**

1. **Reduce connections per client:** SDK connection pools should not have `max_size` greater than what the application actually needs concurrently. A pool of 10-20 connections per service instance is typical. 100+ per instance is a red flag.

2. **Increase `max_connections`** (if the hardware supports it):
```python
admin_cluster_update_settings(
    max_connections=100000,   # up from default 65000
    cluster="prod"
)
```

3. **Increase OS file descriptor limit:** `max_connections` is bounded by `ulimit -n`. Check with `ulimit -n` on the node; should be at least 200,000 for a production Couchbase node.

```bash
# /etc/security/limits.conf
couchbase soft nofile 200000
couchbase hard nofile 200000
```

4. **Add Data nodes:** each new Data node takes an independent share of connections. More nodes = more total connection capacity.

## OS-level tuning

### Swappiness

Couchbase is memory-intensive. If the OS starts swapping Couchbase's working memory to disk, performance collapses. Disable or minimize swap:

```bash
# Immediate (until reboot)
sysctl -w vm.swappiness=1

# Permanent
echo "vm.swappiness=1" >> /etc/sysctl.conf
sysctl -p
```

Setting to 0 disables swap entirely; 1 allows swap only to prevent OOM kills. Either is acceptable for Couchbase nodes. Do NOT use the default (60) — the OS will proactively swap Couchbase's buffers.

### Transparent Huge Pages (THP)

THP can cause latency spikes on Couchbase nodes due to background defragmentation. Disable it:

```bash
# Immediate
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Permanent (add to /etc/rc.local or a systemd unit)
```

### Network buffer sizes

For high-throughput clusters, increase the OS network buffer sizes:

```bash
# /etc/sysctl.conf
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

This matters most for XDCR and Analytics DCP feeds that sustain high throughput.

### Disk scheduler

For NVMe SSDs (most cloud instances), the scheduler should be `none` or `mq-deadline`:

```bash
# Check current scheduler
cat /sys/block/nvme0n1/queue/scheduler

# Set for the session
echo mq-deadline > /sys/block/nvme0n1/queue/scheduler
```

For spinning disks: use `deadline` or `cfq`. For NVMe: `none` (passthrough) or `mq-deadline`.

## NUMA awareness

On multi-socket servers (NUMA systems), Couchbase performs better when pinned to a single NUMA node. Spreading across NUMA nodes causes remote memory access latency.

Couchbase's startup scripts handle NUMA binding automatically on most Linux distributions. Verify with:

```bash
numactl --show
# Should show a single preferred node
```

On Kubernetes, use node affinity or NUMA-topology-aware scheduling if available.

## Couchbase service memory quotas

Each service has its own memory quota. Setting them wrong — especially too low — causes performance problems:

```python
admin_cluster_update_settings(
    data_memory_quota=16384,     # MB — Data service working set
    index_memory_quota=8192,     # MB — GSI index RAM
    fts_memory_quota=4096,       # MB — FTS / Search index RAM
    eventing_memory_quota=2048,  # MB — Eventing workers
    analytics_memory_quota=4096, # MB — Analytics service
    cluster="prod"
)
```

**Common mistake:** leaving Index service quota at the default (256 MB) after adding many indexes. When the Index service runs out of quota, it starts paging index data to disk — query latency spikes dramatically and `index_ram_percent` hits 100%. Always set the Index quota to at least 80% of the installed index RAM on Index nodes.

## Rebalance performance

Rebalance (adding/removing nodes, or after node failure) moves vBuckets between nodes. It's I/O-heavy and can take hours on large clusters.

Tuning rebalance speed vs. impact:

```python
admin_cluster_update_settings(
    rebalance_moves_per_node=4,    # concurrent vBucket moves per node (default 4)
    cluster="prod"
)
```

Increasing `rebalance_moves_per_node` speeds up rebalance but increases I/O pressure on the cluster. In production during business hours: keep at 2-4. During off-hours maintenance: can increase to 8-16 on fast NVMe storage.

Monitor rebalance progress:
```python
admin_rebalance_status(cluster="prod")
# Returns: status, progress %, estimated time
```

## Multi-service node contention

When multiple services share a node (Data + Index + Query), they compete for CPU, RAM, and disk I/O. Common contention patterns:

- **Index build competing with KV writes:** both generate heavy sequential disk I/O. Schedule index builds during low-write periods.
- **Query service CPU competing with Data service:** Query is CPU-intensive for complex queries. On shared nodes, a sudden increase in query load can starve the Data service.
- **Analytics DCP streaming vs. KV writes:** Analytics reads via DCP; heavy DCP streaming adds read I/O load.

Fix: dedicated nodes per service. If not possible, set service memory quotas conservatively and monitor per-service CPU usage via `admin_stats_nodes`.
