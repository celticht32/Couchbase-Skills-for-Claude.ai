# Installation

## Prerequisites

- `kubectl` configured against your target cluster
- Helm 3.x
- Kubernetes 1.27+ (or the equivalent managed service version)
- PersistentVolume provisioner available (EBS, GCE PD, Azure Disk, or local-path for dev)

## Install via Helm (recommended)

```bash
# Add the Couchbase Helm repository
helm repo add couchbase https://couchbase-partners.github.io/helm-charts/
helm repo update

# Install the operator into its own namespace
helm install couchbase-operator couchbase/couchbase-operator \
  --namespace couchbase-operator \
  --create-namespace \
  --set cluster.enabled=false    # operator only, no cluster yet
```

This installs:
- The CAO deployment (operator pod)
- CRDs (CouchbaseCluster, CouchbaseBucket, etc.)
- RBAC (ClusterRole, ClusterRoleBinding, ServiceAccount)
- Admission controller webhook (validates CRD schemas before apply)

Verify the operator is running:
```bash
kubectl get pods -n couchbase-operator
# NAME                                    READY   STATUS    RESTARTS   AGE
# couchbase-operator-7d9c8b6f4d-xk2pq    1/1     Running   0          2m
```

## Namespaces

CAO can watch a single namespace (default) or multiple namespaces. For production, use dedicated namespaces per environment:

```bash
# Create namespaces
kubectl create namespace couchbase-prod
kubectl create namespace couchbase-staging

# If operator watches all namespaces (set watchNamespace in Helm values)
helm install couchbase-operator couchbase/couchbase-operator \
  --set operator.watchNamespace=""   # empty = watch all namespaces
```

## Admission controller

The admission controller webhook validates CRD resources before Kubernetes accepts them. It's installed automatically with the Helm chart. If your cluster restricts webhook registration, check the operator's ClusterRole includes `admissionregistration.k8s.io` permissions.

## Credentials secret

CAO needs the Couchbase admin credentials to manage the cluster. Create a secret before deploying the cluster:

```bash
kubectl create secret generic couchbase-operator-credentials \
  --namespace couchbase-prod \
  --from-literal=username=Administrator \
  --from-literal=password='$(openssl rand -base64 32)'
```

Reference this secret in the `CouchbaseCluster` spec (`spec.security.adminSecret`).

## TLS secret (production)

For TLS, create a secret containing the CA certificate, node certificate, and private key:

```bash
kubectl create secret generic couchbase-server-tls \
  --namespace couchbase-prod \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key \
  --from-file=ca.crt=ca.crt
```

Or use cert-manager (recommended for automated rotation):

```yaml
# cert-manager Certificate resource
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: couchbase-tls
  namespace: couchbase-prod
spec:
  secretName: couchbase-server-tls
  issuerRef:
    name: your-cluster-issuer
    kind: ClusterIssuer
  dnsNames:
    - "*.couchbase-prod.svc.cluster.local"
    - "*.couchbase-prod"
```

## Storage class requirements

For production data nodes, use a storage class that:
- Has `allowVolumeExpansion: true` (for online volume expansion)
- Has `volumeBindingMode: WaitForFirstConsumer` (ensures the PV is created in the same AZ as the pod)
- Provides durable block storage (AWS gp3, GCE pd-ssd, Azure Premium LRS)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: couchbase-data
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain    # IMPORTANT: Retain, not Delete — protects data on pod deletion
```

Use `reclaimPolicy: Retain` for data volumes. `Delete` will destroy the EBS volume when the PVC is deleted — which happens during operator upgrades and node scaling.
