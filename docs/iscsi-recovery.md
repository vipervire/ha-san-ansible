# iSCSI Session Recovery Procedure

This document covers recovery procedures for iSCSI session failures in the HA SAN backend (between storage nodes).

## Background

The HA SAN uses iSCSI for backend replication: each storage node exports its local disks via LIO iSCSI target to the peer, and connects to the peer's target as an initiator. ZFS mirrors local disks with remote iSCSI disks.

If iSCSI sessions drop:
- ZFS will mark the remote vdevs as `FAULTED` (aggressive-safe timeout: ~5s)
- The pool continues running in degraded mode on local disks only
- Pacemaker continues all services normally
- When sessions reconnect, ZFS auto-onlines the vdevs and begins resilvering

## Step 1: Verify Session State

```bash
# Check active iSCSI sessions
iscsiadm -m session

# Expected output when healthy:
# tcp: [1] 10.10.10.2:3260,1 iqn.2025-01.lab.home:storage-b (non-flash)

# If no output: sessions are down
```

## Step 2: Check ZFS Pool Status

```bash
zpool status san-pool
# Look for FAULTED or DEGRADED vdevs
# FAULTED remote vdevs = iSCSI sessions are down
```

## Step 3: Diagnose the Root Cause

### Check storage VLAN connectivity

```bash
# Test storage VLAN reachability
ping -c 4 10.10.10.2    # from storage-a to storage-b storage IP
ping -c 4 10.10.10.1    # from storage-b to storage-a storage IP

# Check interface status
ip link show
ip addr show

# Verify VLAN tagging
ip -d link show | grep vlan
```

### Check MTU consistency

```bash
# Both ends must agree on MTU (9000 for jumbo frames)
ip link show | grep mtu
ping -c 1 -s 8972 -M do 10.10.10.2   # Test jumbo frame path (8972 + 28 = 9000)
```

### Check iSCSI target service on peer

```bash
# On storage-b (the target being connected to from storage-a)
systemctl status rtslib-fb-targetctl
targetcli ls

# Check LIO target is listening
ss -tlnp | grep 3260
```

## Step 4: Reconnect Sessions

### Option A: Rescan existing sessions

```bash
iscsiadm -m node --rescan
```

### Option B: Force logout and re-login

```bash
# Get the session details
iscsiadm -m session -P 1

# Logout from all sessions
iscsiadm -m node --logout

# Log back in automatically
iscsiadm -m node --loginall=automatic

# Verify sessions restored
iscsiadm -m session
```

### Option C: Targeted logout/login for specific target

```bash
TARGET_IQN="iqn.2025-01.lab.home:storage-b"
TARGET_IP="10.10.10.2"

iscsiadm -m node -T "${TARGET_IQN}" -p "${TARGET_IP}:3260" --logout
iscsiadm -m node -T "${TARGET_IQN}" -p "${TARGET_IP}:3260" --login
```

### Option D: Full iSCSI restart (last resort)

```bash
# WARNING: This will briefly interrupt ALL iSCSI sessions
# Only do this if the pool is already degraded (remote vdevs FAULTED)
systemctl restart open-iscsi
iscsiadm -m node --loginall=automatic
```

## Step 5: Verify ZFS Recovery

After sessions reconnect, ZFS auto-onlines the faulted vdevs (autoreplace=on):

```bash
# Check pool status — should show ONLINE or DEGRADED (resilvering)
zpool status san-pool

# If vdevs remain FAULTED after sessions reconnect, online them manually:
zpool online san-pool <device-path>

# Monitor resilver progress
watch zpool status san-pool
```

## Step 6: Monitor Resilver

Resilver time depends on the amount of data written during the outage:

```bash
# Live progress
zpool status san-pool

# Estimated time remaining is shown in the resilver progress line
# Example: "resilvered 127G in 00:12:14 with 0 errors"
```

## Preventing Session Failures

The playbook configures aggressive-safe timeouts:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `noop_out_interval` | 2s | Heartbeat frequency |
| `noop_out_timeout` | 2s | Heartbeat timeout |
| `replacement_timeout` | 5s | Time before LUN faulted |
| `login_timeout` | 10s | Initial login timeout |

These are tuned to detect peer failure within ~4-5s, which matches the Corosync token timeout (4000ms).

## iSCSI Session Recovery After Full Node Failure

When a storage node reboots and rejoins:

1. open-iscsi service starts and auto-logs in (configured by playbook)
2. Sessions re-establish within seconds
3. ZFS auto-onlines the previously faulted vdevs
4. ZED fires `vdev_online` and `resilver_start` events (ntfy notification if configured)
5. Resilver begins to restore full redundancy

**No manual intervention required** for normal node restarts.

## Checking for iSCSI Authentication Issues

If sessions fail to re-establish due to CHAP mismatch:

```bash
# Check initiator auth configuration
grep -i "auth" /etc/iscsi/iscsid.conf

# Check target auth configuration
targetcli ls /iscsi/<target-iqn>/tpg1

# Test with no auth (temporarily)
# Note: This is a diagnostic step only — re-enable auth after diagnosing
targetcli /iscsi/<target-iqn>/tpg1 set attribute authentication=0
iscsiadm -m node -T <target-iqn> -p 10.10.10.2:3260 --login

# If this connects, the issue is CHAP credential mismatch
# Fix: Ensure iscsid.conf credentials match LIO target ACL credentials
```
