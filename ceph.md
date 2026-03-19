# Ceph Cheat‑Sheet  
*A concise, production‑ready reference for the most common Ceph commands*  

---  

## Table of Contents  

1. [Cluster Services Overview](#cluster-services-overview)  
2. [Cluster Health & Topology](#cluster-health--topology)  
3. [Pool & RBD Management](#pool--rbd-management)  
4. [CephFS (MDS) Operations](#cephfs-mds-operations)  
5. [RGW / S3 Operations](#rgw--s3-operations)  
6. [Kubernetes CSI Integration](#kubernetes-csi-integration)  
7. [Recovery & Performance Tuning](#recovery--performance-tuning)  
8. [Trou‑related Settings & Eviction Policies](#eviction‑policies)  
9. [Useful One‑liners & Debugging](#useful‑one‑liners)  

---  

## 1. Cluster Services Overview  

| Service | Role | Systemd unit (per node) |
|---------|------|--------------------------|
| **mon** | Moniteur – assure la cohérence du cluster et la répartition des pools | `ceph-mon@<id>.service` |
| **mgr** | Manager – expose les métriques, le tableau de bord et les interfaces d’authentification | `ceph-mgr@<id>.service` |
| **mds** | Metadata Server – gère le CephFS (méta‑données) | `ceph-mds@<id>.service` |
| **osd** | Object Storage Daemon – stockage brut (disques). Un OSD qui éch en panne indique un disque ou un nœud défectueux. | `ceph-osd@<id>.service` |
| **rgw** | RADOS Gateway – expose les API S3/Swift | `radosgw.service` |

> **Tip:** All services are enabled and started on every node that runs the respective daemon.

---  

## 2. Cluster Health & Topology  

```bash
# Global status (brief)
ceph -s

# Detailed health information
ceph health detail

# OSD tree (host → OSD mapping)
ceph osd tree
```

---  

## 3. Pool & RBD Management  

### 3.1 Create a pool (example: `k8s_dev`)  

```bash
# Create the pool
ceph osd pool create k8s_dev <pg_num>   # <pg_num> optional, defaults to 128

# Initialise the pool for RBD
rbd pool init k8s_dev
```

### 3.2 Create a client for the pool  

```bash
ceph auth get-or-create client.k8s_dev \
    mon 'profile rbd' \
    osd 'profile rbd pool=k8s_dev' \
    mgr 'profile rbd pool=k8s_dev'
```

### 3.3 Verify / List  

```bash
# List all pools
ceph osd pool ls

# List RBD images in a pool (long format)
rbd ls -l k8s_dev

# Show the client caps
ceph auth get client.k8s_dev
```

### 3.4 Quotas  

```bash
# Set a quota on a pool (objects or bytes)
ceph osd pool set-quota <pool-name> max_objects <obj-count>
ceph osd pool set-quota <pool-name> max_bytes   <bytes>

# Retrieve quota configuration
ceph config get-quota <pool-name>
```

---  

## 4. CephFS (MDS) Operations  

### 4.1 Basic FS commands  

```bash
# List all CephFS filesystems
ceph fs ls

# Show detailed FS info
ceph fs get <fs_name>

# FS status (active/standby MDS)
ceph fs status
```

### 4.2 Create pools for a new FS  

```bash
# Data pool
ceph osd pool create my_kns_cephfs_dev

# Metadata pool (usually small, e.g. 128 PGs)
ceph osd pool create my_kns_cephfs_dev_meta 128

# Enable CephFS application type (required for balancer, quotas, …)
ceph osd pool application enable my_kns_cephfs_dev      cephfs
ceph osd pool application enable my_kns_cephfs_dev_meta cephfs
```

### 4.3 Create the filesystem  

```bash
ceph fs new <fs_name> my_kns_cephfs_dev_meta my_kns_cephfs_dev
```

### 4.4 MDS specific configuration  

```bash
# Increase MDS cache memory (example: 16 GiB)
ceph config set mds mds_cache_memory_limit 17179869184   # 16 GiB in bytes

# Enable standby replay (helps with fast fail‑over)
ceph fs set <fs_name> allow_standby_replay true

# Automatic balancer
ceph fs set <fs_name> balance_automate true

# Limit number of active MDS daemons
ceph fs set <fs_name> max_mds 4
```

### 4.5 Sub‑volume groups (useful for CSI)  

```bash
# Create a sub‑volume group named “csi” inside the filesystem “transway_ftp_fs”
ceph fs subvolumegroup create my_ftp_fs csi
```

---  

## 5. RGW / S3 Operations  

```bash
# Bucket statistics
radosgw-admin bucket stats --bucket <bucket_name>

# Set a bucket‑level quota (example: 100 GiB)
radosgw-admin quota set \
    --quota-scope=bucket \
    --bucket=<bucket_name> \
    --max-size=100G
```

---  

## 6. Kubernetes CSI Integration  

> **Prerequisite:** Helm 3 and `kubectl` installed on the control‑plane node.

### 6.1 Add the Ceph‑CSI chart repository  

```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm repo update
```

### 6.2 Install the **RBD** CSI driver  

```bash
# Apply external‑snapshotter CRDs (required for volume snapshots)
kubectl kustomize https://github.com/kubernetes-csi/external-snapshotter/client/config/crd | kubectl apply -f -

helm upgrade --install -n kube-ceph-rbd \
    ceph-csi-rbd ceph-csi/ceph-csi-rbd \
    -f values_3_16_0.yaml \
    --version 3.16.0 \
    --set storageClass.pool=k8s_dev \
    --set volumeGroupSnapshotClass.pool=my_cc_infra_dev_backup \
    --create-namespace
```

### 6.3 Install the **CephFS** CSI driver  

```bash
helm upgrade --install -n kube-ceph-fs \
    ceph-csi-cephfs ceph-csi/ceph-csi-cephfs \
    -f values_3_16_0.yaml \
    --version 3.16.0 \
    --set storageClass.fsName=<fs_name> \
    --create-namespace

# Apply any additional RBAC / provisioner manifests
kubectl apply -f clusterrole-provisioner.yaml
```

---  

## 7. Recovery & Performance Tuning  

### 7.1 Show current recovery‑related settings  

```bash
ceph --show-config | egrep "osd_max_backfills|osd_recovery_max_active|osd_max_scrubs|osd_scrub_load|osd_recovery_op_priority"
```

### 7.2 Adjust on‑the‑fly (example for SSD‑backed OSDs)  

```bash
ceph tell 'osd.*' injectargs \
    '--osd-max-backfills 1 \
     --osd-recovery-max-active 3 \
     --osd-max-scrubs 1 \
     --osd-recovery-op-priority 2'
```

> **Note:** These values are **temporary**; add them to the global config (`ceph config set …`) for persistence.

---  

## 8. Eviction & Session‑Management Policies  

| Setting | Description | Typical Value |
|---------|-------------|---------------|
| `session_autoclose` | Time (seconds) of inactivity after which a client is evicted. | `300` (default) |
| `mds_cap_revoke_eviction_timeout` | Timeout for a client to answer a cap‑revoke request before eviction. | `0` (disabled) |
| `mds_reconnect_timeout` | Timeout during MDS re‑join. | `45` (default) |
| `mds_session_blacklist_on_timeout` | If `false`, slow clients are *not* black‑listed on timeout. | `false` (recommended for graceful) |
| `mds_session_blacklist_on_evict` | Same as above, but for manual evictions. | `false` |
| `mds_deny_all_reconnect` | Force all reconnect attempts to be denied (useful for maintenance). | `true` / `false` |
| `mds_cache_trim_decay_rate` | Controls how aggressively the MDS cache is trimmed. | `0.9` (example) |
| `mds_log_max_segments` | Maximum number of log segments per MDS (increase for heavy workloads). | `305000` |
| `mds_cache_memory_limit` | Upper bound of MDS cache memory (bytes). | `32GiB` (`34359738368`) |

### Example: Enable automatic client eviction after 1 hour  

```bash
ceph config set mds mds_cap_revoke_eviction_timeout 3600
```

### Example: Disable blacklist on timeout/eviction (prevents unnecessary PV unmounts)  

```bash
ceph config set mds mds_session_blacklist_on_timeout false
ceph config set mds mds_session_blacklist_on_evict  false
```

---  

## 9. Useful One‑liners & Debugging  

```bash
# List all OSD blocklists (useful when troubleshooting client bans)
ceph osd blocklist ls

# Show current device class for an OSD (e.g., osd.3)
ceph osd crush class show osd.3

# Change device class to SSD
ceph osd crush set-device-class ssd osd.3

# Remove old device‑class entries (wildcard)
ceph osd crush rm-device-class osd.X*
```

### Common CSI Provisioning Error  

```
Provision:ingFailed … failed to provision volume with StorageClass "csi-cephfs-sc":
 rpc error: code = Aborted desc = an operation with the given Volume ID … already exists
```

**Quick fixes**

1. Verify the CSI driver pod is healthy (`kubectl -n <ns> get pods`).
2. Ensure the **CephFS** filesystem name matches the `fsName` in the StorageClass.
3. Delete the stuck PVC/PV and retry, or manually clean the volume ID from Ceph:

```bash
# Find the volume ID (usually a RADOS object)
rados -p <data_pool> ls | grep <volume-id>
# Remove it if it is orphaned
rados -p <data_pool> rm <object-name>
```

---  
*Date : 2026‑03‑19*  
---  
