# Development Guidelines for HA SAN Ansible Playbook

This file contains guidelines and reminders for maintaining and extending this Ansible playbook.

## What This Playbook Does

This is an **Ansible playbook for deploying a high-availability ZFS-over-iSCSI SAN** on Debian 12 or Rocky Linux 9. The architecture is:

- **storage-a** and **storage-b**: Two symmetric storage nodes. Each exports its local disks via LIO iSCSI target to the peer, and connects to the peer's iSCSI target. A ZFS pool mirrors local physical disks with remote iSCSI disks. Either node can serve as active primary.
- **quorum**: A lightweight third node that participates in Corosync/Pacemaker quorum voting only (no storage role).
- **Active/passive HA**: Pacemaker manages a resource group (ZFS pool, VIPs, NFS, SMB, iSCSI client target, Cockpit) that runs on one node at a time. On failover, the surviving node imports the pool in degraded state and resumes service. The pool re-silvers to full redundancy when the peer reconnects.

## Project Structure

```
.
├── inventory.yml           # Hosts: storage-a (10.20.20.1), storage-b (10.20.20.2), quorum (10.20.20.3)
├── site.yml               # Main playbook — 6 plays with tags
├── os-upgrade.yml         # Rolling OS upgrade helper — pre/post-upgrade plays, always use --limit
├── group_vars/
│   ├── all.yml                      # Global vars: cluster name, VLANs, VIPs, Corosync tuning, SSH, NTP
│   └── storage_nodes/               # Storage vars split by concern (Ansible merges all files in directory)
│       ├── network.yml              # Interface names, TCP tuning
│       ├── zfs.yml                  # ZFS pool, datasets, scrub, Sanoid, ZED, Syncoid
│       ├── iscsi.yml                # iSCSI backend CHAP + client-facing target
│       ├── services.yml             # NFS exports, SMB shares
│       └── cluster.yml              # STONITH config, Pacemaker tuning, monitoring flags
├── host_vars/
│   ├── storage-a.yml      # Node-specific: IPs, iSCSI IQNs, local disk paths
│   ├── storage-b.yml      # Node-specific: IPs, iSCSI IQNs, local disk paths
│   └── quorum.yml         # Minimal: mgmt_ip and ssh_listen_addresses only
├── roles/
│   ├── common/            # Base OS: packages, NTP (chrony), package cleanup
│   │   ├── tasks/main.yml      # Shared tasks — dispatches to OS-specific files
│   │   ├── tasks/Debian.yml    # Debian: apt packages, unattended-upgrades
│   │   ├── tasks/RedHat.yml    # Rocky: dnf packages, dnf-automatic
│   │   ├── vars/Debian.yml     # Debian package names, paths
│   │   └── vars/RedHat.yml     # Rocky package names, paths
│   ├── hardening/         # Security: nftables firewall, SSH hardening, sysctl, PAM faillock
│   ├── zfs/               # ZFS install, ARC tuning, modprobe options, Sanoid snapshots
│   ├── iscsi-target/      # LIO iSCSI target setup (backend disk replication)
│   ├── iscsi-initiator/   # open-iscsi initiator (connects to peer's target); generates create-pool.sh
│   ├── pacemaker/         # Corosync + Pacemaker: cluster auth, corosync.conf, STONITH scripts
│   ├── services/          # NFS, SMB, iSCSI client target; shared config dir on ZFS pool
│   ├── monitoring/        # node_exporter, ha_cluster_exporter, ZFS scrub exporter, STONITH probe, reboot exporter
│   └── cockpit/           # Cockpit + 45Drives Houston plugins
│   # Each role follows the same pattern: tasks/{main,Debian,RedHat}.yml + vars/{Debian,RedHat}.yml
└── docs/
    ├── cluster-monitoring.md      # ha_cluster_exporter, Prometheus scrape configs, key metrics
    ├── cockpit-ha-config.md       # Cockpit VIP + shared storage config sync (symlinks)
    ├── dataset-best-practices.md  # ZFS dataset properties by workload type
    ├── ha-san-design.html         # Architecture overview (HTML)
    ├── ha-san-ops.html            # Operations runbook (HTML)
    ├── nfs-security.md            # Why sec=sys is used; Kerberos alternative explanation
    ├── ntfy-integration.md        # Prometheus Alertmanager → NTFY push notifications
    ├── prometheus-alerts.yml      # Alert rule examples
    ├── prometheus-recording-rules.yml  # Recording rules for aggregation
    ├── stonith-migration.md       # Migrating from old global stonith_method to per-node dict
    ├── stonith-smart-plugs.md     # Smart plug fencing guide (Kasa, Tasmota, ESPHome, HTTP)
    ├── os-upgrade.md              # Rolling OS upgrade guide — dist-upgrade and full reinstall
    └── upgrade-procedure.md       # Manual upgrade reference (pre-dates os-upgrade.yml)
```

## Playbook Tags

Run targeted subsets of the playbook with `--tags`:

```bash
# Full deployment (all roles, all nodes):
ansible-playbook -i inventory.yml site.yml

# Base OS + hardening only (all nodes):
ansible-playbook -i inventory.yml site.yml --tags base

# ZFS + iSCSI on storage nodes only:
ansible-playbook -i inventory.yml site.yml --tags storage

# Pacemaker cluster setup (all nodes):
ansible-playbook -i inventory.yml site.yml --tags cluster

# NFS/SMB/iSCSI client services (storage nodes):
ansible-playbook -i inventory.yml site.yml --tags services

# Cockpit + 45Drives Houston (storage nodes):
ansible-playbook -i inventory.yml site.yml --tags cockpit

# Monitoring exporters (all nodes):
ansible-playbook -i inventory.yml site.yml --tags monitoring

# Dry run with diff:
ansible-playbook -i inventory.yml site.yml --check --diff

# Rolling OS upgrade — always target one node at a time with --limit:
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit storage-b
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit storage-b
```

## Critical Checks When Making Changes

### 1. Firewall Rules (nftables)

**IMPORTANT:** When adding new services, monitoring endpoints, or network-accessible components, **ALWAYS** check if firewall rules need updating.

**File:** `roles/hardening/templates/nftables.conf.j2`

**Current firewall port assignments:**

| VLAN | Ports | Services |
|------|-------|----------|
| Management (10.20.20.0/24) | 22/tcp | SSH |
| Management | 5405/udp | Corosync (knet) |
| Management | 2224/tcp | pcsd (Pacemaker) |
| Management | 9090/tcp | Cockpit web UI |
| Management | 9100/tcp | node_exporter |
| Management | 9664/tcp | ha_cluster_exporter |
| Storage (10.10.10.0/24) | 3260/tcp | iSCSI (backend replication) |
| Storage | 5405/udp | Corosync ring1 (storage nodes only) |
| Client (10.30.30.0/24) | 2049, 20048/tcp | NFS |
| Client | 111/tcp+udp | NFS portmapper |
| Client | 445/tcp | SMB |
| Client | 3260/tcp | iSCSI (client-facing target) |

**Common scenarios requiring firewall updates:**

- **Adding monitoring exporters** — add rule to management VLAN section
- **Adding web services** — add rule to management VLAN section
- **Adding storage protocols** — add rule to client VLAN section
- **Adding cluster services** — check if port already present before adding

**Example firewall rule:**
```nftables
# Prometheus ha_cluster_exporter (Pacemaker/Corosync metrics)
ip saddr {{ vlans.management.subnet }} tcp dport 9664 accept
```

**Testing after changes:**
```bash
ssh storage-a
sudo nft list ruleset | grep <port-number>
curl http://10.20.20.1:<port>  # From management VLAN
```

### 2. STONITH Configuration

When modifying STONITH/fencing configuration:

- **File:** `roles/pacemaker/templates/configure-stonith.sh.j2`
- **Config:** `group_vars/storage_nodes/cluster.yml` (stonith_nodes dict)
- **Test:** Always test fencing in non-production before deploying
- **Document:** Update `docs/stonith-smart-plugs.md` if adding new fence agent types

**Supported methods:** `ipmi`, `kasa`, `tasmota`, `esphome`, `http`

The playbook supports **mixed per-node fencing methods**. Example:
```yaml
stonith_nodes:
  storage-a:
    method: "ipmi"           # Enterprise server with BMC
    ip: "10.20.20.101"
    user: "bmcadmin"
    password: "CHANGEME-vault-this"
  storage-b:
    method: "kasa"           # Consumer server with smart plug
    ip: "10.20.20.202"
    # No credentials needed for Kasa local protocol
```

`python3-kasa` is installed automatically if any node uses method `kasa`. See `docs/stonith-smart-plugs.md` for all plug types and `docs/stonith-migration.md` for migrating from the old `stonith_method` global variable.

**Post-fence verification** is enabled by default (`stonith_fence_verify: true`). After the primary fence agent runs, a custom `fence_check` agent queries the STONITH device power state and pings the target on all three VLANs. If either check fails, Pacemaker considers fencing incomplete and **blocks failover** to prevent split-brain. Both agents are placed in **fencing level 1** — all must succeed for fencing to complete:

```bash
pcs stonith level    # Should show: Level 1 - storage-b: fence-storage-b,fence-verify-storage-b
fence_check -o status -n storage-b    # Non-destructive manual check
```

To disable: set `stonith_fence_verify: false` in `group_vars/storage_nodes.yml`, re-run the playbook, and re-run `configure-stonith.sh`. See `docs/stonith-smart-plugs.md#post-fence-verification` for troubleshooting a blocked failover.

**Never test STONITH without a maintenance window.** `pcs stonith fence <node>` powers off the node immediately.

**Fencing latency and ZFS pool start timeout:** On unplanned node failure, Pacemaker must fence the dead node before importing the ZFS pool on the survivor (`multihost=on` enforces this). The ZFS resource start timeout (150s in `configure-resources.sh`) must exceed the maximum expected fencing time. Typical latencies:

| Method | Typical latency |
|--------|----------------|
| ipmi   | 20–30s          |
| kasa   | 5–15s           |
| http   | varies          |

If your fence agent is slower than the default covers, increase `pcmk_reboot_timeout` in `configure-stonith.sh.j2` and the `op start timeout` in `configure-resources.sh.j2` to match.

### 3. Cockpit HA Configuration

When modifying Cockpit, NFS, or SMB configurations:

- **Important:** Config files are symlinked to shared storage for automatic sync during failover
- **Symlinked paths:**
  - `/etc/exports` → `/san-pool/cluster-config/nfs/exports`
  - `/etc/samba/smb.conf` → `/san-pool/cluster-config/samba/smb.conf`
  - `/var/lib/samba/private/` → `/san-pool/cluster-config/samba/private/`
- **Cockpit VIP:** 10.20.20.10 (management VLAN) follows active node
- **Managed by Pacemaker:** Cockpit service is part of san-resources group
- **Documentation:** `docs/cockpit-ha-config.md` for detailed configuration and troubleshooting

**When editing NFS/SMB configs manually:**
- Always edit the shared storage location, not the symlink target
- Example: `vim /san-pool/cluster-config/nfs/exports` (correct)
- NOT: `vim /etc/exports` (works but less obvious it's on shared storage)

### 4. Monitoring Changes

The monitoring role deploys these exporters on **all cluster nodes** (`hosts: cluster`):

| Exporter | Port | Purpose | Template/Source |
|----------|------|---------|----------------|
| `prometheus-node-exporter` | 9100 | System metrics | apt package |
| `prometheus-hacluster-exporter` | 9664 | Pacemaker/Corosync metrics | `ha-cluster-exporter-default.j2` |
| `zfs-scrub-exporter` (timer) | 9100 (textfile) | ZFS scrub state/timing | `zfs-scrub-exporter.{sh,service,timer}.j2` |
| `reboot-required-exporter` (timer) | 9100 (textfile) | Pending reboot flag | `reboot-required-exporter.{sh,service,timer}.j2` |
| `stonith-probe` (timer, storage nodes only) | 9100 (textfile) | STONITH agent reachability | `stonith-probe.{sh,service,timer}.j2` |

Textfile exporters write `.prom` files to `/var/lib/prometheus/node-exporter/` for node_exporter to collect.

**When adding new metrics or exporters:**
1. Add script + systemd service/timer templates to `roles/monitoring/templates/`
2. Add deploy tasks to `roles/monitoring/tasks/main.yml`
3. **Add firewall rule** if using its own port (see section 1)
4. Add alert rules to `docs/prometheus-alerts.yml`
5. Document in `docs/cluster-monitoring.md` or `docs/ntfy-integration.md`
6. Update README "Monitoring and Alerting" section

**Controlling per-exporter behaviour:**
```yaml
# group_vars/all.yml
ha_cluster_monitoring_enabled: true   # toggles ha_cluster_exporter
# group_vars/storage_nodes/cluster.yml
zfs_scrub_monitoring_enabled: true    # toggles ZFS scrub exporter
smart_monitoring_enabled: true        # toggles S.M.A.R.T disk health exporter
```

### 5. Per-Node vs. Global Configuration

**Per-node settings** (`host_vars/<hostname>.yml`):
- Disk layouts (`local_data_disks`) — must use stable `/dev/disk/by-id/` paths, never `/dev/sdX`
- IP addresses (`mgmt_ip`, `storage_ip`, `client_ip`)
- iSCSI IQNs (`iscsi_target_iqn`, `iscsi_initiator_name`, `iscsi_peer_ip`, `iscsi_peer_iqn`)
- Optional `zfs_arc_max` override (otherwise computed as 50% of RAM at deploy time)
- Optional `slog_disk` and `special_disk` for NVMe SLOG/special vdevs

**Global settings** (`group_vars/all.yml`):
- VLAN configuration, VIPs (`vip_nfs`, `vip_smb`, `vip_iscsi`, `vip_cockpit`)
- Cluster name and Corosync node list
- Corosync tuning (`corosync_token`, `corosync_consensus`, etc.)
- Monitoring flags (`ha_cluster_monitoring_enabled`, ports)
- SSH policy, NTP servers, admin user

**Storage node settings** (`group_vars/storage_nodes/` — split by concern):
- `network.yml`: Interface names (`net_storage_parent`, etc.), TCP tuning for 40GbE
- `zfs.yml`: Pool name, ashift, pool/dataset options, modprobe tunables, scrub schedule, Sanoid, ZED, Syncoid
- `iscsi.yml`: iSCSI CHAP credentials (must be vault-encrypted), queue depth, client target ACLs
- `services.yml`: NFS exports (`nfs_exports`), SMB shares (`smb_shares`)
- `cluster.yml`: STONITH configuration (`stonith_nodes` dict), Pacemaker tuning, monitoring toggles

### 6. Multi-OS Support (Debian 12 / Rocky Linux 9)

The playbook supports both **Debian 12** and **Rocky Linux 9** on all nodes. Each role uses OS-specific task files and variable files, dispatched via `ansible_os_family`:

**Role structure pattern:**
```
roles/<role>/
├── tasks/
│   ├── main.yml        # Loads OS vars, dispatches to OS-specific tasks, then shared tasks
│   ├── Debian.yml      # Debian-specific: apt packages, repos, service names
│   └── RedHat.yml      # Rocky-specific: dnf packages, repos, service names
├── vars/
│   ├── Debian.yml      # Debian package names, paths, service names
│   └── RedHat.yml      # Rocky package names, paths, service names
```

**Key differences between Debian and Rocky:**

| Area | Debian 12 | Rocky Linux 9 |
|------|-----------|---------------|
| Package manager | `apt` | `dnf` |
| Auto-updates | `unattended-upgrades` | `dnf-automatic` |
| PAM faillock | `pam-auth-update --enable faillock` | `authselect enable-feature with-faillock` |
| ZFS source | `deb.debian.org` contrib | OpenZFS ELRepo RPM |
| ZFS prerequisites | `linux-headers-*`, `dkms`, `dpkg-dev` | `kernel-devel-*`, `dkms` |
| ZFS packages | `zfsutils-linux`, `zfs-dkms`, `zfs-zed` | `zfs`, `zfs-dkms` |
| NFS server | `nfs-kernel-server` | `nfs-utils` |
| NFS service name | `nfs-kernel-server` | `nfs-server` |
| Samba services | `smbd`, `nmbd` | `smb`, `nmb` |
| iSCSI target | `targetcli-fb`, `python3-rtslib-fb` | `targetcli`, `python3-rtslib` |
| iSCSI target svc | `rtslib-fb-targetctl` | `target` |
| iSCSI initiator | `open-iscsi` | `iscsi-initiator-utils` |
| Chrony config | `/etc/chrony/chrony.conf` | `/etc/chrony.conf` |
| Service env files | `/etc/default/<service>` | `/etc/sysconfig/<service>` |
| Reboot check | `/var/run/reboot-required` | `needs-restarting -r` |
| Firewall | nftables (native) | nftables (firewalld disabled) |

**When adding new tasks:**
- Use `ansible.builtin.package` for simple installs that share the same package name
- Use OS-specific task files when package names differ or repos need adding
- Always check both `vars/Debian.yml` and `vars/RedHat.yml` when adding packages
- Templates that reference OS-specific paths should use variables from the vars files

### 7. ZFS Tunables

ZFS module parameters are written to `/etc/modprobe.d/zfs.conf` via template (`roles/zfs/templates/zfs-modprobe.conf.j2`). The `zfs_arc_max` is computed dynamically (50% of total RAM) and injected separately as a fact.

**Current tunables (`group_vars/storage_nodes/zfs.yml`):**
```yaml
zfs_modprobe_options:
  zfs_vdev_scheduler: "none"          # No I/O scheduler (SSDs only)
  zfs_txg_timeout: 30                 # Reduces write latency stalls
  zfs_resilver_min_time_ms: 27000     # Steady drive recovery (OpenZFS recommended)
  zfs_resilver_delay: 0
  zfs_nocacheflush: 1                 # ONLY safe with enterprise SSDs with hardware PLP
```

**WARNING:** `zfs_nocacheflush: 1` bypasses drive write-cache flush. Only enable on enterprise SSDs with power-loss protection (PLP). Never enable on consumer drives or spinning disks.

**To override ARC max per-host:**
```yaml
# host_vars/storage-a.yml
zfs_arc_max: 17179869184  # 16GB — explicit per-host override
```

### 8. Sanoid Snapshot Policy

Sanoid is optionally installed and configured for automated ZFS snapshots:
```yaml
# group_vars/storage_nodes/zfs.yml
sanoid_install: true     # Set false to skip Sanoid entirely

sanoid_templates:
  production:  { hourly: 24, daily: 30, monthly: 6, autosnap: true, autoprune: true }
  vm_storage:  { hourly: 12, daily: 14, monthly: 3, autosnap: true, autoprune: true }
```

Default datasets snapshotted: `san-pool/nfs` (production), `san-pool/smb` (production), `san-pool/iscsi` (vm_storage).

### 9. Idempotency

Always ensure tasks are idempotent:

- Use `creates:` with command/shell tasks when possible
- Use `|| true` or `failed_when: false` for pcs commands that may already be configured
- Check for resource existence before creating
- Use `changed_when: false` for read-only commands

**Example:**
```yaml
- name: Create STONITH resource
  ansible.builtin.command:
    cmd: pcs stonith create fence-storage-a ...
  register: result
  failed_when: result.rc != 0 and 'already exists' not in result.stderr
```

### 10. Secrets Management

**Never commit plain text secrets!**

- Use `ansible-vault` for passwords
- Mark vault-required values with comments: `# ansible-vault encrypt_string`
- Files with secrets: `group_vars/storage_nodes.yml`, `host_vars/*.yml`

The playbook has a **pre-flight validation play** (first play in `site.yml`) that fails immediately if any credential still contains the string `CHANGEME`. Vault these before deploying:
- `hacluster_password` (`group_vars/all.yml`)
- `iscsi_chap_password` (`group_vars/storage_nodes.yml`)
- `iscsi_mutual_chap_password` (`group_vars/storage_nodes.yml`)
- `stonith_nodes.<node>.password` (`group_vars/storage_nodes.yml`)

```bash
ansible-vault encrypt_string 'yourpassword' --name 'hacluster_password'
```

**Example vault entry:**
```yaml
hacluster_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...encrypted...
```

## Common Tasks

### Adding a New Service

1. Create role in `roles/<service-name>/`
2. Add tasks, templates, handlers
3. Include role in `site.yml` (choose the right play and hosts group)
4. **Check firewall rules** (`roles/hardening/templates/nftables.conf.j2`)
5. Add configuration variables to appropriate `group_vars/`
6. Document in `README.md`
7. Test: `ansible-playbook -i inventory.yml site.yml --tags <tag>`

### Adding Monitoring for a Component

1. Create textfile exporter script or install package
2. Deploy via `roles/monitoring/tasks/main.yml`
3. **Add firewall rule for metrics port** (if not using textfile collector via 9100)
4. Add Prometheus scrape config
5. Create alert rules in `docs/prometheus-alerts.yml`
6. Document in `docs/cluster-monitoring.md`
7. Test: `curl http://10.20.20.1:9100/metrics | grep <your_metric>`

### Modifying STONITH Configuration

1. Update `stonith_nodes` dict in `group_vars/storage_nodes.yml`
2. Run playbook: `ansible-playbook -i inventory.yml site.yml --tags cluster`
3. Regenerate config: `ssh storage-a 'sudo bash /root/configure-stonith.sh'`
4. Verify: `pcs stonith status`
5. Test (careful!): `pcs stonith fence storage-b` (powers off node!)

### Adding or Changing ZFS Datasets

1. Update `zfs_datasets` in `group_vars/storage_nodes.yml`
2. Consult `docs/dataset-best-practices.md` for recommended properties by workload
3. Run: `ansible-playbook -i inventory.yml site.yml --tags storage`
4. Note: the `zpool create` step is always manual (verify iSCSI paths first)

### Rolling OS Upgrade

Always target **one node at a time** with `--limit`. Recommended order: quorum → standby storage node → active storage node.

```bash
# 1. Pre-upgrade: health checks, optional failover, put node in standby
ansible-playbook -i inventory.yml os-upgrade.yml --tags pre-upgrade --limit storage-b

# 2. Upgrade the node manually (SSH in and run apt, then reboot)

# 3. Re-apply Ansible configuration to the upgraded node
ansible-playbook -i inventory.yml site.yml --limit storage-b

# 4. Post-upgrade: verify services, reconnect iSCSI, remove standby
ansible-playbook -i inventory.yml os-upgrade.yml --tags post-upgrade --limit storage-b

# 5. Wait for ZFS resilver before upgrading the next node (storage nodes only)
#    ssh storage-a 'watch zpool status san-pool'
```

Bypass failed-resource-actions check (pre-upgrade only): `-e force_upgrade=true`

See `docs/os-upgrade.md` for the full procedure, edge cases, and rollback steps.

## Known Issues / Gotchas

### 1. ZFS Pool Creation Is Manual

The playbook configures everything _up to_ pool creation. The actual `zpool create` requires manual verification of iSCSI device paths (they vary per environment). A helper script `/root/create-pool.sh` is generated on the `pacemaker_preferred_node` after iSCSI initiator role runs.

```bash
ssh storage-a 'ls -la /dev/disk/by-path/ | grep iscsi'  # verify paths first
ssh storage-a 'sudo bash /root/create-pool.sh'
ssh storage-a 'sudo zpool export san-pool'               # required before Pacemaker starts
```

### 2. iSCSI Disk Paths

- `/dev/sdX` paths are **not stable** — they reorder on reboot, drive replacement, or BIOS update
- Always use `/dev/disk/by-id/` paths in `host_vars/`
- Never use `wwn-*` identifiers (hardware-specific, may not match across environments)
- iSCSI device paths (`/dev/disk/by-path/`) may also vary between runs — always verify in `create-pool.sh`

### 3. STONITH Testing Is Destructive

- STONITH tests power off nodes immediately
- Always test during maintenance windows
- `pcs stonith fence <node>` will POWER OFF the node!

### 4. ZFS Pool Must Be Exported Before Pacemaker Starts

Pacemaker refuses to start the ZFS pool resource if the pool is already imported. After manual pool creation, always run `zpool export san-pool` before starting Pacemaker resources.

### 5. Mutual CHAP Passwords Must Differ

`iscsi_mutual_chap_password` must be different from `iscsi_chap_password`. iSCSI rejects identical bidirectional credentials.

### 6. Corosync Requires Synchronized Clocks

All nodes must have synchronized clocks (chrony is deployed by the `common` role). Corosync token timeouts are aggressive-safe (`corosync_token: 4000ms`). Do not lower below 3000ms.

### 7. 45Drives Houston Packages May Fail

`cockpit-file-sharing`, `cockpit-identities`, `cockpit-navigator` are installed with `ignore_errors: true` because the 45Drives repository may not have packages for all architectures or Debian releases.

### 8. Firewall Port Requirements (Summary)

- **Management VLAN (10.20.20.0/24):** SSH: 22, Corosync: 5405/udp, pcsd: 2224, Cockpit: 9090, node_exporter: 9100, ha_cluster_exporter: 9664
- **Storage VLAN (10.10.10.0/24):** iSCSI (backend): 3260, Corosync ring1: 5405/udp
- **Client VLAN (10.30.30.0/24):** NFS: 2049, 20048, 111, SMB: 445, iSCSI (client): 3260

## Full Deployment Workflow

```bash
# 1. Set secrets in group_vars and host_vars (vault-encrypt all passwords)
# 2. Set local disk paths in host_vars/storage-a.yml and storage-b.yml using /dev/disk/by-id/
# 3. Deploy configuration (stops before pool creation)
ansible-playbook -i inventory.yml site.yml

# 4. Verify cluster formation
ssh storage-a 'pcs status'

# 5. Configure STONITH
ssh storage-a 'sudo bash /root/configure-stonith.sh'
ssh storage-a 'pcs stonith status'

# 6. Verify iSCSI paths and create ZFS pool
ssh storage-a 'ls -la /dev/disk/by-path/ | grep iscsi'
ssh storage-a 'sudo bash /root/create-pool.sh'    # verify paths first!
ssh storage-a 'sudo zpool export san-pool'

# 7. Configure Pacemaker resources
ssh storage-a 'sudo bash /root/configure-pacemaker-resources.sh'

# 8. Verify all resources running
ssh storage-a 'pcs status'
```

## Testing Procedures

### Monitoring Test

```bash
# Check exporters are running
ssh storage-a 'systemctl status prometheus-node-exporter'
ssh storage-a 'systemctl status prometheus-hacluster-exporter'
ssh storage-a 'systemctl status zfs-scrub-exporter.timer'
ssh storage-a 'systemctl status stonith-probe.timer'
ssh storage-a 'systemctl status reboot-required-exporter.timer'

# Test metrics endpoints
curl http://10.20.20.1:9100/metrics | grep zfs_scrub
curl http://10.20.20.1:9100/metrics | grep stonith_agent_reachable
curl http://10.20.20.1:9100/metrics | grep node_reboot_required
curl http://10.20.20.1:9664/metrics | grep ha_cluster
```

### Failover Test

```bash
# 1. Check current resource location
pcs status

# 2. Planned failover (graceful migration)
pcs resource move san-resources storage-b
# Wait for resources to come up on storage-b
pcs resource clear san-resources  # remove location constraint

# 3. Verify resources on new node
ssh storage-b 'pcs status'
ssh storage-b 'zpool status san-pool'
```

## References

- [Pacemaker Documentation](https://clusterlabs.org/pacemaker/doc/)
- [Corosync Documentation](https://clusterlabs.org/corosync/doc/)
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)
- [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/)
- [NTFY Documentation](https://docs.ntfy.sh/)
- [TP-Link Kasa Python library](https://python-kasa.readthedocs.io/)

## Change Log

- **2026-03-01**: Added Rocky Linux 9 support alongside Debian 12 — every role now dispatches to OS-specific task/vars files (`{Debian,RedHat}.yml`) via `ansible_os_family`, covering package management, repo setup, service names, PAM configuration, monitoring paths, and reboot-required detection
- **2026-02-23**: Added `os-upgrade.yml` rolling upgrade playbook and `docs/os-upgrade.md` — automates pre-upgrade health checks, standby/failover, iSCSI verification, and post-upgrade rejoin
- **2026-02-20**: Comprehensive CLAUDE.md update — added architecture overview, playbook tags, new monitoring exporters (stonith-probe, reboot-required-exporter), ZFS tunable notes, Sanoid snapshot policy, complete firewall port table, NFS security doc, dataset best practices doc, HTML design/ops docs, stonith-migration doc, full deployment workflow
- **2025-02-16**: Added Cockpit HA configuration (VIP + shared storage config sync)
- **2025-02-16**: Added mixed STONITH configuration support (per-node methods)
- **2025-02-16**: Added comprehensive monitoring (ZFS scrubs, cluster health)
- **2025-02-16**: Added ESPHome smart plug support for STONITH
- **Initial**: Base HA ZFS-over-iSCSI SAN playbook created
