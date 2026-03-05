---
title: HA ZFS-over-iSCSI SAN — Ansible Deployment
---

Two-node active/passive storage cluster with quorum, deploying ZFS mirroring over iSCSI, Pacemaker/Corosync failover, floating VIPs for NFS/SMB/iSCSI, STONITH fencing, and Prometheus monitoring.

**Supported OS:** Debian 12 · Ubuntu 22.04/24.04 · Rocky Linux 9 · AlmaLinux 9

---

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

Each storage node exports its local disks via LIO iSCSI to the peer, and imports the peer's target. ZFS mirrors local physical disks with the remote iSCSI disks. On failover, the surviving node imports the pool in degraded state and resumes service. The pool re-silvers to full redundancy when the peer reconnects.

**Failover times:** ~5–8s planned · ~10–12s unplanned

---

## Quick Start

```bash
# 1. Configure inventory and variables
vim inventory.yml                          # Set hostnames and IPs
vim group_vars/all.yml                     # Cluster name, VLANs, VIPs, SSH key
vim group_vars/storage_nodes/cluster.yml   # STONITH config
vim group_vars/storage_nodes/iscsi.yml     # CHAP credentials
vim host_vars/storage-a.yml               # Disk devices, IPs
vim host_vars/storage-b.yml               # Disk devices, IPs

# 2. Vault secrets
ansible-vault encrypt_string 'your-password' --name 'hacluster_password'
ansible-vault encrypt_string 'your-chap-pass' --name 'iscsi_chap_password'

# 3. Deploy
ansible-playbook -i inventory.yml site.yml --ask-vault-pass

# 4. Manual steps (SSH to storage-a)
iscsiadm -m session                        # Verify iSCSI sessions
vim /root/create-pool.sh                   # Fix REMOTE_DISKS paths
bash /root/create-pool.sh
zpool export san-pool
bash /root/configure-stonith.sh
bash /root/configure-pacemaker-resources.sh
```

---

## Playbook Tags

```bash
ansible-playbook -i inventory.yml site.yml --tags base       # OS + hardening
ansible-playbook -i inventory.yml site.yml --tags storage    # ZFS + iSCSI
ansible-playbook -i inventory.yml site.yml --tags cluster    # Pacemaker
ansible-playbook -i inventory.yml site.yml --tags services   # NFS/SMB configs
ansible-playbook -i inventory.yml site.yml --tags cockpit    # Houston UI
ansible-playbook -i inventory.yml site.yml --tags monitoring # exporters
ansible-playbook -i inventory.yml site.yml --check --diff    # dry run
```

---

## What's Automated vs. Manual

| Step | Automated | Manual | Why |
|------|:---------:|:------:|-----|
| OS packages + repos | ✓ | | Deterministic |
| Security hardening | ✓ | | Deterministic |
| ZFS installation | ✓ | | Deterministic |
| LIO target setup | ✓ | | Per-host config |
| open-iscsi setup | ✓ | | Per-host config |
| Corosync/Pacemaker install | ✓ | | Deterministic |
| Cluster formation | ✓ | | Idempotent |
| NFS/SMB/iSCSI config files | ✓ | | Templates |
| ZFS pool creation | | ✓ | iSCSI paths vary |
| STONITH configuration | | ✓ | Destructive — test first |
| Pacemaker resources | | ✓ | Depends on pool existing |

---

## Monitoring

Exporters deployed on all nodes:

| Exporter | Port | Metrics |
|----------|------|---------|
| node_exporter | 9100 | CPU, memory, disk, network |
| ha_cluster_exporter | 9664 | Pacemaker/Corosync health |
| zfs-scrub-exporter | 9100 (textfile) | Scrub state, pool health, vdev errors, resilver |
| stonith-probe | 9100 (textfile) | Fence agent reachability |
| reboot-required-exporter | 9100 (textfile) | Pending reboot flag |

```bash
curl http://10.20.20.1:9100/metrics | grep zfs_scrub
curl http://10.20.20.1:9664/metrics | grep ha_cluster
```

---

## Roles

| Role | Purpose |
|------|---------|
| `common` | Base packages, chrony NTP, `/etc/hosts` |
| `hardening` | nftables firewall, SSH hardening, sysctl, PAM faillock, auditd, watchdog |
| `zfs` | ZFS install, ARC tuning, modprobe options, Sanoid snapshots |
| `iscsi-target` | LIO iSCSI target (backend disk replication to peer) |
| `iscsi-initiator` | open-iscsi initiator + `create-pool.sh` generation |
| `pacemaker` | Corosync + Pacemaker cluster auth, config, STONITH scripts |
| `services` | NFS, SMB, iSCSI client target; shared config dir on ZFS |
| `monitoring` | node_exporter, ha_cluster_exporter, ZFS scrub, STONITH probe, reboot exporter |
| `cockpit` | Cockpit + 45Drives Houston plugins |
| `networking` | Interface/VLAN config via systemd-networkd (opt-in) |

---

[View on GitHub](https://github.com/vipervire/ha-san-ansible)
