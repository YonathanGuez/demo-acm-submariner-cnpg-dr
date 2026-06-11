# Demonstration Submariner: OCP ACM Multi-Cluster Disaster Recovery Stack - Helm Charts

## General Overview

This guide demonstrates how to deploy **Submariner** for cross-cluster networking on two AWS-based OpenShift clusters managed by Advanced Cluster Management (ACM).

The setup includes:

- **Submariner**: Enables secure networking between clusters
- **CNPG (CloudNativePG)**: PostgreSQL operator for database deployment
- **Barman**: Cloud-native backup and restore capabilities
- **MinIO**: Object storage for database backups
- **Database Replication**: Real-time data synchronization between clusters

**Demo Environment**: demo.redhat.com → Advanced Cluster Management for Kubernetes Demo

### Specific Demonstration Overview: Active-Active Disaster Recovery with Submariner, CNPG, and MinIO

This demonstration showcases a production-ready, geographically distributed disaster recovery solution using Red Hat OpenShift and Advanced Cluster Management (ACM).

The setup ensures zero data loss and continuous availability by replicating a PostgreSQL database across two AWS regions with automated backups to object storage.

### What We're Building

**Two OpenShift clusters** (Ireland & Paris) connected via **Submariner** for secure cross-cluster networking:

- **Cluster 1 (Primary)**: Runs the main PostgreSQL database + MinIO backup storage
- **Cluster 2 (Replica)**: Maintains a real-time copy of the database for failover capability
- **ACM Hub**: Orchestrates and manages both clusters centrally

---

## Architecture

### Cluster Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    ACM Hub (local-cluster)                  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ManagedClusterSet: submariner                       │   │
│  │  - Cluster1 (Primary)                                │   │
│  │  - Cluster2 (Replica)                                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │                                    │
         │ Submariner Tunnel (VXLAN)          │
         │ UDP 4500 (NAT Discovery)           │
         │ UDP 4800 (Data Plane)              │
         ▼                                    ▼
┌──────────────────────────┐      ┌──────────────────────────┐
│   Cluster1 (Primary)     │      │   Cluster2 (Replica)     │
│   eu-west-2 (Ireland)    │      │   eu-west-3 (Paris)      │
│                          │      │                          │
│ Pods: 10.128.0.0/14      │      │ Pods: 10.132.0.0/14      │
│ Svc:  172.30.0.0/16      │      │ Svc:  172.31.0.0/16      │
│                          │      │                          │
│ ┌──────────────────────┐ │      │ ┌──────────────────────┐ │
│ │ CNPG Database        │ │      │ │ CNPG Replica         │ │
│ │ (Primary)            │ │      │ │ (Streaming)          │ │
│ └──────────────────────┘ │      │ └──────────────────────┘ │
│ ┌──────────────────────┐ │      │                          │
│ │ MinIO Storage        │ │      │                          │
│ │(Backups:cnpg-backups)│ │      │                          │
│ └──────────────────────┘ │      │                          │
└──────────────────────────┘      └──────────────────────────┘
```

### Network Configuration

| Component           | Cluster1 (Primary)  | Cluster2 (Replica) |
| ------------------- | ------------------- | ------------------ |
| **Region**          | eu-west-2 (Ireland) | eu-west-3 (Paris)  |
| **Pod CIDR**        | 10.128.0.0/14       | 10.132.0.0/14      |
| **Service CIDR**    | 172.30.0.0/16       | 172.31.0.0/16      |
| **Machine Network** | 10.0.0.0/16         | 10.0.0.0/16        |
| **Host Prefix**     | /23                 | /23                |

### Deployment Flow

1. **Install Operators** on both clusters (CNPG, Barman, Cert-Manager)
2. **Deploy MinIO** on Cluster 1 for backup storage
3. **Deploy PostgreSQL Primary** on Cluster 1 with automated backups
4. **Configure Submariner** on ACM Hub to enable cross-cluster networking
5. **Deploy PostgreSQL Replica** on Cluster 2 with streaming replication from Cluster 1
6. **Verify Replication** by checking data synchronization between clusters

### Key Features

- **Automated Backups**: Database automatically backed up to MinIO every hour
- **Point-in-Time Recovery**: Restore database to any point in the past
- **Transparent Failover**: Applications can switch to replica cluster with minimal downtime
- **Centralized Monitoring**: ACM provides unified visibility across both clusters
- **GitOps Ready**: All configurations managed via Helm charts for version control

## Prerequisites

- OpenShift clusters: `acm-hub`, `cluster1`, `cluster2` contexts configured
- ACM installed on the hub

### Recommendation:

You can use the project [ocp-acm-cluster-provisioner](https://github.com/YonathanGuez/ocp-acm-cluster-provisioner) to provision clusters in your ACM. After building all your clusters, use the script [init-local-clusters.sh](https://github.com/YonathanGuez/ocp-acm-cluster-provisioner/blob/main/init-local-clusters.sh) to configure the naming convention `acm-hub`, `cluster1`, `cluster2`.

## Quick Install

Multi-cluster deployment stack: **Operators** → **MinIO** → **CNPG Database** → **Submariner ACM** :

```bash
# 1. Operators (cert-manager, CNPG, Barman) on both clusters
helm install operators operators --kube-context cluster1 --dependency-update
helm install operators operators --kube-context cluster2 --dependency-update

# 2. MinIO storage (on cluster1)
helm install minio-storage minio-storage --kube-context cluster1

# 3. CNPG database (on cluster1)
helm install cnpg-deployment cnpg-deployment --kube-context cluster1

# 4. Submariner ACM (on hub - configures ManagedClusterSet, Broker, gateway labels)
helm install submariner-acm submariner-acm --kube-context acm-hub
```

Submariner must be deployed last because it uses service exports that depend on the `cloudnative-pg-operator` namespace, which is only available after deploying `cnpg-deployment`.

If you need to remove and reinstall `cnpg-deployment`, remember to run the following command afterward:

```
helm upgrade submariner-acm submariner-acm --kube-context acm-hub
```

Then verify that the service export has been redeployed:

```
oc get serviceexport -n cloudnative-pg-operator --context cluster1
```

<!-- #### Export Service via Submariner

Submariner does not automatically share all services. Export the primary cluster's read-write service:

```bash
helm install submariner-acm charts/submariner-acm --kube-context cluster1
``` -->

## Replica Database Deployment (Cluster2)

```bash
 helm install cnpg-deployment cnpg-deployment \
    -f cnpg-deployment/values-cluster2.yaml \
    --set-file replica.certBridge.cluster1Kubeconfig=<(oc --context cluster1 config view --minify --flatten)  --kube-context cluster2
```

## Manual Database Verification

Access the database pod directly:

```bash
# Connect to the replica pod
oc --context cluster2 rsh -n cloudnative-pg-operator cluster-replica-stream-1

# Inside the pod, connect to PostgreSQL
psql -h localhost -U app -d yoni
```

Then run:

```sql
\x
SELECT * FROM users;
\x
```

## Uninstall

Run in reverse install order. Use the script below — it handles the extra cleanup steps required (finalizers, orphaned CRDs, stuck namespaces).

```bash
#!/usr/bin/env bash
set -euo pipefail

CLUSTER1_CTX="cluster1"
CLUSTER2_CTX="cluster2"
HUB_CTX="acm-hub"

echo "==> Uninstalling cnpg-deployment..."
helm uninstall cnpg-deployment --kube-context "$CLUSTER1_CTX" 2>/dev/null || true

echo "==> Uninstalling minio-storage..."
helm uninstall minio-storage --kube-context "$CLUSTER1_CTX" 2>/dev/null || true

echo "==> Uninstalling submariner-acm..."
helm uninstall submariner-acm --kube-context "$HUB_CTX" 2>/dev/null || true

uninstall_operators() {
  local ctx="$1"
  echo "==> Uninstalling operators on $ctx..."

  # Remove the cnpg.io/cleanupPlugin finalizer that blocks namespace termination
  oc patch service barman-cloud -n cloudnative-pg-operator --context "$ctx" \
    --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]' 2>/dev/null || true

  helm uninstall operators --kube-context "$ctx" 2>/dev/null || true

  # Helm intentionally keeps CNPG CRDs on uninstall — delete them manually
  echo "==> Deleting leftover CNPG CRDs on $ctx..."
  oc delete crd \
    backups.postgresql.cnpg.io \
    clusterimagecatalogs.postgresql.cnpg.io \
    clusters.postgresql.cnpg.io \
    databases.postgresql.cnpg.io \
    failoverquorums.postgresql.cnpg.io \
    imagecatalogs.postgresql.cnpg.io \
    poolers.postgresql.cnpg.io \
    publications.postgresql.cnpg.io \
    scheduledbackups.postgresql.cnpg.io \
    subscriptions.postgresql.cnpg.io \
    --context "$ctx" 2>/dev/null || true

  # Wait for the cloudnative-pg-operator namespace to fully terminate
  echo "==> Waiting for cloudnative-pg-operator namespace to terminate on $ctx..."
  oc wait --for=delete namespace/cloudnative-pg-operator \
    --context "$ctx" --timeout=120s 2>/dev/null || true
}

uninstall_operators "$CLUSTER1_CTX"
uninstall_operators "$CLUSTER2_CTX"

echo "==> Uninstall complete."
```

> **Why the extra steps?**
>
> - The `barman-cloud` service carries a `cnpg.io/cleanupPlugin` finalizer that prevents the `cloudnative-pg-operator` namespace from terminating. It must be removed before or after `helm uninstall`.
> - Helm never deletes CRDs on uninstall (by design, to protect existing data). Leaving them causes a `cannot be imported into the current release` error on the next `helm install`. Delete them manually if you want a clean reinstall.

## Subcharts

| Chart             | Target            | Description                                                                       |
| ----------------- | ----------------- | --------------------------------------------------------------------------------- |
| `operators`       | cluster1/cluster2 | cert-manager, CloudNativePG operator, Barman plugin                               |
| `submariner-acm`  | acm-hub           | ManagedClusterSet, Broker, SubmarinerConfig, gateway node labeling, ServiceExport |
| `minio-storage`   | cluster1          | MinIO deployment with bucket auto-creation                                        |
| `cnpg-deployment` | cluster1          | PostgreSQL cluster (primary) with backups to MinIO                                |
| `cnpg-deployment` | cluster2          | PostgreSQL cluster (replica)                                                      |
