# Rolling OS Upgrade Procedure

> **Automated helper available:** The `os-upgrade.yml` playbook automates pre-upgrade safety
> checks, node standby/failover, and post-upgrade verification. This document explains the
> full procedure; the commands reference the playbook where appropriate.

## Overview

This guide covers rolling OS upgrades for the HA SAN cluster — both in-place dist-upgrades (e.g., `apt full-upgrade`) and full-node reinstalls. The cluster can be upgraded with zero client downtime by upgrading one node at a time.

The `os-upgrade.yml` playbook automates:
- Pre-upgrade cluster health checks (quorum, failed actions, ZFS pool health)
- Placing the target node in Pacemaker standby (triggering graceful failover if needed)
- Post-upgrade service verification (Pacemaker, Corosync, iSCSI target, iSCSI sessions)
- Removing the node from standby and checking pool/resilver status

## Prerequisites

Before starting any upgrade:

1. **Cluster is healthy**: `pcs status` shows all nodes online, no failed actions, quorum established
2. **ZFS pool is ONLINE**: `zpool status san-pool` shows no DEGRADED, FAULTED, or resilver in progress
3. **iSCSI sessions established** on both storage nodes: `iscsiadm -m session`
4. **Backups verified**: Sanoid snapshots are current; optionally trigger a manual Syncoid replication
5. **Maintenance window**: Notify clients of the planned storage failover window

```bash
# Quick pre-flight check (run before starting)
ssh storage-a 'pcs status && zpool status san-pool'
ssh storage-a 'iscsiadm -m session'
ssh storage-b 'iscsiadm -m session'
```

## Upgrade Order

Always upgrade in this order:

| Order | Node | Reason |
|-------|------|--------|
| 1st | **quorum** | No storage role; cluster retains quorum (2/3 nodes stay up) |
| 2nd | **storage-b** (standby) | No active resources; ZFS mirror degrades but clients unaffected |
| 3rd | **storage-a** (active) | Triggers planned failover to storage-b; brief client interruption |

If storage-b is currently the active node (resources running on it), swap the order of steps 2 and 3.

## Dist-Upgrade Workflow

### Step 1: Upgrade quorum (optional but recommended first)

The quorum node is the safest to upgrade since it has no storage role.

```bash
# Pre-upgrade checks and standby
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit quorum
```

SSH to the node and upgrade:

```bash
ssh quorum
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

Wait for the node to come back, then rejoin:

```bash
# Re-apply Ansible configuration
ansible-playbook -i inventory.yml site.yml --limit quorum

# Post-upgrade verification and unstandby
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit quorum
```

---

### Step 2: Upgrade the standby storage node (storage-b)

```bash
# Pre-upgrade checks and standby
# If storage-b has failed resources, either fix them or use -e force_upgrade=true
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit storage-b
```

The playbook confirms storage-b is the standby node (no active resources), then puts it in Pacemaker standby. The ZFS pool continues running on storage-a in degraded mode (local disks only).

```bash
ssh storage-b
sudo apt update && sudo apt full-upgrade -y
# If upgrading ZFS DKMS specifically:
sudo apt install --only-upgrade zfs-dkms zfsutils-linux zfs-zed
sudo reboot
```

Wait for storage-b to come back (~2-3 minutes), then re-apply configuration:

```bash
# Re-apply all roles to storage-b (ZFS, iSCSI, Pacemaker, services, monitoring)
ansible-playbook -i inventory.yml site.yml --limit storage-b

# Post-upgrade verification: verifies services, reconnects iSCSI, removes standby
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit storage-b
```

The post-upgrade play will report whether the pool is resilvering. **Wait for resilver to complete** before proceeding to step 3. Monitor with:

```bash
ssh storage-a 'watch zpool status san-pool'
# Resilver progress shown under "scan:" — typical time: 30-90 minutes depending on pool size
```

---

### Step 3: Upgrade the active storage node (storage-a)

This step triggers a planned failover. Resources migrate to storage-b before the upgrade begins.

```bash
# Pre-upgrade checks and standby — this triggers failover to storage-b
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit storage-a
```

The playbook detects storage-a is the active node, displays the failover warning, waits for your confirmation, then puts it in standby. Resources migrate to storage-b (~5-10s).

Verify services are running from storage-b before upgrading:

```bash
ping 10.30.30.10          # NFS VIP
showmount -e 10.30.30.10  # NFS exports visible
curl -k https://10.20.20.10:9090  # Cockpit VIP
```

```bash
ssh storage-a
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

```bash
# Re-apply configuration to storage-a
ansible-playbook -i inventory.yml site.yml --limit storage-a

# Post-upgrade verification and rejoin
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit storage-a
```

---

### Step 4: Restore preferred resource location (optional)

After storage-a rejoins, resources remain on storage-b (sticky). To migrate them back:

```bash
ssh storage-b 'pcs resource move zfs-pool storage-a'
ssh storage-b 'pcs resource clear zfs-pool'  # remove location constraint after migration
ssh storage-a 'pcs status'
```

Or just let Pacemaker handle it — the resource location preference (`pacemaker_preferred_node: storage-a`) is a soft preference, not a hard pin. Resources will naturally migrate back during the next planned maintenance or failover.

---

## Full Reinstall Workflow

A full reinstall requires the same sequencing as a dist-upgrade, but step 3 (re-apply Ansible) must run the full playbook (not just `--tags storage` or `--tags cluster`):

```bash
# After fresh OS install on the node:

# 1. Run full Ansible deployment on the reinstalled node
ansible-playbook -i inventory.yml site.yml --limit <node>

# 2. For storage nodes: verify iSCSI paths and re-export pool if needed
#    (Pool creation itself is NOT re-run — the pool already exists on disk)
ssh <node> 'iscsiadm -m session'          # confirm iSCSI connected
ssh <node> 'zpool import san-pool'        # import existing pool
ssh <node> 'zpool export san-pool'        # export again for Pacemaker

# 3. Run post-upgrade to rejoin cluster
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit <node>
```

> **Note:** If the node being reinstalled was the active node, Pacemaker will have already
> failed over during the downtime. The pool will have been imported in degraded mode on the
> peer. Do not attempt `zpool import` on the reinstalled node until after the pre-upgrade
> step has confirmed the cluster is stable on the surviving node.

## Quorum Node Procedure

The quorum node has no storage role. The simplified procedure:

```bash
# 1. Check cluster health (2/3 nodes will remain up during quorum node upgrade)
ssh storage-a 'pcs status'

# 2. Run pre-upgrade (puts quorum in standby, reduces votes to 2/3)
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit quorum

# 3. Upgrade and reboot
ssh quorum 'sudo apt update && sudo apt full-upgrade -y && sudo reboot'

# 4. Re-apply configuration
ansible-playbook -i inventory.yml site.yml --limit quorum

# 5. Rejoin cluster (restores full 3/3 quorum)
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit quorum
```

Cluster quorum (2/3 votes) is maintained throughout since both storage nodes remain up.

## What Happens Under the Hood

### Planned failover (upgrading the active node)

1. `pcs node standby storage-a` signals Pacemaker to evacuate all resources from storage-a
2. Pacemaker stops resources on storage-a in reverse dependency order: services → VIPs → ZFS pool
3. ZFS pool is exported cleanly from storage-a (`zpool export san-pool`)
4. Pacemaker imports the pool on storage-b (`zpool import san-pool`) — pool is ONLINE (full mirror)
5. Resources start on storage-b: ZFS pool → VIPs → NFS/SMB/Cockpit
6. Total client interruption: ~5-10 seconds for planned migration

### Storage node standby (upgrading the standby node)

1. `pcs node standby storage-b` puts storage-b in standby — no resources move (none are on storage-b)
2. iSCSI sessions from storage-a to storage-b disconnect (timeout ~5s with `iscsi_replacement_timeout: 5`)
3. ZFS pool on storage-a transitions to DEGRADED (missing mirror leg from storage-b's iSCSI target)
4. Pool continues serving all data from local disks — no client impact
5. After upgrade and rejoin, iSCSI reconnects and ZFS begins resilvering the mirror

### ZFS resilver after rejoin

When a storage node rejoins after an upgrade:
1. iSCSI sessions re-establish (automatically via `open-iscsi` service)
2. ZFS detects the previously-missing vdev is back online
3. ZED triggers resilver — data written during the downtime is copied to the rejoined mirror leg
4. Resilver typically takes 30-90 minutes for a 12-disk pool; monitor with `zpool status san-pool`
5. **Do not upgrade the peer node until resilver is complete** — upgrading during resilver leaves the pool with no redundancy

## Edge Cases

### ZFS DKMS build fails after kernel upgrade

If `zfs-dkms` fails to build against the new kernel:

```bash
# Check DKMS status
dkms status

# Rebuild manually
sudo dkms autoinstall
journalctl -u dkms -n 50

# If build fails, hold the kernel upgrade and investigate:
sudo apt-mark hold linux-image-$(uname -r)
# Open a bug report with the ZFS DKMS error output
```

ZFS DKMS support lags kernel releases by days to weeks. If the new kernel is too recent, temporarily hold the kernel package and re-try after a ZFS point release.

### Corosync version mismatch after dist-upgrade

If upgrading from one Debian major version to another, Corosync protocol versions may differ:

```bash
# Check Corosync version on each node
ssh storage-a 'corosync --version'
ssh storage-b 'corosync --version'
ssh quorum 'corosync --version'

# If a node shows "TOTEM ERROR" or won't form membership, restart Corosync on all nodes:
sudo pcs cluster stop --all
sudo pcs cluster start --all
```

If Corosync config format changed between versions, re-apply the cluster role:

```bash
ansible-playbook -i inventory.yml site.yml --tags cluster --limit <upgraded-node>
```

### iSCSI sessions not reconnecting after reboot

If `iscsiadm -m session` shows no sessions after the node comes back:

```bash
# Manually trigger reconnect
sudo systemctl restart open-iscsi
sudo iscsiadm -m node --loginall=automatic
sudo iscsiadm -m session  # verify sessions now established

# If still no sessions, check the iSCSI target on the peer is running:
ssh storage-a 'sudo targetcli ls /iscsi'  # from storage-b, or vice versa

# Check network connectivity on storage VLAN:
ping 10.10.10.1  # from storage-b (peer storage IP)
```

The `post-upgrade` play automatically detects missing sessions and attempts `iscsiadm -m node --loginall=automatic`. If it still fails, the play exits with an error and manual investigation is needed before unstandby.

### pacemaker-node-standby.service auto-unstandby on boot

The `pacemaker-node-standby.service` systemd unit (deployed by the pacemaker role) puts a node in standby during graceful shutdown and **removes standby on next boot**. This means:

- A reboot during the upgrade will automatically bring the node out of standby
- This is intentional: if the OS upgrade completes cleanly and the node boots up, it will rejoin the cluster
- However, the `post-upgrade` play must still be run to verify iSCSI, run site.yml re-apply, and confirm pool health before proceeding to the next node

To prevent auto-unstandby (e.g., if you want to manually verify before rejoining):

```bash
# After reboot, immediately put back in standby before pacemaker rejoins:
ssh <node> 'sudo pcs node standby <node>'
```

### ZFS pool import fails on failover (multihost=on)

If the surviving node cannot import the pool after failover (STONITH must succeed first):

```bash
# Check if STONITH has confirmed the failed node is off:
ssh storage-b 'pcs stonith status'

# If STONITH failed (node not confirmed off), pool import will be blocked.
# Manually fence if safe to do so:
ssh storage-b 'pcs stonith fence storage-a'

# Then clear the failed resource and allow retry:
ssh storage-b 'pcs resource cleanup zfs-pool'
```

## Rollback

If an upgrade breaks a node and it cannot rejoin the cluster:

1. **Put the node in standby** (if Pacemaker is running): `pcs node standby <node>`
2. **Downgrade packages**: `sudo apt install <package>=<version>`
3. **Reboot** with the previous kernel (hold GRUB menu: Shift or Esc at boot → Advanced options)
4. **Unstandby**: `pcs node unstandby <node>`

To prevent future upgrades from touching a specific package:

```bash
sudo apt-mark hold zfs-dkms zfsutils-linux zfs-zed linux-image-amd64
sudo apt-mark showhold  # list held packages
```

To cancel an in-progress upgrade and restore the previous node state:

```bash
# If node is in standby and you want to abort the upgrade:
pcs node unstandby <node>  # manually removes standby without running post-upgrade play

# If resources ended up split across both nodes (split-brain), stop all resources and re-evaluate:
pcs cluster stop --all
# Investigate, then restart carefully
```
