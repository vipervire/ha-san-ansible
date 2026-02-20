# Cockpit HA Configuration Guide

## Overview

Cockpit is configured for high availability with:
1. **Shared storage configs** - NFS/SMB configurations stored on ZFS pool
2. **Cockpit VIP** - Virtual IP (10.20.20.10) follows active node
3. **Automatic failover** - Changes made via web UI transfer during failover

## Architecture

```
User → https://10.20.20.10:9090 (VIP)
         ↓
    Active Node (storage-a or storage-b)
         ↓
    Cockpit Web UI
         ↓
    Configs on /san-pool/cluster-config/
         ↓ (symlinked to)
    /etc/exports, /etc/samba/smb.conf, /var/lib/samba/private/
```

## Configuration Locations

### Shared Storage (Transfers on Failover)

- **NFS exports:** `/san-pool/cluster-config/nfs/exports`
  - Symlinked from: `/etc/exports`
  - Edit via Cockpit File Sharing plugin or manually

- **SMB config:** `/san-pool/cluster-config/samba/smb.conf`
  - Symlinked from: `/etc/samba/smb.conf`
  - Edit via Cockpit File Sharing plugin or manually

- **Samba users:** `/san-pool/cluster-config/samba/private/`
  - Symlinked from: `/var/lib/samba/private/`
  - Add users via Cockpit or: `smbpasswd -a username`

### Local to Each Node (Doesn't Transfer)

- **Cockpit settings:** `/etc/cockpit/` (appearance, certificates, etc.)
- **System configs:** OS-level settings not related to storage

## Usage

### Adding NFS Export via Cockpit

1. Connect to `https://10.20.20.10:9090`
2. Navigate to "File Sharing" (Houston plugin)
3. Create new NFS export
4. Export is written to `/san-pool/cluster-config/nfs/exports`
5. Available on both nodes after failover ✅

### Adding SMB Share via Cockpit

1. Connect to `https://10.20.20.10:9090`
2. Navigate to "File Sharing"
3. Create new SMB share
4. Share is written to `/san-pool/cluster-config/samba/smb.conf`
5. Available on both nodes after failover ✅

### Adding Samba User

1. SSH to active node or use Cockpit Terminal
2. Add user: `sudo smbpasswd -a username`
3. User added to `/san-pool/cluster-config/samba/private/passdb.tdb`
4. Available on both nodes after failover ✅

## Failover Behavior

**Before failover:**
- storage-a is active
- Cockpit VIP: 10.20.20.10 → storage-a (10.20.20.1)
- Pool imported on storage-a
- Configs accessible at `/san-pool/cluster-config/`

**During failover:**
1. Pacemaker stops resources on storage-a
2. Exports ZFS pool
3. Imports ZFS pool on storage-b
4. Starts services on storage-b
5. Moves VIP to storage-b

**After failover:**
- storage-b is active
- Cockpit VIP: 10.20.20.10 → storage-b (10.20.20.2)
- Pool imported on storage-b
- Same configs accessible at `/san-pool/cluster-config/`
- User reconnects to same URL, sees same configs ✅

## Limitations

### Concurrent Access Protection

- **Risk:** Both nodes editing configs simultaneously during split-brain
- **Mitigation:**
  - Pacemaker ensures only one node imports pool at a time
  - Symlinks fail gracefully if pool not imported
  - STONITH fencing prevents split-brain

### Pool Import Requirement

- Configs only accessible when pool is imported
- If pool fails to import, services won't start (expected behavior)
- Pacemaker resource ordering ensures correct startup sequence

## Troubleshooting

### Cockpit VIP not responding

```bash
# Check VIP location
pcs status | grep vip-cockpit

# Check if VIP is bound
ip addr show | grep 10.20.20.10

# Test from another machine
ping 10.20.20.10
curl -k https://10.20.20.10:9090
```

### Configs not syncing after failover

```bash
# Check if symlinks exist
ls -la /etc/exports
ls -la /etc/samba/smb.conf
ls -la /var/lib/samba/private

# Should show: /etc/exports -> /san-pool/cluster-config/nfs/exports

# Check if shared configs exist
ls -la /san-pool/cluster-config/nfs/
ls -la /san-pool/cluster-config/samba/

# If missing, re-run Ansible playbook
ansible-playbook -i inventory.yml site.yml --tags services
```

### Manual config edit doesn't take effect

```bash
# If you edited local file instead of shared storage:
# Edit the correct location:
sudo vim /san-pool/cluster-config/nfs/exports

# Reload service
sudo exportfs -ra  # NFS
sudo systemctl reload smbd  # SMB
```

## Best Practices

1. **Always use VIP:** Connect to `https://10.20.20.10:9090`, not node IPs
2. **Verify configs:** Check `/san-pool/cluster-config/` not `/etc/` paths
3. **Test failover:** After making changes, test that they survive failover
4. **Backup configs:** Include `/san-pool/cluster-config/` in backups
5. **Monitor symlinks:** Ensure symlinks don't break during updates

## Verification Steps

### Test Cockpit VIP Access

```bash
# From your workstation
ping 10.20.20.10  # Should respond from active node

# Access Cockpit via VIP
curl -k https://10.20.20.10:9090
# Should return Cockpit login page

# Open in browser
https://10.20.20.10:9090
```

### Test Config Sync

```bash
# Connect to Cockpit via VIP
# Navigate to File Sharing → NFS
# Add a new export: /san-pool/test-share

# Verify export was written to shared storage
ssh storage-a "cat /san-pool/cluster-config/nfs/exports | grep test-share"

# Perform planned failover — put active node in standby (resources migrate automatically)
ssh storage-a "pcs node standby storage-a"

# Wait for failover to complete (~5-8s)
pcs status

# Verify VIP moved
ping 10.20.20.10  # Now responds from storage-b

# Check export still exists
ssh storage-b "cat /san-pool/cluster-config/nfs/exports | grep test-share"

# Access Cockpit again via same VIP
https://10.20.20.10:9090
# Should show same test-share

# Return storage-a to service when ready
ssh storage-a "pcs node unstandby storage-a"
```

### Test Samba User Sync

```bash
# Connect to Cockpit via VIP
# Open Terminal

# Add Samba user
sudo smbpasswd -a testuser
# Enter password when prompted

# Verify user in shared storage
sudo pdbedit -L -d 0
sudo ls -la /san-pool/cluster-config/samba/private/

# Perform failover
pcs node standby storage-a

# After failover, verify user still exists
ssh storage-b "sudo pdbedit -L"
# Should show testuser
```

## Rollback Procedure

If issues arise:

```bash
# Remove Cockpit VIP and service from cluster
pcs resource delete vip-cockpit
pcs resource delete cockpit-service

# Re-enable Cockpit on storage nodes
ssh storage-a "sudo systemctl enable --now cockpit.socket"
ssh storage-b "sudo systemctl enable --now cockpit.socket"

# Restore local configs (remove symlinks)
ssh storage-a "
  sudo rm /etc/exports
  sudo rm /etc/samba/smb.conf
  sudo rm /var/lib/samba/private
"

ssh storage-b "
  sudo rm /etc/exports
  sudo rm /etc/samba/smb.conf
  sudo rm /var/lib/samba/private
"

# Re-deploy configs locally
ansible-playbook -i inventory.yml site.yml --tags services
```

## References

- Main README: `README.md`
- Cockpit documentation: https://cockpit-project.org/
- 45Drives Houston: https://github.com/45Drives/cockpit-file-sharing
- Pacemaker documentation: https://clusterlabs.org/pacemaker/doc/
