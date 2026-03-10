# HA ZFS-over-iSCSI SAN — Ansible Deployment

Two-node active/passive storage cluster with quorum, deploying:
- ZFS mirroring over iSCSI for cross-node redundancy
- Pacemaker/Corosync with aggressive-safe failover (~10-12s unplanned, ~5-8s planned)
- Floating VIPs for NFS, SMB, and iSCSI client services
- STONITH fencing (IPMI, Kasa, Tasmota, ESPHome, HTTP)
- Security hardening (nftables, SSH, sysctl)
- 45Drives Houston Cockpit plugins for web management
- Sanoid automated snapshots
- Prometheus monitoring (node, ZFS, cluster, STONITH exporters)

## Architecture

```
┌─────────────────┐     iSCSI/40GbE       ┌─────────────────┐
│   storage-a     │◄─────────────────────►│   storage-b     │
│   (ACTIVE)      │    VLAN 10 / MTU 9000 │   (STANDBY)     │
│                 │                       │                 │
│  12× 1TB SSDs   │                       │  12× 1TB SSDs   │
│  LIO target     │                       │  LIO target     │
│  open-iscsi     │                       │  open-iscsi     │
│  ZFS pool       │                       │  (ready)        │
│  NFS/SMB/iSCSI  │                       │                 │
│  Pacemaker      │◄──── Corosync ───────►│  Pacemaker      │
└────────┬────────┘                       └────────┬────────┘
         │              ┌──────────┐               │
         └──────────────┤  quorum  ├───────────────┘
                        │  (voter) │
                        └──────────┘
```

## Prerequisites

1. **Three hosts** running Debian 12, Ubuntu 22.04/24.04, Rocky Linux 9, or AlmaLinux 9 minimal (fresh install)
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
vim inventory.yml                          # Set hostnames and IPs
vim group_vars/all.yml                     # Cluster name, VLANs, VIPs, SSH key
vim group_vars/storage_nodes/cluster.yml   # STONITH config
vim group_vars/storage_nodes/iscsi.yml     # CHAP credentials
vim host_vars/storage-a.yml               # Disk devices, IPs
vim host_vars/storage-b.yml               # Disk devices, IPs

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
ansible-playbook -i inventory.yml site.yml --tags monitoring # monitoring exporters
```

## Directory Structure

```
ha-san-ansible/
├── site.yml                    # Main playbook
├── os-upgrade.yml              # Rolling OS upgrade helper (always use --limit)
├── verify.yml                  # Post-deployment verification
├── inventory.yml               # Host inventory
├── group_vars/
│   ├── all.yml                 # Cluster-wide variables
│   └── storage_nodes/
│       ├── cluster.yml         # STONITH config, Pacemaker tuning
│       ├── iscsi.yml           # iSCSI CHAP credentials, queue depth
│       ├── network.yml         # Interface names, TCP tuning
│       ├── services.yml        # NFS exports, SMB shares
│       └── zfs.yml             # ZFS pool, datasets, scrub, Sanoid
├── host_vars/
│   ├── storage-a.yml           # Node A disks, IPs, IQNs
│   ├── storage-b.yml           # Node B disks, IPs, IQNs
│   └── quorum.yml              # Quorum node config
└── roles/
    ├── common/                 # Base packages, NTP, /etc/hosts
    ├── hardening/              # SSH, nftables, sysctl, PAM
    ├── zfs/                    # ZFS install, tunables, Sanoid
    ├── iscsi-target/           # LIO targetcli setup
    ├── iscsi-initiator/        # open-iscsi + pool creation helper
    ├── pacemaker/              # Corosync + Pacemaker cluster
    ├── services/               # NFS, SMB, iSCSI client config
    ├── cockpit/                # Cockpit + 45Drives Houston
    ├── monitoring/             # Monitoring exporters (node, ZFS, cluster, STONITH)
    └── networking/             # Interface/VLAN config (opt-in)
```

## Post-Deployment Testing

After the manual steps are complete, run through this checklist:

```bash
# Verify cluster health
pcs status

# Test planned failover (~5-8s)
pcs resource move san-services storage-b
pcs resource clear san-services

# Test unplanned failover (~10-12s) — PULL THE POWER CORD on active node
# Verify clients reconnect within 15-25s total

# Test STONITH (WARNING: this WILL power off the node — only run in maintenance)
pcs stonith fence storage-b

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

## Cockpit Web Interface

Access Cockpit at:
- **Active node VIP:** `https://10.20.20.10:9090` (recommended)
- Direct node access: `https://10.20.20.1:9090` (storage-a) or `https://10.20.20.2:9090` (storage-b)

**Features:**
- 45Drives Houston plugins for ZFS, NFS, SMB management
- File browser (cockpit-navigator)
- User/group management (cockpit-identities)
- System monitoring

**Configuration sync:**
- NFS exports: `/etc/exports` → symlinked to `/san-pool/cluster-config/nfs/exports`
- SMB config: `/etc/samba/smb.conf` → symlinked to `/san-pool/cluster-config/samba/smb.conf`
- Samba users: `/var/lib/samba/private/` → symlinked to `/san-pool/cluster-config/samba/private/`
- Changes via Cockpit automatically sync between nodes during failover ✅

**Documentation:** See `docs/cockpit-ha-config.md` for detailed HA configuration guide, troubleshooting, and verification steps.

## Customization Points

- **Disk layout**: Edit `local_data_disks` in `host_vars/` for your drives
- **VLANs/subnets**: Edit `vlans` dict in `group_vars/all.yml`
- **STONITH method**: Configure per-node in `stonith_nodes` dict in `group_vars/storage_nodes/cluster.yml`
  - Supports mixed methods: storage-a can use IPMI while storage-b uses smart plug
  - Smart plug guide: `docs/stonith-smart-plugs.md` (TP-Link Kasa, ESPHome, Tasmota)
- **Snapshot policy**: Edit `sanoid_templates` in `group_vars/storage_nodes/zfs.yml`
- **ZFS scrub schedule**: Edit `zfs_scrub_schedule` in `group_vars/storage_nodes/zfs.yml` (default: monthly on 1st at 2 AM)
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
- `zfs_pool_size_bytes`, `zfs_pool_allocated_bytes`, `zfs_pool_free_bytes` - Pool capacity
- `zfs_pool_fragmentation_percent` - Pool fragmentation percentage
- `zfs_vdev_read_errors`, `zfs_vdev_write_errors`, `zfs_vdev_cksum_errors` - Per-vdev I/O errors
- `zfs_resilver_in_progress`, `zfs_resilver_percent_complete` - Resilver status
- `zfs_dataset_last_snapshot_seconds` - Timestamp of most recent Sanoid snapshot per dataset

### Hardware Health Metrics (custom exporters, updated every 5 min)
- `node_disk_smart_*` - SMART disk health: healthy status, temperature, reallocated sectors, power-on hours, NVMe-specific metrics (smart-exporter, storage nodes only)
- `node_memory_correctable_errors_total`, `node_memory_uncorrectable_errors_total` - RAS/ECC hardware memory errors (ras-exporter, storage nodes only)
- `node_nic_temperature_celsius`, `node_nic_sfp_temperature_celsius`, `node_hba_temperature_celsius` - NIC/HBA hardware temperatures (hwtemp-exporter, storage nodes only)

### STONITH Probe Metrics (stonith-probe exporter, updated every 2 min)
- `stonith_agent_reachable` - Whether each fence agent IP is pingable (storage nodes only)

### OS Reboot Metrics (reboot-required exporter, updated every 15 min)
- `node_reboot_required` - Whether a system reboot is pending (all nodes)

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
- Hardware/software watchdog: `docs/watchdog.md`
- Ubuntu/AlmaLinux-specific notes: `docs/ubuntu-notes.md`
- Rolling OS upgrade guide: `docs/os-upgrade.md`
- iSCSI path recovery: `docs/iscsi-recovery.md`
- NFS client configuration: `docs/nfs-client-config.md`
- Cockpit HA configuration: `docs/cockpit-ha-config.md`
- Example Prometheus alert rules: `docs/prometheus-alerts.yml`
- Prometheus recording rules: `docs/prometheus-recording-rules.yml`
- NTFY integration + dead-man's switch: `docs/ntfy-integration.md`
- NFS authentication levels: `docs/nfs-security.md`
- ZFS dataset configuration by workload: `docs/dataset-best-practices.md`

**Alert Coverage**:
- Storage: overdue scrubs, scrub errors, pool degradation, split-brain, capacity, vdev errors, resilver stalls, snapshot age
- Cluster: quorum loss, node offline, resource failures, stuck resource transitions, STONITH unreachable, failover detection
- Services: NFS/SMB resource stopped
- OS: reboot required, memory pressure
- Pipeline: Watchdog dead-man's switch (Uptime Kuma)

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
systemctl status stonith-probe.timer
systemctl status reboot-required-exporter.timer
```

## Rolling OS Upgrade

Use `os-upgrade.yml` to upgrade one node at a time with automated health checks and failover handling. See `docs/os-upgrade.md` for the full procedure.

```bash
# Pre-upgrade: safety checks, standby, failover (if active node)
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit storage-b

# (Upgrade the OS manually, then re-apply Ansible)
ansible-playbook -i inventory.yml site.yml --limit storage-b

# Post-upgrade: verify services, iSCSI, rejoin cluster
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit storage-b
```

Upgrade order: quorum → standby storage node → active storage node.

## Re-Deploying and Rollback

Almost all tasks in this playbook are idempotent. Re-running with a tag is the standard
rollback/re-apply mechanism after manual changes:

```bash
# Re-apply hardening after a manual change (firewall, sysctl, SSH config)
ansible-playbook -i inventory.yml site.yml --tags base

# Re-apply ZFS tuning (ARC, txg_timeout, scrub schedule)
ansible-playbook -i inventory.yml site.yml --tags storage

# Re-apply cluster config (corosync.conf, STONITH script, pcs properties)
ansible-playbook -i inventory.yml site.yml --tags cluster

# Re-apply NFS/SMB config files (exports, smb.conf, iscsid.conf)
ansible-playbook -i inventory.yml site.yml --tags services

# Re-apply monitoring exporters
ansible-playbook -i inventory.yml site.yml --tags monitoring

# Dry run before applying — always safe
ansible-playbook -i inventory.yml site.yml --check --diff --tags <role>
```

**What is NOT idempotent (manual steps):**
- `create-pool.sh` — pool creation is irreversible; run once after verifying disk paths
- `configure-stonith.sh` — regenerated by Ansible but must be manually run; test fencing
  before running in production
- `configure-pacemaker-resources.sh` — regenerated by Ansible; `pcs` commands use
  `|| true` guards to skip already-existing resources

**Undoing a manual pcs change:**
```bash
# Clear all resource constraints (returns to Ansible-defined state)
pcs constraint remove --all  # then re-run configure-pacemaker-resources.sh

# Reset resource failure counts
pcs resource cleanup <resource>

# Remove a resource (if accidentally misconfigured)
pcs resource remove <resource>  # then re-run configure-pacemaker-resources.sh
```
