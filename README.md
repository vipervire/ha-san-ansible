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
```

## Customization Points

- **Disk layout**: Edit `local_data_disks` in `host_vars/` for your drives
- **VLANs/subnets**: Edit `vlans` dict in `group_vars/all.yml`
- **STONITH method**: Switch between `ipmi` and `smart_plug` in `storage_nodes.yml`
- **Snapshot policy**: Edit `sanoid_templates` in `storage_nodes.yml`
- **SMB shares**: Add entries to `smb_shares` list
- **NFS exports**: Add entries to `nfs_exports` list
- **iSCSI zvols**: Create manually with `zfs create -V <size> san-pool/iscsi/<name>`
