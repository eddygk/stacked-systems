---
layout: post
title: "Kubernetes in the Homelab: Solving the Persistent Storage Challenge"
date: 2025-04-30
categories: [kubernetes, homelab, storage]
tags: [k3s, truenas, nfs, longhorn, openebs]
---

# Kubernetes in the Homelab: Solving the Persistent Storage Challenge

One of the most challenging aspects of running Kubernetes in a homelab environment is setting up reliable persistent storage. After experimenting with multiple solutions over the past six months, I've settled on a hybrid approach that balances performance, reliability, and ease of management.

## The Persistent Storage Problem

Unlike cloud environments where you can easily provision managed storage services, homelabs require custom storage solutions. The key challenges I faced were:

1. **Reliability**: Ensuring data persistence across node failures and reboots
2. **Performance**: Maintaining acceptable I/O for database workloads
3. **Simplicity**: Avoiding overly complex management overhead
4. **Resource Efficiency**: Working within the constraints of homelab hardware

## Solutions I Tested

I systematically tested the following storage options in my K3s cluster:

### 1. NFS Server with TrueNAS Scale

**Setup**: External TrueNAS Scale server with NFS shares mounted by Kubernetes using the NFS provisioner.

**Performance**:
- Sequential Read: 110 MB/s
- Sequential Write: 85 MB/s
- Random 4K Read: 3,200 IOPS
- Random 4K Write: 850 IOPS

**Pros**:
- Easy to set up
- Data managed outside of Kubernetes
- Snapshot support via ZFS
- Reliable for most applications

**Cons**:
- Network bottlenecks
- Lock contention issues with some applications
- Single point of failure
- Performance inadequate for database workloads

### 2. OpenEBS with Local PVs

**Setup**: OpenEBS with LocalPV storage class using dedicated SSDs on each node.

**Performance**:
- Sequential Read: 520 MB/s  
- Sequential Write: 480 MB/s
- Random 4K Read: 24,000 IOPS
- Random 4K Write: 12,500 IOPS

**Pros**:
- Excellent performance
- Simple setup
- Low overhead

**Cons**:
- No replication (data loss on node failure)
- Manual PV management across nodes
- Limited data protection features

### 3. Longhorn

**Setup**: Longhorn deployed across three K3s nodes with dedicated storage devices.

**Performance**:
- Sequential Read: 220 MB/s
- Sequential Write: 110 MB/s
- Random 4K Read: 4,500 IOPS
- Random 4K Write: 1,200 IOPS

**Pros**:
- Built-in replication
- Snapshot and backup support
- Nice web UI
- Good integration with Kubernetes

**Cons**:
- Significant resource overhead
- Complex setup for optimal performance
- Occasional stability issues with large volumes

### 4. OpenEBS with cStor

**Setup**: OpenEBS with cStor pools spread across three nodes with JBOD disks.

**Performance**:
- Sequential Read: 180 MB/s
- Sequential Write: 95 MB/s
- Random 4K Read: 3,800 IOPS
- Random 4K Write: 950 IOPS

**Pros**:
- Data replication
- Snapshot and clone support
- ZFS features without external ZFS server

**Cons**:
- Complex setup and management
- Higher resource usage
- Performance inconsistencies

## My Hybrid Solution

After extensive testing, I settled on a hybrid approach:

1. **NFS via TrueNAS Scale**: For general storage needs, media, and applications without strict performance requirements.

2. **OpenEBS LocalPV**: For database workloads requiring high performance (PostgreSQL, MongoDB).

3. **Longhorn**: For stateful applications requiring replication but with moderate performance needs.

This diagram shows my current implementation:

```
┌───────────────────────────────────────────────────────────┐
│                                                           │
│                     K3s Cluster                           │
│                                                           │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │   Node 1    │    │   Node 2    │    │   Node 3    │    │
│  │             │    │             │    │             │    │
│  │  ┌────────┐ │    │  ┌────────┐ │    │  ┌────────┐ │    │
│  │  │Longhorn│ │    │  │Longhorn│ │    │  │Longhorn│ │    │
│  │  └────────┘ │    │  └────────┘ │    │  └────────┘ │    │
│  │             │    │             │    │             │    │
│  │  ┌────────┐ │    │  ┌────────┐ │    │  ┌────────┐ │    │
│  │  │OpenEBS │ │    │  │OpenEBS │ │    │  │OpenEBS │ │    │
│  │  │LocalPV │ │    │  │LocalPV │ │    │  │LocalPV │ │    │
│  │  └────────┘ │    │  └────────┘ │    │  └────────┘ │    │
│  │             │    │             │    │             │    │
│  └─────────────┘    └─────────────┘    └─────────────┘    │
│                                                           │
└───────────────────────────────────────────────────┬───────┘
                                                    │
                   10GbE Network                    │
                                                    │
┌─────────────────────────────────────────────────┐ │
│                                                 │ │
│              TrueNAS Scale Server               │ │
│                                                 │ │
│  ┌─────────────────────────────────────────┐    │ │
│  │                  ZFS                     │◄───┘
│  │                                         │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │  │   NFS   │ │ Datasets│ │Snapshots│    │
│  │  └─────────┘ └─────────┘ └─────────┘    │
│  └─────────────────────────────────────────┘
│                                             │
└─────────────────────────────────────────────┘
```

## Implementation Details

### Storage Class Configuration

I created multiple storage classes to direct different types of workloads to appropriate storage backends:

```yaml
# NFS Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
# OpenEBS LocalPV Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-local-ssd
  annotations:
    openebs.io/cas-type: local
    cas.openebs.io/config: |
      - name: StorageType
        value: hostpath
      - name: BasePath
        value: /mnt/local-ssd
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# Longhorn Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-replicated
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fromBackup: ""
```

### Usage Guidelines

I follow these guidelines for workload placement:

1. **General Purpose Applications** (ConfigMaps, small persistent data): NFS storage
2. **Database Workloads**: OpenEBS LocalPV (with separate backups to TrueNAS)
3. **Critical Stateful Applications**: Longhorn with 2-3 replicas
4. **Media and Large Files**: Direct NFS mounts

### Backup Strategy

My backup strategy involves multiple layers:

1. **ZFS Snapshots** on TrueNAS (hourly, daily, weekly)
2. **Application-Level Backups** (e.g., PostgreSQL dumps)
3. **Velero** for Kubernetes-aware backups of selected applications
4. **Longhorn Backups** to S3-compatible storage (MinIO running on TrueNAS)

## Performance Tuning

After implementing the hybrid approach, I spent considerable time tuning each storage solution:

### NFS Tuning

```bash
# On TrueNAS
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w sunrpc.tcp_slot_table_entries=128

# On K3s nodes
echo "nfs.server.mount.min_version=4.2" >> /etc/modprobe.d/nfs.conf
echo "options sunrpc tcp_slot_table_entries=128" >> /etc/modprobe.d/sunrpc.conf
```

### Longhorn Tuning

I adjusted the following settings in Longhorn:

- Reserved 5GB memory per node for Longhorn
- Increased default replica count to 2
- Set storage over-provisioning percentage to 100%
- Enabled ReadWriteMany support for applicable workloads

### OpenEBS LocalPV

```bash
# On each K3s node with SSD
echo 'mq-deadline' > /sys/block/sda/queue/scheduler
echo '8192' > /sys/block/sda/queue/read_ahead_kb
echo '2' > /sys/block/sda/queue/rotational
```

## Lessons Learned

After six months of running this hybrid setup, here are my key takeaways:

1. **No Single Solution**: No single storage solution fits all workloads in a homelab
2. **Performance Testing**: Always test with real workloads, not just synthetic benchmarks
3. **Backup First**: Ensure reliable backups before committing to any storage solution
4. **Resource Overhead**: Consider the CPU/memory impact of storage solutions
5. **Failure Testing**: Regularly test node failures to ensure data integrity

## Future Plans

I'm considering the following improvements:

1. Evaluating **Rook/Ceph** for a more integrated storage solution
2. Implementing **S3 object storage** directly in Kubernetes
3. Testing **NVMe over TCP** for higher performance shared storage
4. Moving to a dedicated **storage fabric network** for increased throughput

## Conclusion

The hybrid storage approach has been working extremely well in my homelab environment. Each storage solution serves its specific purpose, and the combination provides a good balance of performance, reliability, and ease of management.

If you're setting up Kubernetes in your homelab, I recommend testing multiple storage options rather than trying to force a single solution to handle all your needs.

---

*What storage solutions are you using in your Kubernetes homelab? I'd love to hear about your experiences in the comments below.*
