# Capella networking

## Allowed CIDRs (public internet access)

The simplest connectivity option: add your application's public IP(s) to the cluster's allowlist.

**Clusters → your cluster → Connect → Allowed IP Addresses → Add Allowed IP**

Add the CIDR notation of your application servers' outbound IPs. For a single IP: `203.0.113.42/32`.

**Gotchas:**
- Cloud provider NAT gateways change IPs on restart. Pin a static Elastic IP (AWS) or equivalent.
- CI/CD pipelines often have dynamic IPs — add the CI provider's IP range or use a proxy.
- `0.0.0.0/0` allows all internet traffic — never use in production.

## AWS VPC Peering

Connects your AWS VPC directly to the Capella VPC. Traffic stays within AWS, off the public internet. Lower latency, no egress costs, more secure.

**Requirements:**
- Your application and Capella cluster must be in the same AWS region
- Your VPC CIDR must not overlap with Capella's VPC CIDR (Capella uses 10.0.0.0/23 by default)

**Setup:**
1. Capella UI → Clusters → your cluster → Connect → VPC Peering → Add VPC Peering
2. Provide your AWS Account ID, VPC ID, and VPC CIDR
3. Capella creates a VPC peering request in your AWS account
4. Accept the peering request in AWS Console (VPC → Peering Connections)
5. Update your VPC route tables to route Capella's CIDR through the peering connection
6. Add a Capella allowed IP entry for your VPC's CIDR (even with peering, the allowlist still applies)

Test: `nc -zv cb.<cluster-id>.cloud.couchbase.com 11207` from an EC2 instance in the peered VPC.

## AWS PrivateLink (Enterprise tier)

Provides a private endpoint in your VPC that resolves to Capella without VPC peering. No route table changes required. Available on Enterprise tier clusters.

**Setup:**
1. Capella UI → Clusters → your cluster → Connect → Private Endpoints → Add Private Endpoint
2. Capella provides a VPC Endpoint Service name (e.g. `com.amazonaws.vpce.us-east-1.vpce-svc-xxx`)
3. In AWS Console: VPC → Endpoints → Create Endpoint → use the service name
4. Choose the subnets where your application runs
5. Capella approves the endpoint connection (usually automatic for Enterprise tier)
6. DNS: the endpoint provides a private DNS hostname. Use this in your connection string instead of the public `cb.*.cloud.couchbase.com` hostname.

## Azure Private Endpoint (Enterprise tier)

Same concept as AWS PrivateLink but for Azure.

1. Capella UI → Private Endpoints → Add
2. Provide your Azure Subscription ID, Resource Group, and VNet details
3. Capella creates a Private Link Service
4. In Azure Portal: create a Private Endpoint connected to the Capella service
5. Approve the connection in Capella UI
6. Update your application's connection string to use the private endpoint hostname

## GCP Private Service Connect

Available on Enterprise tier for GCP-hosted Capella clusters. Same general pattern as AWS PrivateLink.

## Recommended networking by environment

| Environment | Recommended connectivity |
|---|---|
| Development / local | Allowed CIDR (your public IP) |
| Staging (cloud-hosted) | VPC Peering (same region as cluster) |
| Production (AWS) | AWS PrivateLink (Enterprise) or VPC Peering |
| Production (Azure) | Azure Private Endpoint (Enterprise) |
| CI/CD pipelines | Allowed CIDR for CI provider's IPs, or deploy test cluster in same VPC |

## Troubleshooting connectivity

**"Connection refused" / timeout:**
1. Check allowed CIDRs — is your current IP in the list?
2. Check `nslookup cb.<cluster-id>.cloud.couchbase.com` — should resolve to a public IP (or private IP if using PrivateLink)
3. Check port 11207 (KV) and 18091 (management) are not blocked by your firewall or security group
4. Verify the connection string uses `couchbases://` (TLS) not `couchbase://`

**"Authentication failed":**
Verify you're using database credentials (Clusters → Connect → Database Access), not your Capella UI login credentials.
