# Harbor Registry with Rook-Ceph Storage Analysis

## Executive Summary

This document analyzes the deployment of Harbor container registry on the OwnTube MicroK8s cluster using Rook-Ceph for S3-compatible object storage. The goal is to provide multi-TB Docker registry hosting for a third-party company migrating from JFrog Artifactory, while comparing Ceph's operational features against the existing MinIO deployment.

**Key Benefits:**
- Utilize 28.7 TB of unused NVMe storage across the cluster
- Enterprise monitoring without licensing fees (vs MinIO Enterprise)
- Side-by-side comparison of Ceph vs MinIO S3 implementations
- Separate concerns: Registry (Ceph/NVMe) vs Video Storage (MinIO/SATA)
- No impact on existing MinIO workloads

## Current Infrastructure

### Hardware per Node (4 nodes total)

| Component | Specification | Current Usage |
|-----------|--------------|---------------|
| **CPU** | Intel i5-1340P / AMD Ryzen 9/7 | Kubernetes workloads |
| **RAM** | 64 GB | ~16-20 GB used |
| **M.2 NVMe** | 8 TB (7.27 TB usable) | 90 GB OS, **7.18 TB free** |
| **SATA SSD** | 8 TB (7.28 TB usable) | 6.4 TB MinIO data (87% used) |

### Storage Analysis

#### MinIO (Existing - SATA SSD)
```
Total Capacity: 4 nodes × 8 TB × 8 disks/node = 256 TB raw
Usable (distributed mode): 32 TB (accounting for parity/distribution)
Current Usage: 19 TiB (25.6 TB) data = 60% capacity
Free Space: ~28 GB per disk = 224 GB total (nearly full!)
```

**Pain Points:**
- Open source version lacks advanced monitoring
- No built-in Prometheus metrics exporter
- Limited operational visibility
- MinIO Enterprise license expensive ($$$)
- Nearly at capacity (87% per disk)

#### Available for Ceph (M.2 NVMe)
```
Total Available: 4 nodes × 7.18 TB = 28.72 TB raw NVMe capacity
Performance: NVMe (much faster than SATA)
Current Usage: Only 90 GB for OS (ubuntu-vg/ubuntu-lv)
```

**Advantages for Registry Workload:**
- High IOPS for frequent image pulls/pushes
- Lower latency than SATA
- Completely separate from MinIO storage pool

## Rook-Ceph Deployment Plan

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│           MicroK8s Cluster (4 nodes)                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌───────────────┐         ┌──────────────────┐   │
│  │   Harbor      │────────>│  Rook-Ceph RGW   │   │
│  │  (Registry)   │   S3    │  (Object Gateway)│   │
│  └───────────────┘         └──────────────────┘   │
│         │                           │              │
│         │                    ┌──────┴──────┐       │
│         │                    │   Ceph OSDs  │       │
│         │                    │  (4 × 7TB)   │       │
│         │                    └──────────────┘       │
│         │                                           │
│  ┌──────┴───────┐                                  │
│  │  MinIO S3    │  (Unchanged - Video Storage)     │
│  │  (4 × 6.4TB) │                                  │
│  └──────────────┘                                  │
└─────────────────────────────────────────────────────┘
```

### Ceph Storage Capacity Planning

**Option 1: Replication (High Availability)**
```
Raw Capacity:     28.72 TB
Replication:      3× (3 copies of data)
Overhead:         ~10% (Ceph metadata, journals)
Usable Capacity:  ~8.6 TB

Best for: Production workloads requiring high availability
Trade-off: Lower capacity, higher reliability
```

**Option 2: Erasure Coding (Space Efficient)**
```
Raw Capacity:     28.72 TB
Erasure Code:     k=2, m=1 (2 data + 1 parity chunks)
Overhead:         ~5% (Ceph metadata, journals)
Usable Capacity:  ~18.2 TB

Best for: Large registry with acceptable recovery time
Trade-off: Higher capacity, slower rebuild on failures
```

**Recommendation for Harbor:** **Erasure Coding k=2, m=1**
- Registry images can be re-pushed if lost (source of truth is JFrog)
- 18 TB sufficient for multi-TB Docker registry
- Better resource utilization
- 33% storage overhead vs 200% for replication

### Resource Requirements

#### Ceph Daemons per Node

| Daemon | Count | RAM per Instance | CPU | Purpose |
|--------|-------|------------------|-----|---------|
| **OSD** | 1 | 2-4 GB | 0.5-1 core | Object Storage Device |
| **MON** | 1 (3 total) | 1-2 GB | 0.2 core | Cluster Monitor |
| **MGR** | 1 (2 total) | 1-2 GB | 0.2 core | Manager/Dashboard |
| **RGW** | 1-2 | 2-4 GB | 0.5 core | S3 Gateway |

**Total per Node:** ~6-10 GB RAM, ~1.5-2 CPU cores

**Cluster Total:** ~30 GB RAM, ~8 CPU cores

**Assessment:** 64 GB RAM per node can easily accommodate this overhead alongside existing workloads.

## Rook-Ceph vs MinIO Comparison

| Feature | Rook-Ceph (OSS) | MinIO (OSS) | MinIO Enterprise |
|---------|-----------------|-------------|------------------|
| **Prometheus Metrics** | ✅ Native | ⚠️ Limited | ✅ Full |
| **Grafana Dashboards** | ✅ Included | ❌ Manual | ✅ Included |
| **Web UI** | ✅ Ceph Dashboard | ✅ MinIO Console | ✅ Enhanced |
| **Multi-site Replication** | ✅ RGW Sync | ❌ | ✅ Site Replication |
| **Erasure Coding** | ✅ Flexible (k+m) | ✅ Fixed schemes | ✅ Advanced |
| **Object Versioning** | ✅ S3 compatible | ✅ S3 compatible | ✅ Enhanced |
| **IAM / Access Control** | ✅ Keystone/LDAP | ✅ Basic | ✅ Advanced RBAC |
| **Encryption at Rest** | ✅ LUKS/dm-crypt | ❌ | ✅ KMS integration |
| **Tiering (Hot/Cold)** | ✅ Cache tiers | ❌ | ✅ Lifecycle |
| **Cost** | **Free** | **Free** | **$$$** |

**Key Advantages of Ceph for this Use Case:**
1. **Better operational visibility** - Built-in Prometheus exporters, detailed metrics
2. **No license walls** - All monitoring features available in OSS version
3. **Mature ecosystem** - CNCF backing, used by major cloud providers
4. **Performance comparison** - Real data to evaluate Ceph vs MinIO

## Harbor Deployment Architecture

### Harbor Components

```
Harbor Stack (Helm Chart)
├── Core (API, Web UI)
├── Registry (Distribution v2)
├── Job Service (Replication, GC)
├── Trivy (Vulnerability Scanning)
├── Database (PostgreSQL)
├── Redis (Cache)
└── Storage Backend → Rook-Ceph RGW (S3)
```

### Storage Backend Configuration

Harbor supports multiple storage drivers:
- **S3**: AWS S3, MinIO, Ceph RGW, etc.
- **Filesystem**: Local or NFS (not recommended for multi-node)
- **Swift**: OpenStack Object Storage

**For this deployment:** Use Ceph RGW with S3 protocol

### Harbor Helm Values (S3 Backend)

```yaml
persistence:
  persistentVolumeClaim:
    registry:
      storageClass: "rook-ceph-block"  # For PostgreSQL, Redis PVCs
      size: 100Gi
    database:
      storageClass: "rook-ceph-block"
      size: 20Gi
    redis:
      storageClass: "rook-ceph-block"
      size: 5Gi

# Container image storage via S3
registry:
  storage:
    s3:
      region: us-east-1  # Arbitrary for Ceph
      bucket: harbor-registry
      accesskey: <ceph-rgw-access-key>
      secretkey: <ceph-rgw-secret-key>
      regionendpoint: http://rook-ceph-rgw-ceph-objectstore.rook-ceph.svc.cluster.local
      secure: false  # Use true with ingress TLS termination
```

## Implementation Roadmap

### Phase 1: Rook-Ceph Deployment (Week 1)

1. **Install Rook Operator**
   ```bash
   kubectl create -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/crds.yaml
   kubectl create -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/common.yaml
   kubectl create -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/operator.yaml
   ```

2. **Create Ceph Cluster** (Custom Resource)
   - 4 OSDs (one per node using available NVMe LVM space)
   - 3 MONs (distributed across nodes)
   - 2 MGRs (active-standby)
   - Erasure coding pool: k=2, m=1

3. **Deploy Object Gateway (RGW)**
   ```bash
   kubectl create -f object-store.yaml
   ```

4. **Verify Cluster Health**
   ```bash
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
   ```

### Phase 2: Monitoring Setup (Week 1)

1. **Deploy Ceph Dashboard**
   - Integrated with Rook
   - Create ingress: `https://ceph-dashboard.owntube.tv`

2. **Configure Prometheus ServiceMonitor**
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: rook-ceph-mgr
     namespace: rook-ceph
   spec:
     selector:
       matchLabels:
         app: rook-ceph-mgr
   ```

3. **Import Grafana Dashboards**
   - Ceph Cluster Dashboard (ID: 2842)
   - Ceph OSD Dashboard (ID: 5336)
   - Ceph Pools Dashboard (ID: 5342)

### Phase 3: Harbor Deployment (Week 2)

1. **Create S3 Bucket in Ceph**
   ```bash
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
     radosgw-admin bucket create --bucket=harbor-registry --uid=harbor
   ```

2. **Generate S3 Credentials**
   ```bash
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
     radosgw-admin user create --uid=harbor --display-name="Harbor Registry"
   ```

3. **Deploy Harbor via Helm**
   ```bash
   helm repo add harbor https://helm.goharbor.io
   helm install harbor harbor/harbor \
     --namespace harbor --create-namespace \
     --values harbor-values.yaml
   ```

4. **Create Ingress**
   - HTTPS: `https://registry.owntube.tv`
   - Let's Encrypt certificate via cert-manager

### Phase 4: Migration & Testing (Week 3)

1. **Test Image Push/Pull**
   ```bash
   docker login registry.owntube.tv
   docker tag nginx:latest registry.owntube.tv/library/nginx:latest
   docker push registry.owntube.tv/library/nginx:latest
   ```

2. **Performance Benchmarking**
   - Compare push/pull speeds: Ceph vs MinIO
   - Measure IOPS and latency
   - Monitor resource usage

3. **JFrog Artifactory Migration**
   - Use Harbor replication feature
   - Or: `docker save/load` pipeline
   - Verify image integrity (SHA256 digests)

### Phase 5: Production Handoff (Week 4)

1. **Documentation**
   - User guide for customer
   - API access credentials
   - Backup/restore procedures

2. **Monitoring Alerts**
   - Ceph cluster health
   - Storage capacity thresholds
   - Harbor service availability

3. **Backup Strategy**
   - Database backups (PostgreSQL)
   - Ceph RGW metadata backups
   - Image layer dedupe verification

## Cost-Benefit Analysis

### Infrastructure Costs

| Item | Current (MinIO) | With Ceph Addition | Incremental |
|------|----------------|-------------------|-------------|
| **Hardware** | Paid (4 servers) | Same | $0 |
| **Storage Capacity** | 32 TB usable (SATA) | +18 TB usable (NVMe) | $0 |
| **Licensing** | $0 (OSS) | $0 (OSS) | $0 |
| **Power/Cooling** | ~$200/month | ~$220/month | +$20/month |
| **Monitoring Tools** | Limited | Enterprise-grade | $0 |

### Operational Benefits

**Quantifiable:**
- 18 TB additional storage capacity ($0 vs buying new hardware)
- Enterprise monitoring comparable to $5-10k/year MinIO license
- Performance improvement: NVMe (500k IOPS) vs SATA (100k IOPS)

**Strategic:**
- Real-world Ceph experience for team
- Revenue opportunity: Hosting registry for customer
- Better infrastructure utilization (NVMe was underused)
- Risk mitigation: Two S3 implementations (redundancy)

## Risk Assessment

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Ceph learning curve** | Medium | Low | Excellent documentation, CNCF community |
| **Resource contention** | Low | Medium | Monitor RAM/CPU, adjust if needed |
| **Data loss (4 node cluster)** | Low | High | Regular backups, customer keeps JFrog copy |
| **Network bottleneck** | Low | Low | 2.5 GbE links sufficient for registry |
| **Disk failure** | Medium | Low | Erasure coding handles 1 disk failure |

### Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Increased complexity** | High | Medium | Document procedures, use Ansible for automation |
| **Support burden** | Medium | Medium | SLA with customer, clear escalation path |
| **Backup failures** | Low | High | Automated testing, monitoring alerts |

**Overall Risk Level:** **MEDIUM-LOW** - Acceptable for production deployment with proper planning.

## Monitoring & Alerting Plan

### Ceph Metrics to Monitor

**Cluster Health:**
- `ceph_health_status` (HEALTH_OK, HEALTH_WARN, HEALTH_ERR)
- `ceph_mon_quorum_status` (monitor availability)
- `ceph_osd_up` / `ceph_osd_in` (OSD availability)

**Performance:**
- `ceph_osd_op_r_latency_sum` (read latency)
- `ceph_osd_op_w_latency_sum` (write latency)
- `ceph_pool_rd` / `ceph_pool_wr` (throughput)

**Capacity:**
- `ceph_cluster_total_bytes` (total capacity)
- `ceph_cluster_total_used_bytes` (used capacity)
- `ceph_pool_stored` (per-pool usage)

### Alert Rules (Prometheus)

```yaml
- alert: CephClusterWarning
  expr: ceph_health_status == 1
  for: 5m
  annotations:
    summary: "Ceph cluster health is WARN"

- alert: CephOSDDown
  expr: ceph_osd_up == 0
  for: 5m
  annotations:
    summary: "Ceph OSD {{ $labels.ceph_daemon }} is down"

- alert: CephStorageFull
  expr: (ceph_cluster_total_used_bytes / ceph_cluster_total_bytes) > 0.85
  annotations:
    summary: "Ceph cluster is 85% full"
```

## Comparison to MinIO Enterprise Features

This deployment provides **free alternatives** to MinIO Enterprise features:

| MinIO Enterprise | Ceph OSS Equivalent |
|------------------|---------------------|
| Prometheus metrics | ✅ Native Ceph exporter |
| Grafana dashboards | ✅ Community dashboards (IDs: 2842, 5336, 5342) |
| Multi-site replication | ✅ RGW multi-site sync |
| KMS encryption | ✅ dm-crypt / LUKS at rest |
| Lifecycle policies | ✅ RGW lifecycle (S3 compatible) |
| Health monitoring | ✅ Ceph health system |
| Audit logging | ✅ RGW ops log |
| Advanced RBAC | ✅ Keystone integration (optional) |

**Estimated License Savings:** $10,000 - $20,000/year (MinIO Enterprise subscription)

## Next Steps

1. **Approve deployment plan** - Review this document, get stakeholder buy-in
2. **Schedule maintenance window** - 4-hour window for initial Ceph deployment
3. **Prepare LVM volumes** - Create logical volumes on ubuntu-vg for Ceph OSDs
4. **Deploy Rook-Ceph** - Follow Phase 1 implementation roadmap
5. **Configure monitoring** - Integrate with existing Prometheus/Grafana
6. **Deploy Harbor** - Install and configure with Ceph backend
7. **Customer onboarding** - Coordinate JFrog migration, provide credentials

## References

- **Rook Documentation:** https://rook.io/docs/rook/latest/
- **Ceph Documentation:** https://docs.ceph.com/en/latest/
- **Harbor Documentation:** https://goharbor.io/docs/
- **CNCF Landscape:** https://landscape.cncf.io/
- **Kubernetes Storage:** https://kubernetes.io/docs/concepts/storage/

---

**Document Version:** 1.0
**Last Updated:** 2025-11-11
**Author:** Infrastructure Team
**Status:** Proposal - Awaiting Approval
