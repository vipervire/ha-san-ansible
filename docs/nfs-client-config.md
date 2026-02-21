# NFS Client Mount Options

Recommended NFS client mount options for the HA SAN, organized by network speed and workload type.

## VIP Address

The NFS VIP (`vip_nfs: 10.30.30.10`) follows the active node via Pacemaker. Always mount using the VIP, not the node IP.

## NFS Version

The server is configured for NFSv4.1 only (`nfs_v41_enabled: true`, `nfs_v3_enabled: false`). Mount with `nfsvers=4.1` or `nfsvers=4`.

NFSv4.1 provides:
- pNFS (parallel NFS) for improved multi-client performance
- Sessions with connection multiplexing
- Delegations for client-side caching with server callbacks
- No portmapper dependency (simpler firewall rules)

## Mount Options by Network Speed

### 1GbE (standard home/lab)

```bash
# /etc/fstab
10.30.30.10:/san-pool/nfs  /mnt/san  nfs  \
  nfsvers=4.1,rsize=131072,wsize=131072,hard,timeo=600,retrans=2,\
  _netdev,noatime,nordirplus  0 0
```

Key parameters:
- `rsize=131072,wsize=131072` — 128KB I/O blocks (matches ZFS recordsize=128k)
- `hard` — keep retrying on timeout (prevents silent data loss on failover)
- `timeo=600` — 60 second timeout before retry (1GbE is more latency-sensitive)
- `retrans=2` — 2 retries before returning error to application
- `noatime` — skip access time updates (reduces write amplification)
- `nordirplus` — disable READDIRPLUS (reduces server load for directory-heavy workloads)

### 10GbE (moderate performance)

```bash
10.30.30.10:/san-pool/nfs  /mnt/san  nfs  \
  nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=150,retrans=3,\
  _netdev,noatime,proto=tcp  0 0
```

Key changes from 1GbE:
- `rsize=1048576,wsize=1048576` — 1MB I/O blocks (saturates 10GbE better)
- `timeo=150` — 15 second timeout (lower latency link)
- `retrans=3` — more retries for intermittent issues
- `proto=tcp` — explicit TCP (NFSv4 always uses TCP, but explicit is clear)

### 25/40GbE (high performance)

```bash
10.30.30.10:/san-pool/nfs  /mnt/san  nfs  \
  nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=100,retrans=3,\
  _netdev,noatime,proto=tcp,nconnect=4  0 0
```

Key additions:
- `nconnect=4` — use 4 TCP connections per mount (Linux 5.3+, maximizes bandwidth)
- `timeo=100` — 10 second timeout (fast link, quick detection)

## Workload-Specific Recommendations

### VM/Container Storage (Proxmox, etc.)

```bash
10.30.30.10:/san-pool/nfs  /mnt/san  nfs  \
  nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=100,retrans=3,\
  _netdev,noatime,proto=tcp,async  0 0
```

- `async` — write-behind caching (improves VM performance; Proxmox handles fsync explicitly)
- For production VMs with databases: use `sync` instead of `async`

### Database / Synchronous Workloads

```bash
10.30.30.10:/san-pool/nfs  /mnt/san  nfs  \
  nfsvers=4.1,rsize=131072,wsize=131072,hard,timeo=100,retrans=3,\
  _netdev,noatime,proto=tcp,sync  0 0
```

- `sync` — flush writes to server before returning (data safety for databases)
- Smaller block sizes match database page sizes (128KB for PostgreSQL, InnoDB)
- Consider using iSCSI instead of NFS for database workloads (lower overhead)

### General File Sharing (Samba/CIFS is preferred)

For Windows clients and mixed environments, use the SMB VIP (`10.30.30.11`) instead of NFS.

## Failover Behavior

During failover (node failure or `pcs resource move`):
1. NFS VIP migrates to the surviving node (~5-10 seconds)
2. NFS service restarts on the surviving node
3. NFS grace period allows clients to reclaim locks (`nfs_grace_time: 45` seconds)
4. Clients with `hard` mount option will pause I/O and automatically reconnect
5. Clients with `soft` mount option may get I/O errors during failover — **avoid `soft`**

### Estimated client reconnect time

| Phase | Duration |
|-------|----------|
| Node failure detection (Corosync) | ~4s |
| STONITH fence action | 3-15s (depends on method) |
| ZFS pool import on surviving node | 5-15s |
| NFS service start | 2-5s |
| VIP assignment | 1-2s |
| **Total** | **~15-45s** |

With `timeo=600` (60 seconds), 1GbE clients will typically not experience visible errors for short failovers.

## Automount (systemd)

For systemd automount (lazy mount on access):

```ini
# /etc/systemd/system/mnt-san.automount
[Unit]
Description=NFS automount for SAN

[Automount]
Where=/mnt/san
TimeoutIdleSec=600

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/mnt-san.mount
[Unit]
Description=NFS mount for SAN
After=network-online.target
Wants=network-online.target

[Mount]
What=10.30.30.10:/san-pool/nfs
Where=/mnt/san
Type=nfs
Options=nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=150,retrans=3,noatime

[Install]
WantedBy=multi-user.target
```

## Troubleshooting

```bash
# Check mount options currently in use
mount | grep nfs
cat /proc/mounts | grep nfs

# Check NFS statistics
nfsstat -c    # client stats
nfsstat -s    # server stats (on storage node)

# Check NFS server exports
showmount -e 10.30.30.10

# Debug connection issues
tcpdump -n -i any host 10.30.30.10 and port 2049

# Check for stale file handle errors
dmesg | grep "NFS\|nfs"

# Force cache flush
echo 3 > /proc/sys/vm/drop_caches   # on client
```
