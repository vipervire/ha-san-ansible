# Rolling Cluster Upgrade Procedure

This document describes how to perform a rolling upgrade of the HA SAN cluster (kernel, ZFS DKMS, Pacemaker/Corosync, or OS packages) with zero downtime for clients.

## Pre-Upgrade Checklist

Before starting, verify the cluster is in a healthy state:

```bash
# Verify cluster health and resource locations
ssh storage-a 'pcs status'

# Expected: all resources running, no FAILED resources, quorum OK
# san-resources group should show on one node (e.g., storage-a)

# Verify ZFS pool health
ssh storage-a 'zpool status san-pool'

# Expected: ONLINE, no errors, no resilver in progress

# Verify STONITH is functional
ssh storage-a 'pcs stonith status'

# Confirm iSCSI sessions are established
ssh storage-a 'iscsiadm -m session'
ssh storage-b 'iscsiadm -m session'

# Check no pending reboots on either node
ssh storage-a 'ls /var/run/reboot-required 2>/dev/null && echo "REBOOT PENDING" || echo "clean"'
ssh storage-b 'ls /var/run/reboot-required 2>/dev/null && echo "REBOOT PENDING" || echo "clean"'
```

**Do not proceed if:**
- Any resource is FAILED or in a transitional state
- A ZFS scrub or resilver is in progress
- Quorum is not established
- STONITH is non-functional

## Phase 1: Upgrade the Standby Node (storage-b)

### Step 1: Put storage-b in standby mode

```bash
ssh storage-a 'pcs node standby storage-b'
```

Wait for resources to fully migrate to storage-a:
```bash
ssh storage-a 'watch pcs status'
# Wait until all san-resources are running on storage-a and storage-b shows "standby"
```

### Step 2: Upgrade storage-b

```bash
ssh storage-b
sudo apt update
sudo apt upgrade -y
# If upgrading ZFS DKMS specifically:
sudo apt install --only-upgrade zfs-dkms zfsutils-linux zfs-zed

# Check for config file prompts — review and apply as appropriate
```

### Step 3: Reboot storage-b

```bash
ssh storage-b 'sudo reboot'
```

### Step 4: Verify storage-b rejoined the cluster

```bash
# Wait 2-3 minutes for boot, then check:
ssh storage-a 'pcs status'
# Expected: storage-b is "online" (not "standby" or "offline")

# Verify iSCSI sessions re-established
ssh storage-b 'iscsiadm -m session'

# If sessions didn't auto-reconnect:
ssh storage-b 'sudo iscsiadm -m node --loginall=automatic'
```

### Step 5: Resume storage-b in cluster

```bash
ssh storage-a 'pcs node unstandby storage-b'
# Resources remain on storage-a (resource stickiness keeps them there)
```

### Step 6: Wait for ZFS resilver to complete (if triggered)

```bash
ssh storage-a 'watch zpool status san-pool'
# If resilvering: wait for completion before proceeding to Phase 2
```

## Phase 2: Upgrade the Active Node (storage-a)

### Step 7: Migrate resources to storage-b

```bash
ssh storage-a 'pcs resource move san-resources storage-b'
# Wait for resources to fully migrate:
ssh storage-a 'watch pcs status'
# Expected: san-resources running on storage-b
```

Verify services are working from storage-b:
```bash
# Test NFS VIP from a client:
ping 10.30.30.10
showmount -e 10.30.30.10

# Test Cockpit:
curl -k https://10.20.20.10:9090
```

### Step 8: Upgrade storage-a

```bash
ssh storage-a
sudo apt update
sudo apt upgrade -y
```

### Step 9: Reboot storage-a

```bash
ssh storage-a 'sudo reboot'
```

### Step 10: Verify storage-a rejoined

```bash
ssh storage-b 'pcs status'
# Expected: storage-a is "online"
ssh storage-a 'iscsiadm -m session'
```

## Phase 3: Restore Normal Cluster State

### Step 11: Clear resource location constraint

```bash
# Remove the forced location constraint from Step 7
ssh storage-a 'pcs resource clear san-resources'
```

Resources will naturally migrate back to storage-a after the sticky timeout, or immediately if resource stickiness is low. To force it back now:

```bash
ssh storage-a 'pcs resource move san-resources storage-a'
ssh storage-a 'pcs resource clear san-resources'  # remove constraint after migration
```

### Step 12: Final verification

```bash
ssh storage-a 'pcs status'
# All resources running, both nodes online, no constraints

ssh storage-a 'zpool status san-pool'
# ONLINE, no errors

# Check upgrade versions
ssh storage-a 'uname -r && zfs --version'
ssh storage-b 'uname -r && zfs --version'
```

## Upgrade via Ansible

To re-apply the Ansible playbook after upgrade:

```bash
# Re-apply hardening and common base only (safe on live cluster)
ansible-playbook -i inventory.yml site.yml --tags base

# Check only (no changes):
ansible-playbook -i inventory.yml site.yml --check --diff

# Full re-apply (careful on live cluster — review what will change first)
ansible-playbook -i inventory.yml site.yml --check --diff
ansible-playbook -i inventory.yml site.yml
```

## Quorum Node (quorum) Upgrade

The quorum node has no storage role and can be upgraded independently:

```bash
ssh quorum
sudo apt update && sudo apt upgrade -y
sudo reboot
# Cluster retains quorum (2/3 nodes remain) during quorum node reboot
```

## Rollback

If an upgrade causes issues:

1. Put the problematic node in standby: `pcs node standby <node>`
2. Downgrade the package(s): `sudo apt install <package>=<version>`
3. Reboot if kernel was downgraded (hold the GRUB menu)
4. Unstandby the node: `pcs node unstandby <node>`

To hold a specific package version to prevent future upgrades:
```bash
sudo apt-mark hold zfs-dkms zfsutils-linux
```
