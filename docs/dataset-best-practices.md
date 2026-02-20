# ZFS Dataset Best Practices

Configuration recommendations for ZFS datasets by workload type.
Set these properties at dataset creation or with `zfs set <property>=<value> <dataset>`.

## Recommended Settings by Workload

| Workload | recordsize | compression | atime | sync | primarycache |
|----------|-----------|-------------|-------|------|--------------|
| NFS general | 128K | lz4 | off | on | all |
| NFS database files | 16K | lz4 | off | on | all |
| SMB file server | 128K | lz4 | off | on | all |
| iSCSI (VMs/general) | 16K | off | off | on | metadata |
| iSCSI (databases) | 8K | off | off | on | metadata |
| k3s persistent volumes | 128K | lz4 | off | on | all |

### Key Properties Explained

**`recordsize`** — maximum block size for files. Match to the typical I/O size of the workload:
- `128K` is appropriate for sequential file workloads (NFS, SMB, backups)
- `16K` or `8K` aligns with database page sizes (PostgreSQL default: 8K, MySQL InnoDB: 16K)
- For iSCSI/zvol, match the guest filesystem's cluster size or the hypervisor's I/O size

**`compression=lz4`** — fast compression with minimal CPU cost. Recommended for all text/data
workloads. Disable for iSCSI datasets that pass through compressed or encrypted data (VMs, DBs)
since the guest OS handles compression — double-compression wastes CPU and reduces performance.

**`atime=off`** — disables access time updates on every read. Dramatically reduces metadata write
amplification for read-heavy workloads. There is rarely a reason to leave atime on for SAN use.

**`sync=on`** — ensures writes are committed to the SLOG (or disk) before returning to the
application. Required for data safety. Do not set `sync=disabled` on production datasets unless
you have a UPS and understand the implications.

**`primarycache=metadata`** — for iSCSI datasets, the ARC should only cache metadata, not block
data. The guest OS (and its own page cache) handles data caching. Using `primarycache=all` on
iSCSI datasets causes double-caching and wastes ARC space.

## Additional Recommendations

### For iSCSI Datasets (Zvols)

```bash
# Create a zvol for VM storage
zfs create -V 500G \
  -o volblocksize=16K \
  -o compression=off \
  -o primarycache=metadata \
  -o sync=on \
  san-pool/iscsi/vm-storage

# Create a zvol for a database workload
zfs create -V 200G \
  -o volblocksize=8K \
  -o compression=off \
  -o primarycache=metadata \
  -o sync=on \
  san-pool/iscsi/db-storage
```

Note: `volblocksize` cannot be changed after creation. Set it correctly at creation time.

### For Critical Metadata Datasets

For datasets containing critical, hard-to-recreate data (e.g., Samba private directory,
cluster configuration):

```bash
# Extra redundancy for critical small datasets
zfs set copies=2 san-pool/cluster-config
```

`copies=2` stores two copies of each block on different vdevs. This protects against a single
silent corruption event that a scrub would not catch until too late.

### For SMB Shadow Copies

Enable `snapdir=visible` so that Windows clients can access the `.zfs/snapshot` directory
directly via the "Previous Versions" tab:

```bash
zfs set snapdir=visible san-pool/smb
```

This is already configured in the `smb.conf.j2` template via the `shadow_copy2` VFS module
(`shadow:snapdir = .zfs/snapshot`).

### Snapshot Naming Convention

Sanoid uses the pattern `autosnap_YYYY-MM-DD_HH:MM:SS_<policy>` by default. The SMB shadow copy
configuration uses:
```
shadow:format = autosnap_%Y-%m-%d_%H:%M:%S_hourly
```
If you change the Sanoid snapshot naming template, update `smb.conf.j2` to match.

## Example Dataset Hierarchy

```
san-pool/
├── nfs/                    # NFS exports root (recordsize=128K, lz4)
│   ├── data/               # General file shares
│   └── backups/            # Backup storage
├── smb/                    # SMB shares root (recordsize=128K, lz4, snapdir=visible)
│   └── shared/             # Shared SMB folder
├── iscsi/                  # iSCSI zvols (recordsize=16K, compression=off, primarycache=metadata)
│   ├── proxmox-vms         # VM disk images
│   └── proxmox-ct          # Container storage
└── cluster-config/         # Pacemaker-managed config symlink target (copies=2)
    ├── nfs/                # /etc/exports symlink target
    └── samba/              # /etc/samba/smb.conf symlink target
```

## References

- [OpenZFS recordsize Guidance](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html)
- [OpenZFS Dataset Properties](https://openzfs.github.io/openzfs-docs/man/master/8/zfsprops.8.html)
- [Samba Shadow Copy VFS](https://www.samba.org/samba/docs/current/man-html/vfs_shadow_copy2.8.html)
