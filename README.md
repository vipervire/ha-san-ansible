# HA ZFS-over-iSCSI SAN — Ansible Deployment

Two-node active/passive storage cluster with quorum, deploying:
- ZFS mirroring over iSCSI for cross-node redundancy
- Pacemaker/Corosync with aggressive-safe failover (~10-12s unplanned, ~5-8s planned)
- Floating VIPs for NFS, SMB, and iSCSI client services
- STONITH fencing (IPMI or smart plug)
- Security hardening (nftables, SSH, sysctl)
- 45Drives Houston Cockpit plugins for web management
- Sanoid automated snapshots
- Prometheus node_exporter for monitoring

## Architecture

```
┌─────────────────┐     iSCSI/40GbE      ┌─────────────────┐
│   storage-a     │◄────────────────────►│   storage-b     │
│   (ACTIVE)      │    VLAN 10 / MTU 9000 │   (STANDBY)     │
│                 │                       │                 │
│  12× 1TB SSDs   │                       │  12× 1TB SSDs   │
│  LIO target     │                       │  LIO target     │
│  open-iscsi     │                       │  open-iscsi     │
│  ZFS pool       │                       │  (ready)        │
│  NFS/SMB/iSCSI  │                       │                 │
│  Pacemaker      │◄──── Corosync ──────►│  Pacemaker      │
└────────┬────────┘                       └────────┬────────┘
         │              ┌──────────┐               │
         └──────────────┤  quorum  ├───────────────┘
                        │  (voter) │
                        └──────────┘
```

## Prerequisites

1. **Three hosts** running Debian 12 minimal (fresh install)
2. **SSH key access** as `storageadmin` with passwordless sudo
3. **Network connectivity** on management VLAN between all nodes
4. **Ansible 2.14+** on your control machine

```bash
# On your control machine
pip install ansible
ansible-galaxy collection install community.general
```

## Quick Start

```bash
# 1. Clone/copy this directory
# 2. Edit inventory and variables:
vim inventory.yml          # Set hostnames and IPs
vim group_vars/all.yml     # Cluster name, VLANs, VIPs, SSH key
vim group_vars/storage_nodes.yml  # STONITH, CHAP credentials
vim host_vars/storage-a.yml      # Disk devices, IPs
vim host_vars/storage-b.yml      # Disk devices, IPs

# 3. Vault your secrets (recommended)
ansible-vault encrypt_string 'your-password' --name 'hacluster_password'
ansible-vault encrypt_string 'your-chap-pass' --name 'iscsi_chap_password'

# 4. Run the playbook
ansible-playbook -i inventory.yml site.yml --ask-vault-pass

# 5. Manual steps (SSH to storage-a):
#    a. Verify iSCSI sessions:
iscsiadm -m session
#    b. Edit and run pool creation:
vim /root/create-pool.sh   # Fix REMOTE_DISKS paths
bash /root/create-pool.sh
#    c. Export pool for Pacemaker:
zpool export san-pool
#    d. Configure STONITH:
bash /root/configure-stonith.sh
#    e. Configure Pacemaker resources:
bash /root/configure-pacemaker-resources.sh
```

## What's Automated vs. Manual

| Step | Automated | Manual | Why |
|------|-----------|--------|-----|
| OS packages + repos | ✅ | | Deterministic |
| Security hardening | ✅ | | Deterministic |
| ZFS installation | ✅ | | Deterministic |
| LIO target setup | ✅ | | Per-host config |
| open-iscsi setup | ✅ | | Per-host config |
| Corosync/Pacemaker install | ✅ | | Deterministic |
| Cluster formation | ✅ | | Idempotent |
| NFS/SMB/iSCSI config files | ✅ | | Templates |
| ZFS pool creation | | ✅ | iSCSI paths vary |
| STONITH configuration | | ✅ | Destructive — test first |
| Pacemaker resources | | ✅ | Depends on pool existing |

The manual steps are deliberate. Pool creation requires verifying that iSCSI
device paths (`/dev/disk/by-path/...`) match between what the playbook expects
and what the kernel actually assigned. STONITH configuration is kept manual
because a misconfigured fencing agent can power-cycle your nodes unexpectedly.

## Tags

```bash
# Run specific phases:
ansible-playbook -i inventory.yml site.yml --tags base       # OS + hardening
ansible-playbook -i inventory.yml site.yml --tags storage    # ZFS + iSCSI
ansible-playbook -i inventory.yml site.yml --tags cluster    # Pacemaker
ansible-playbook -i inventory.yml site.yml --tags services   # NFS/SMB configs
ansible-playbook -i inventory.yml site.yml --tags cockpit    # Houston UI
ansible-playbook -i inventory.yml site.yml --tags monitoring # node_exporter
```

## Directory Structure

```
ha-san-ansible/
├── site.yml                    # Main playbook
├── inventory.yml               # Host inventory
├── group_vars/
│   ├── all.yml                 # Cluster-wide variables
│   └── storage_nodes.yml       # Storage-specific variables
├── host_vars/
│   ├── storage-a.yml           # Node A disks, IPs
│   ├── storage-b.yml           # Node B disks, IPs
│   └── quorum.yml              # Quorum node config
└── roles/
    ├── common/                 # Base packages, NTP, /etc/hosts
    ├── hardening/              # SSH, nftables, sysctl
    ├── zfs/                    # ZFS install, tunables, sanoid
    ├── iscsi-target/           # LIO targetcli setup
    ├── iscsi-initiator/        # open-iscsi + pool creation helper
    ├── pacemaker/              # Corosync + Pacemaker cluster
    ├── services/               # NFS, SMB config files
    ├── cockpit/                # Cockpit + 45Drives Houston
    └── monitoring/             # Prometheus node_exporter
```

## Post-Deployment Testing

After the manual steps are complete, run through this checklist:

```bash
# Verify cluster health
pcs status

# Test planned failover (~5-8s)
pcs resource move san-resources storage-b
pcs resource clear san-resources

# Test unplanned failover (~10-12s) — PULL THE POWER CORD on active node
# Verify clients reconnect within 15-25s total

# Test STONITH
stonith_admin -t fence_ipmilan -r -F storage-b  # dry run

# Verify resilver after recovery
zpool status san-pool

# Verify ZFS scrub automation
systemctl status zfs-scrub@san-pool.timer
systemctl list-timers zfs-scrub@san-pool.timer

# Manually trigger a scrub (safe to test)
systemctl start zfs-scrub@san-pool.service
journalctl -u zfs-scrub@san-pool.service -f

# Check scrub progress
zpool status san-pool
```

## Customization Points

- **Disk layout**: Edit `local_data_disks` in `host_vars/` for your drives
- **VLANs/subnets**: Edit `vlans` dict in `group_vars/all.yml`
- **STONITH method**: Configure per-node in `stonith_nodes` dict in `storage_nodes.yml`
  - Supports mixed methods: storage-a can use IPMI while storage-b uses smart plug
  - Smart plug guide: `docs/stonith-smart-plugs.md` (TP-Link Kasa, ESPHome, Tasmota)
  - Migration guide: `docs/stonith-migration.md` (upgrading from old config format)
- **Snapshot policy**: Edit `sanoid_templates` in `storage_nodes.yml`
- **ZFS scrub schedule**: Edit `zfs_scrub_schedule` in `storage_nodes.yml` (default: monthly on 1st at 2 AM)
  - Use systemd OnCalendar syntax: `"*-*-01 02:00:00"` = 1st of month at 2am
  - Disable with `zfs_scrub_enabled: false`
  - Monitoring: See `docs/ntfy-integration.md` for Prometheus + NTFY alerting setup
- **SMB shares**: Add entries to `smb_shares` list
- **NFS exports**: Add entries to `nfs_exports` list
- **iSCSI zvols**: Create manually with `zfs create -V <size> san-pool/iscsi/<name>`

## Monitoring and Alerting

This playbook deploys comprehensive monitoring for both storage and cluster health:

### Node-Level Metrics (node_exporter:9100)
- CPU, memory, disk, network, systemd services
- Deployed on all nodes (storage-a, storage-b, quorum)

### ZFS Storage Metrics (custom exporter, updated every 5 min)
- `zfs_scrub_last_run_timestamp_seconds` - When last scrub completed
- `zfs_scrub_last_run_errors_total` - Errors found in last scrub
- `zfs_scrub_in_progress` - Whether scrub is currently running
- `zfs_scrub_pool_health` - Pool health status (0=ONLINE, 1=DEGRADED, etc.)
- `zfs_scrub_pool_imported` - Whether pool is imported on this node

### Cluster Health Metrics (ha_cluster_exporter:9664, updated every 30 sec)
- `ha_cluster_corosync_quorate` - Cluster quorum status
- `ha_cluster_pacemaker_nodes` - Node online/offline status
- `ha_cluster_pacemaker_resources` - Resource health and location
- `ha_cluster_pacemaker_fail_count` - Resource failure counts
- `ha_cluster_pacemaker_stonith_enabled` - STONITH status
- `ha_cluster_corosync_rings` - Corosync ring health

**Documentation**:
- Cluster monitoring guide: `docs/cluster-monitoring.md`
- STONITH smart plug setup: `docs/stonith-smart-plugs.md`
- Example Prometheus alert rules: `docs/prometheus-alerts.yml`
- NTFY integration guide: `docs/ntfy-integration.md`

**Alert Coverage**:
- Storage: overdue scrubs, scrub errors, pool degradation, split-brain
- Cluster: quorum loss, node offline, resource failures, STONITH status, failover detection

**Grafana Dashboard**: Import dashboard #12229 from Grafana.com for pre-built cluster visualization.

**Quick Check**:
```bash
# View ZFS metrics
curl http://10.20.20.1:9100/metrics | grep zfs_scrub

# View cluster metrics
curl http://10.20.20.1:9664/metrics | grep ha_cluster

# Check exporters
systemctl status zfs-scrub-exporter.timer
systemctl status prometheus-hacluster-exporter
```
