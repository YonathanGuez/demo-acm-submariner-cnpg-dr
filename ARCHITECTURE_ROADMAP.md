# OCP Stacks Lab - Architecture Roadmap & Deployment Map

## Overview

This document provides a comprehensive architecture roadmap showing what gets deployed on each OpenShift cluster in the ACM multi-cluster disaster recovery setup.

---

## Deployment Architecture by Cluster

### 1. **ACM Hub Cluster** (Management Plane)

**Role:** Central management and orchestration hub

#### Deployed Components:

```
ACM Hub (local-cluster)
│
├── Submariner ACM Configuration
│   ├── ManagedClusterSet: submariner-set
│   │   ├── Cluster1 (Primary)
│   │   └── Cluster2 (Replica)
│   ├── Submariner Broker
│   │   └── Namespace: submariner-broker
│   ├── SubmarinerConfig (for Cluster1)
│   │   ├── Gateway Configuration
│   │   ├── IPSec Settings (IKE Port: 500, NATT Port: 4500)
│   │   └── Cable Driver: VXLAN
│   ├── SubmarinerConfig (for Cluster2)
│   │   ├── Gateway Configuration
│   │   ├── IPSec Settings (IKE Port: 500, NATT Port: 4500)
│   │   └── Cable Driver: VXLAN
│   ├── Gateway Node Labeling Jobs
│   │   ├── Label Cluster1 gateway nodes
│   │   └── Label Cluster2 gateway nodes
│   └── ManagedClusterSet Labels
│       └── submariner: enabled
│
└── Helm Release: submariner-acm
    └── Manages all Submariner resources
```

**Helm Chart:** `submariner-acm`
**Deployment Command:**

```bash
helm install submariner-acm submariner-acm --kube-context acm-hub
```

---

### 2. **Cluster 1 (Primary)** - eu-west-2 (Ireland)

**Role:** Primary database and backup storage

#### Network Configuration:

| Parameter       | Value         |
| --------------- | ------------- |
| Pod CIDR        | 10.128.0.0/14 |
| Service CIDR    | 172.30.0.0/16 |
| Machine Network | 10.0.0.0/16   |
| Host Prefix     | /23           |
| AWS Region      | eu-west-2     |
| Instance Type   | c5d.large     |

#### Deployed Components:

```
Cluster 1 (Primary)
│
├── Operators (Helm Release: operators)
│   ├── Namespace: cert-manager
│   │   └── Cert-Manager v1.20.2
│   │       ├── Certificate Management
│   │       └── TLS/SSL Handling
│   │
│   └── Namespace: cloudnative-pg-operator
│       ├── CloudNativePG Operator (stable-v1)
│       │   ├── PostgreSQL Cluster Management
│       │   ├── Backup & Recovery
│       │   └── Replication Control
│       │
│       └── Barman Cloud Plugin v0.12.0
│           ├── Cloud-native Backup
│           ├── WAL Archiving
│           └── Point-in-Time Recovery
│
├── MinIO Storage (Helm Release: minio-storage)
│   └── Namespace: minio
│       ├── MinIO Deployment
│       │   ├── Image: quay.io/minio/minio:latest
│       │   ├── Replicas: 1
│       │   ├── Storage: 5Gi
│       │   └── Credentials: minio/minio123
│       │
│       ├── Service: minio-service
│       │   ├── API Port: 9000
│       │   └── Console Port: 9090
│       │
│       ├── PersistentVolumeClaim: 5Gi
│       │
│       ├── Route (OpenShift)
│       │   ├── TLS Termination: edge
│       │   └── Access: minio-console.example.com
│       │
│       └── Bucket: cnpg-backups
│           └── Auto-created for database backups
│
├── CNPG Database (Helm Release: cnpg-deployment)
│   └── Namespace: cloudnative-pg-operator
│       ├── PostgreSQL Cluster: ocp-database
│       │   ├── Instances: 3
│       │   ├── Storage: 1Gi per instance
│       │   ├── Role: Primary (Read-Write)
│       │   ├── Database: yoni
│       │   ├── User: app
│       │   └── Password: app
│       │
│       ├── Bootstrap Configuration
│       │   ├── Init Database: yoni
│       │   ├── Owner: app
│       │   └── Init SQL Script
│       │       └── Creates 'users' table with sample data
│       │
│       ├── Replication Configuration
│       │   ├── Method: Streaming
│       │   ├── User: streaming_replication_user
│       │   ├── Password: streaming_replication_pwd
│       │   └── Target: Cluster2 (Replica)
│       │
│       ├── Backup Configuration
│       │   ├── Enabled: true
│       │   ├── Schedule: Every 2 hours (0 */2 * * * *)
│       │   ├── Destination: MinIO (s3://cnpg-backups/)
│       │   ├── Compression: gzip
│       │   └── Plugin: barman-cloud.cloudnative-pg.io
│       │
│       ├── Data Generator Job
│       │   ├── Image: postgres:16-alpine
│       │   ├── Insert Interval: 10 seconds
│       │   └── Generates test data for replication
│       │
│       ├── Secrets
│       │   ├── app-secret (database credentials)
│       │   ├── ocp-database-ca (CA certificate)
│       │   └── ocp-database-replication (replication certs)
│       │
│       └── ConfigMap
│           └── configmap-db (init SQL script)
│
├── Submariner Integration
│   ├── Service Export
│   │   └── Service: ocp-database-rw
│   │       ├── Namespace: cloudnative-pg-operator
│   │       ├── Port: 5432
│   │       └── Accessible from: Cluster2 via Submariner
│   │
│   ├── Gateway Node
│   │   ├── Label: submariner.io/gateway=true
│   │   └── Handles cross-cluster traffic
│   │
│   └── Submariner Operator
│       ├── Namespace: submariner-operator
│       ├── Cable Driver: VXLAN
│       ├── IPSec Enabled: true
│       └── NAT Traversal: Enabled
│
└── Network Connectivity
    └── Submariner Tunnel to Cluster2
        ├── Protocol: VXLAN over UDP
        ├── NAT Discovery Port: 4500
        ├── Data Plane Port: 4800
        └── Encryption: IPSec
```

**Helm Releases:**

```bash
# 1. Install Operators
helm install operators operators --kube-context cluster1 --dependency-update

# 2. Install MinIO Storage
helm install minio-storage minio-storage --kube-context cluster1

# 3. Install CNPG Database (Primary)
helm install cnpg-deployment cnpg-deployment --kube-context cluster1
```

---

### 3. **Cluster 2 (Replica)** - eu-west-3 (Paris)

**Role:** Replica database for disaster recovery

#### Network Configuration:

| Parameter       | Value         |
| --------------- | ------------- |
| Pod CIDR        | 10.132.0.0/14 |
| Service CIDR    | 172.31.0.0/16 |
| Machine Network | 10.0.0.0/16   |
| Host Prefix     | /23           |
| AWS Region      | eu-west-3     |
| Instance Type   | c5d.large     |

#### Deployed Components:

```
Cluster 2 (Replica)
│
├── Operators (Helm Release: operators)
│   ├── Namespace: cert-manager
│   │   └── Cert-Manager v1.20.2
│   │       ├── Certificate Management
│   │       └── TLS/SSL Handling
│   │
│   └── Namespace: cloudnative-pg-operator
│       ├── CloudNativePG Operator (stable-v1)
│       │   ├── PostgreSQL Cluster Management
│       │   ├── Backup & Recovery
│       │   └── Replication Control
│       │
│       └── Barman Cloud Plugin v0.12.0
│           ├── Cloud-native Backup
│           ├── WAL Archiving
│           └── Point-in-Time Recovery
│
├── CNPG Database (Helm Release: cnpg-deployment)
│   └── Namespace: cloudnative-pg-operator
│       ├── PostgreSQL Cluster: cluster-replica-stream
│       │   ├── Instances: 3
│       │   ├── Storage: 1Gi per instance
│       │   ├── Role: Replica (Read-Only)
│       │   ├── Database: postgres
│       │   ├── User: app
│       │   └── Password: app
│       │
│       ├── Bootstrap Configuration
│       │   ├── Method: pg_basebackup
│       │   ├── Source: ocp-database (Cluster1)
│       │   └── Connection String:
│       │       └── host=ocp-database-rw.cloudnative-pg-operator.svc.clusterset.local
│       │           user=streaming_replica
│       │           port=5432
│       │           dbname=postgres
│       │
│       ├── Replication Configuration
│       │   ├── Enabled: true
│       │   ├── Source: ocp-database (Cluster1)
│       │   ├── Method: Streaming
│       │   ├── User: streaming_replica
│       │   ├── Password: streaming_replication_pwd
│       │   └── SSL Mode: verify-ca
│       │
│       ├── External Cluster Reference
│       │   ├── Name: ocp-database
│       │   ├── Host: ocp-database-rw.cloudnative-pg-operator.svc.clusterset.local
│       │   │   (Resolved via Submariner)
│       │   ├── Port: 5432
│       │   ├── SSL Certificates
│       │   │   ├── CA: ocp-database-ca
│       │   │   ├── Key: ocp-database-replication (tls.key)
│       │   │   └── Cert: ocp-database-replication (tls.crt)
│       │   └── Kubeconfig Bridge
│       │       ├── Enabled: true
│       │       └── Source: cluster1-kubeconfig secret
│       │
│       ├── Backup Configuration
│       │   ├── Enabled: true
│       │   ├── Schedule: Every 2 hours (0 */2 * * * *)
│       │   ├── Destination: MinIO (s3://cnpg-backups/)
│       │   ├── Compression: gzip
│       │   └── Plugin: barman-cloud.cloudnative-pg.io
│       │
│       ├── Data Generator Job
│       │   ├── Image: postgres:16-alpine
│       │   ├── Insert Interval: 10 seconds
│       │   └── Generates test data for replication
│       │
│       ├── Secrets
│       │   ├── app-secret (database credentials)
│       │   ├── ocp-database-ca (CA certificate from Cluster1)
│       │   ├── ocp-database-replication (replication certs from Cluster1)
│       │   └── cluster1-kubeconfig (Cluster1 kubeconfig for cert bridge)
│       │
│       └── ConfigMap
│           └── configmap-db (init SQL script)
│
├── Submariner Integration
│   ├── Gateway Node
│   │   ├── Label: submariner.io/gateway=true
│   │   └── Handles cross-cluster traffic
│   │
│   ├── Service Import
│   │   └── Service: ocp-database-rw
│   │       ├── Imported from: Cluster1
│   │       ├── Namespace: cloudnative-pg-operator
│   │       ├── Port: 5432
│   │       └── Accessible via: Submariner tunnel
│   │
│   └── Submariner Operator
│       ├── Namespace: submariner-operator
│       ├── Cable Driver: VXLAN
│       ├── IPSec Enabled: true
│       └── NAT Traversal: Enabled
│
└── Network Connectivity
    └── Submariner Tunnel to Cluster1
        ├── Protocol: VXLAN over UDP
        ├── NAT Discovery Port: 4500
        ├── Data Plane Port: 4800
        └── Encryption: IPSec
```

**Helm Release:**

```bash
# Install Operators
helm install operators operators --kube-context cluster2 --dependency-update

# Install CNPG Database (Replica)
helm install cnpg-deployment cnpg-deployment \
  -f cnpg-deployment/values-cluster2.yaml \
  --set-file replica.certBridge.cluster1Kubeconfig=<(oc --context cluster1 config view --minify --flatten) \
  --kube-context cluster2
```

---

## Data Flow & Replication Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ACM Hub (Management)                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Submariner Broker & ManagedClusterSet Configuration    │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
         │                                    │
         │ Submariner Tunnel (VXLAN)          │
         │ UDP 4500/4800 (Encrypted)          │
         │                                    │
         ▼                                    ▼
┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│   Cluster 1 (Primary)            │  │   Cluster 2 (Replica)            │
│   eu-west-2 (Ireland)            │  │   eu-west-3 (Paris)              │
│                                  │  │                                  │
│  ┌────────────────────────────┐  │  │  ┌────────────────────────────┐  │
│  │ PostgreSQL Primary (RW)    │  │  │  │ PostgreSQL Replica (RO)    │  │
│  │ ocp-database               │  │  │  │ cluster-replica-stream     │  │
│  │ Instances: 3               │  │  │  │ Instances: 3               │  │
│  │ Database: yoni             │  │  │  │ Database: postgres         │  │
│  │                            │  │  │  │                            │  │
│  │ ┌──────────────────────┐   │  │  │  │ ┌──────────────────────┐   │  │
│  │ │ Write Operations     │   │  │  │  │ │ Read Operations      │   │  │
│  │ │ - INSERT             │   │  │  │  │ │ - SELECT             │   │  │
│  │ │ - UPDATE             │   │  │  │  │ │ (Read-Only)          │   │  │
│  │ │ - DELETE             │   │  │  │  │ │                      │   │  │
│  │ └──────────────────────┘   │  │  │  │ └──────────────────────┘   │  │
│  │                            │  │  │  │                            │  │
│  │ Streaming Replication ────────────────────────────────────────┐  │  │
│  │ (Real-time WAL streaming)  │  │  │  │                        │   │  │
│  │                            │  │  │  │ ◄─ Receives WAL logs   │   │  │
│  │                            │  │  │  │                        │   │  │
│  └────────────────────────────┘  │  │  └────────────────────────┘   │  │
│                                  │  │                                  │
│  ┌────────────────────────────┐  │  │                                  │
│  │ MinIO Storage              │  │  │                                  │
│  │ Namespace: minio           │  │  │                                  │
│  │                            │  │  │                                  │
│  │ ┌──────────────────────┐   │  │  │                                  │
│  │ │ Bucket: cnpg-backups │   │  │  │                                  │
│  │ │                      │   │  │  │                                  │
│  │ │ Backup Schedule:     │   │  │  │                                  │
│  │ │ Every 2 minutes      │   │  │  │                                  │
│  │ │                      │   │  │  │                                  │
│  │ │ Barman Cloud Plugin: │   │  │  │                                  │
│  │ │ - WAL Archiving      │   │  │  │                                  │
│  │ │ - Full Backups       │   │  │  │                                  │
│  │ │ - Point-in-Time      │   │  │  │                                  │
│  │ │   Recovery           │   │  │  │                                  │
│  │ └──────────────────────┘   │  │  │                                  │
│  │                            │  │  │                                  │
│  └────────────────────────────┘  │  │                                  │
│                                  │  │                                  │
└──────────────────────────────────┘  └──────────────────────────────────┘
```

---

## Deployment Order & Dependencies

### Phase 1: Operator Installation (Both Clusters)

```
1. Install Operators on Cluster1
   └── Installs: cert-manager, CNPG, Barman

2. Install Operators on Cluster2
   └── Installs: cert-manager, CNPG, Barman
```

### Phase 2: Storage & Database (Cluster1 Only)

```
3. Deploy MinIO Storage on Cluster1
   └── Creates: minio namespace, deployment, PVC, bucket

4. Deploy CNPG Primary Database on Cluster1
   └── Creates: ocp-database cluster, secrets, configmaps
```

### Phase 3: Submariner Configuration (ACM Hub)

```
5. Deploy Submariner ACM on Hub
   └── Creates: ManagedClusterSet, Broker, SubmarinerConfig
   └── Configures: Gateway nodes, Service exports
```

### Phase 4: Replica Database (Cluster2)

```
6. Deploy CNPG Replica Database on Cluster2
   └── Creates: cluster-replica-stream cluster
   └── Connects: To Cluster1 via Submariner tunnel
   └── Starts: Streaming replication
```

---

## Key Network Paths

### 1. **Cluster1 → Cluster2 (Replication)**

```
Cluster1 (Primary)
  └─ ocp-database-rw service (5432)
     └─ Exported via Submariner ServiceExport
        └─ Submariner Tunnel (VXLAN/IPSec)
           └─ Cluster2 (Replica)
              └─ cluster-replica-stream (receives WAL)
```

### 2. **Cluster1 → MinIO (Backups)**

```
Cluster1 (Primary)
  └─ PostgreSQL (Barman Cloud Plugin)
     └─ WAL Archiving & Full Backups
        └─ MinIO Service (9000)
           └─ S3 Bucket: cnpg-backups
```

### 3. **Cluster2 → Cluster1 (Cert Bridge)**

```
Cluster2 (Replica)
  └─ Kubeconfig Bridge
     └─ Accesses Cluster1 API
        └─ Retrieves: CA certs, replication certs
           └─ Establishes: Secure replication connection
```

---

## Namespace Organization

### Cluster1 & Cluster2:

```
cert-manager/
  └─ Cert-Manager Operator
  └─ Certificates for TLS/SSL

cloudnative-pg-operator/
  ├─ CNPG Operator
  ├─ Barman Cloud Plugin
  ├─ PostgreSQL Cluster (Primary or Replica)
  ├─ Secrets (credentials, certs)
  └─ ConfigMaps (init scripts)

minio/ (Cluster1 only)
  ├─ MinIO Deployment
  ├─ MinIO Service
  ├─ PersistentVolumeClaim
  └─ Bucket Configuration

submariner-operator/
  ├─ Submariner Operator
  ├─ Gateway Pods
  └─ Network Policies
```

### ACM Hub:

```
submariner-broker/
  ├─ Submariner Broker
  └─ Cluster Registration

cluster1/ (ManagedCluster namespace)
  ├─ ManagedCluster resource
  ├─ SubmarinerConfig
  └─ ManagedClusterAddOn

cluster2/ (ManagedCluster namespace)
  ├─ ManagedCluster resource
  ├─ SubmarinerConfig
  └─ ManagedClusterAddOn
```

---

## Storage & Persistence

### Cluster1:

```
MinIO PVC: 5Gi
  └─ Stores: Database backups, WAL archives
  └─ Bucket: cnpg-backups

CNPG PVC (per instance): 1Gi × 3 instances = 3Gi
  └─ Stores: PostgreSQL data files
```

### Cluster2:

```
CNPG PVC (per instance): 1Gi × 3 instances = 3Gi
  └─ Stores: Replicated PostgreSQL data files
```

---

## Security & Encryption

### Network Security:

- **Submariner Tunnel:** VXLAN over IPSec
- **Encryption:** AES-256 (IPSec)
- **NAT Traversal:** Enabled (UDP 4500)
- **Data Plane:** UDP 4800

### Database Security:

- **Replication User:** streaming_replication_user / streaming_replica
- **SSL/TLS:** verify-ca mode
- **Certificates:** Managed by Cert-Manager
- **Secrets:** Kubernetes Secrets (encrypted at rest)

### Backup Security:

- **MinIO Credentials:** minio/minio123
- **S3 Path Style:** Force path style for MinIO compatibility
- **Compression:** gzip (WAL & data)

---

## Monitoring & Verification Points

### Cluster1 (Primary):

```
✓ PostgreSQL cluster running (3 instances)
✓ MinIO storage accessible
✓ Backups scheduled and running
✓ Replication user created
✓ Service export active
✓ Submariner gateway operational
```

### Cluster2 (Replica):

```
✓ PostgreSQL cluster running (3 instances)
✓ Streaming replication active
✓ Data synchronized with Cluster1
✓ Kubeconfig bridge working
✓ Certificates valid
✓ Submariner gateway operational
```

### ACM Hub:

```
✓ ManagedClusterSet created
✓ Submariner Broker running
✓ Both clusters registered
✓ SubmarinerConfig applied
✓ Gateway nodes labeled
✓ Service exports visible
```

---

## Disaster Recovery Scenarios

### Scenario 1: Cluster1 Failure

```
1. Cluster1 becomes unavailable
2. Cluster2 has full replica of data
3. Applications can failover to Cluster2
4. Cluster2 can be promoted to primary
5. Backups in MinIO can restore Cluster1
```

### Scenario 2: Network Partition

```
1. Submariner tunnel detects partition
2. Replication pauses (data consistency)
3. Once network restored, replication resumes
4. WAL logs ensure no data loss
```

### Scenario 3: Data Corruption

```
1. Point-in-time recovery from MinIO backups
2. Barman Cloud plugin restores to specific time
3. Replication resynchronizes
4. Data integrity verified
```

---

## Summary Table

| Component               | Cluster1     | Cluster2     | ACM Hub |
| ----------------------- | ------------ | ------------ | ------- |
| **Cert-Manager**        | ✓            | ✓            | -       |
| **CNPG Operator**       | ✓            | ✓            | -       |
| **Barman Plugin**       | ✓            | ✓            | -       |
| **PostgreSQL**          | Primary (RW) | Replica (RO) | -       |
| **MinIO Storage**       | ✓            | -            | -       |
| **Submariner Operator** | ✓            | ✓            | -       |
| **Submariner Broker**   | -            | -            | ✓       |
| **SubmarinerConfig**    | ✓            | ✓            | -       |
| **ManagedClusterSet**   | -            | -            | ✓       |
| **Service Export**      | ✓            | -            | -       |
| **Service Import**      | -            | ✓            | -       |

---

## Helm Chart Deployment Summary

```bash
# Step 1: Install Operators on both clusters
helm install operators operators --kube-context cluster1 --dependency-update
helm install operators operators --kube-context cluster2 --dependency-update

# Step 2: Install MinIO on Cluster1
helm install minio-storage minio-storage --kube-context cluster1

# Step 3: Install CNPG Primary on Cluster1
helm install cnpg-deployment cnpg-deployment --kube-context cluster1

# Step 4: Install Submariner ACM on Hub
helm install submariner-acm submariner-acm --kube-context acm-hub

# Step 5: Install CNPG Replica on Cluster2
helm install cnpg-deployment cnpg-deployment \
  -f cnpg-deployment/values-cluster2.yaml \
  --set-file replica.certBridge.cluster1Kubeconfig=<(oc --context cluster1 config view --minify --flatten) \
  --kube-context cluster2
```

---

This architecture provides a **production-ready, geographically distributed disaster recovery solution** with:

- ✅ Real-time data replication
- ✅ Automated backups
- ✅ Secure cross-cluster networking
- ✅ Point-in-time recovery
- ✅ Centralized management via ACM
