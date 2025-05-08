---
layout: post
title: "Optimizing Proxmox and TrueNAS ZFS Integration in a Homelab"
date: 2025-05-05
categories: [homelab, storage, virtualization]
tags: [proxmox, truenas, zfs, performance, iscsi]
---

# Optimizing Proxmox and TrueNAS ZFS Integration in a Homelab

After months of experimentation, I've finally landed on a stable, high-performance integration between my Proxmox virtualization cluster and TrueNAS Scale storage server. This post documents my configuration, performance findings, and lessons learned.

## The Infrastructure Setup

My current homelab storage infrastructure consists of:

- **TrueNAS Scale**: Running on a dedicated physical server
  - CPU: AMD Ryzen 9 5950X
  - RAM: 128GB ECC DDR4-3200
  - Boot: 2x 500GB NVMe in mirror
  - Storage Pool: 8x 16TB CMR drives in RAIDZ2
  - Cache: 2x 2TB NVMe (special metadata vdev)
  - Network: 2x 10GbE SFP+ (LACP)

- **Proxmox Cluster**: 3 identical nodes
  - CPU: AMD Ryzen 7 5800X
  - RAM: 64GB DDR4-3600
  - Local Storage: 1TB NVMe for OS/local VMs
  - Network: 10GbE SFP+ for storage traffic

## Storage Connectivity Options Tested

I've experimented with multiple connectivity methods between Proxmox and TrueNAS:

1. **NFS Shares**
2. **iSCSI LUNs**
3. **ZFS over iSCSI**
4. **Direct ZFS Replication**

### Performance Comparison

After extensive testing, here are my findings (all numbers in MB/s with 4K random workloads):

| Method | Read | Write | Latency (ms) | Stability Notes |
|--------|------|-------|--------------|----------------|
| NFS | 525 | 320 | 2.8 | Occasional lock contention |
| iSCSI (zvol) | 890 | 760 | 1.2 | Very stable, best overall |
| ZFS Replication | 780 | 730 | 0.8 | Complex management |

## The Optimal Configuration: iSCSI with ZVOLs

After thorough testing, I've settled on using iSCSI with ZVOLs as the primary storage connectivity method. Here's my configuration:

### TrueNAS ZFS Dataset Structure

```
tank/
├── proxmox/
│   ├── vm-storage/
│   │   ├── production.zvol
│   │   ├── development.zvol
│   │   └── testing.zvol
│   └── backup/
│       └── daily-mirror/
```

### Key ZFS Tuning Parameters

```bash
# On TrueNAS for the zvols
zfs set sync=disabled tank/proxmox/vm-storage/production
zfs set volblocksize=64K tank/proxmox/vm-storage/production
zfs set primarycache=metadata tank/proxmox/vm-storage/production
zfs set compression=lz4 tank/proxmox/vm-storage/production

# On Proxmox
zfs set atime=off rpool
echo "options zfs zfs_arc_max=16G" > /etc/modprobe.d/zfs.conf
```

### iSCSI Target Configuration

I've configured TrueNAS to expose these ZVOLs as iSCSI targets with the following settings:

- **Block Size**: 64K (matching ZFS volblocksize)
- **I/O Queue Depth**: 64
- **Multi-path**: Enabled across both 10GbE interfaces
- **CHAP Authentication**: Enabled for security

### Network Configuration

The network configuration is critical for optimal performance:

- **Dedicated VLANs** for iSCSI traffic
- **Jumbo Frames** (MTU 9000) on all storage interfaces
- **LACP Bonding** on TrueNAS 10GbE interfaces
- **Flow Control** enabled on switches and endpoints

## Benchmarking & Tuning Results

When running VMs on the iSCSI targets, I initially encountered performance issues. After tuning, I achieved substantial improvements:

| VM Storage | Before Tuning | After Tuning | % Improvement |
|------------|---------------|--------------|--------------|
| Random 4K Read IOPS | 45,000 | 112,000 | 149% |
| Random 4K Write IOPS | 12,000 | 38,000 | 217% |
| Sequential Read | 780 MB/s | 1,120 MB/s | 44% |
| Sequential Write | 620 MB/s | 980 MB/s | 58% |

### Key Tuning Changes That Made a Difference

1. **Switched from Virtio-SCSI to Virtio-Block** for VM disk presentation
2. **Increased AIO threads** in QEMU configuration
3. **Tuned ZFS recordsize** to match VM workload patterns
4. **Implemented proper multipathing** for iSCSI connections
5. **Adjusted sysctl parameters** for network stack performance

## Backup Strategy

With this setup, I've implemented a comprehensive backup strategy:

1. **ZFS Snapshots**: Hourly, daily, and weekly snapshots of the ZVOLs
2. **Replication**: Daily ZFS send/receive to a backup pool
3. **Proxmox Backup Server**: Weekly full VM backups independent of storage

The following script handles my automated snapshot rotation:

```bash
#!/bin/bash
# ZFS snapshot rotation script for TrueNAS

# Hourly snapshots (keep 24)
zfs snapshot -r tank/proxmox@hourly-$(date +%Y%m%d%H)
find_old_hourly=$(zfs list -t snapshot -o name -s creation | grep hourly | head -n -24)
if [ ! -z "$find_old_hourly" ]; then
  echo "$find_old_hourly" | xargs -n 1 zfs destroy
fi

# Daily snapshots (keep 30)
if [ $(date +%H) -eq 0 ]; then
  zfs snapshot -r tank/proxmox@daily-$(date +%Y%m%d)
  find_old_daily=$(zfs list -t snapshot -o name -s creation | grep daily | head -n -30)
  if [ ! -z "$find_old_daily" ]; then
    echo "$find_old_daily" | xargs -n 1 zfs destroy
  fi
fi

# Weekly snapshots (keep 12)
if [ $(date +%u) -eq 7 ] && [ $(date +%H) -eq 0 ]; then
  zfs snapshot -r tank/proxmox@weekly-$(date +%Y%m%d)
  find_old_weekly=$(zfs list -t snapshot -o name -s creation | grep weekly | head -n -12)
  if [ ! -z "$find_old_weekly" ]; then
    echo "$find_old_weekly" | xargs -n 1 zfs destroy
  fi
fi
```

## Monitoring and Alerting

For comprehensive monitoring, I've set up:

1. **Prometheus + Grafana** for metrics visualization
2. **ZFS monitoring with custom thresholds** for proactive alerts
3. **SMART data collection** for physical disk health monitoring

Here's a sample of the alerts I've configured:

```yaml
groups:
- name: zfs-health
  rules:
  - alert: ZpoolStatusNotHealthy
    expr: zpool_status_health{pool="tank"} > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "ZFS pool not healthy"
      description: "Pool 'tank' health status is degraded"

  - alert: ZfsHighFragmentation
    expr: zfs_fragmentation_percent{dataset=~"tank/proxmox/.*"} > 50
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "ZFS fragmentation high"
      description: "Dataset {{ $labels.dataset }} is experiencing high fragmentation"
  
  - alert: ZfsLowFreeSpace
    expr: zfs_available_bytes / zfs_size_bytes < 0.15
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "ZFS free space low"
      description: "Less than 15% free space on {{ $labels.dataset }}"
```

## Lessons Learned

After six months of running this configuration, here are my key takeaways:

1. **Proper network design is critical** - Most performance issues traced back to network configuration
2. **ZFS tuning should match workload** - Generic tuning guides rarely provide optimal results
3. **iSCSI multipathing is worth the effort** - Provides both redundancy and performance benefits
4. **Benchmark, benchmark, benchmark** - Never assume performance based on configurations

## Future Improvements

I'm considering the following upgrades to this setup:

1. **Moving to NVMe-oF** (NVMe over Fabrics) instead of iSCSI
2. **Implementing 25GbE networking** for storage traffic
3. **Exploring ZFS native encryption** for VM storage
4. **Testing bcachefs** as a potential alternative to ZFS

## Conclusion

The combination of Proxmox and TrueNAS with ZFS provides an extremely capable homelab infrastructure with enterprise-class features. While the initial setup and tuning require significant effort, the resulting performance, reliability, and data protection capabilities make it well worth the investment.

If you're running a similar setup, I'd love to hear about your experiences, especially regarding NVMe-oF implementations or alternative storage configurations.

---

*What storage configuration are you using in your homelab? Share your experiences in the comments below.*
