# Development Guidelines for HA SAN Ansible Playbook

This file contains guidelines and reminders for maintaining and extending this Ansible playbook.

## What This Playbook Does

This is an **Ansible playbook for deploying a high-availability ZFS-over-iSCSI SAN** on Debian 12, Ubuntu 22.04/24.04, Rocky Linux 9, or AlmaLinux 9. The architecture is:

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
│   # Each role follows the same pattern: tasks/{main,Debian,Ubuntu,RedHat}.yml + vars/{Debian,Ubuntu,RedHat}.yml
│   # Ubuntu.yml overrides only values that differ from Debian.yml (loaded second, wins on conflict)
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
    ├── stonith-smart-plugs.md     # Smart plug fencing guide (Kasa, Tasmota, ESPHome, HTTP)
    ├── watchdog.md                # Hardware/software watchdog setup, module options, troubleshooting
    ├── ubuntu-notes.md            # Ubuntu/AlmaLinux-specific notes: ufw, ZFS native, sanoid, ha_cluster_exporter
    └── os-upgrade.md              # Rolling OS upgrade guide — dist-upgrade and full reinstall
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
| Client VLANs (per `client_vlans`) | Per-VLAN, based on `services` list | See below |

**Per-VLAN service → port mapping** (firewall rules auto-generated from `client_vlans[].services`):
| Service | Ports |
|---------|-------|
| `nfs` | 2049/tcp (+ 20048, 111 if `nfs_v3_enabled`) |
| `smb` | 445/tcp |
| `iscsi` | 3260/tcp |
| `ssh` | 22/tcp (for Proxmox ZFS-over-iSCSI plugin) |

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

`python3-kasa` is installed automatically if any node uses method `kasa`. See `docs/stonith-smart-plugs.md` for all plug types and options.

**Post-fence verification** is enabled by default (`stonith_fence_verify: true`). After the primary fence agent runs, a custom `fence_check` agent queries the STONITH device power state and pings the target on all three VLANs. If either check fails, Pacemaker considers fencing incomplete and **blocks failover** to prevent split-brain. Both agents are placed in **fencing level 1** — all must succeed for fencing to complete:

```bash
pcs stonith level    # Should show: Level 1 - storage-b: fence-storage-b,fence-verify-storage-b
fence_check -o status -n storage-b    # Non-destructive manual check
```

To disable: set `stonith_fence_verify: false` in `group_vars/storage_nodes/cluster.yml`, re-run the playbook, and re-run `configure-stonith.sh`. See `docs/stonith-smart-plugs.md#post-fence-verification` for troubleshooting a blocked failover.

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
- **Shared config (not symlinked — read by sync script):**
  - `/san-pool/cluster-config/iscsi/acls.conf` — iSCSI initiator ACL list (one IQN per line)
- **Cockpit VIP:** 10.20.20.10 (management VLAN) follows active node
- **Cockpit listen binding:** Socket drop-in (`cockpit.socket.d/listen.conf`) binds to `mgmt_ip:9090` and `vip_cockpit:9090` only (defense-in-depth; firewall also restricts)
- **Managed by Pacemaker:** Cockpit service is part of cockpit-group
- **Documentation:** `docs/cockpit-ha-config.md` for detailed configuration and troubleshooting

**When editing NFS/SMB configs manually:**
- Always edit the shared storage location, not the symlink target
- Example: `vim /san-pool/cluster-config/nfs/exports` (correct)
- NOT: `vim /etc/exports` (works but less obvious it's on shared storage)

### 3a. iSCSI LUN Auto-Sync (Client-Facing Target)

**Problem:** LIO backstore-to-LUN mappings are stored in `/etc/rtslib-fb-target/saveconfig.json` — a node-local file. Zvols created after initial setup (via Houston Cockpit, ZFS-over-iSCSI Proxmox plugin, or manual `zfs create -V`) only get LUN mappings on the node where they were created. On failover, the other node doesn't know about the new LUNs.

**Solution:** The `sync-iscsi-luns` Pacemaker resource runs after every pool import and auto-discovers zvols under `san-pool/iscsi/`, mapping any unmapped zvols as LIO backstores + LUNs and cleaning up stale backstores for deleted zvols.

**Resource ordering:**
```
zfs-pool → san-services (vip-enduser → vip-hypervisor → sync-iscsi-luns → nfs-server → smb-server)
zfs-pool → cockpit-group (vip-cockpit → cockpit-service)
```

**Files:**
- `roles/services/templates/sync-iscsi-luns.sh.j2` — auto-discover script
- `roles/services/templates/sync-iscsi-luns.service.j2` — systemd oneshot for Pacemaker
- `roles/pacemaker/templates/configure-resources.sh.j2` — `san-services` resource group

**Configuration:** `iscsi_client_zvol_dataset` (default: `iscsi`) in `roles/services/defaults/main.yml` controls which dataset is scanned. All zvols directly under `san-pool/<dataset>/` are auto-mapped.

**When creating new zvols for iSCSI clients:**
- Create the zvol under `san-pool/iscsi/` using any method (Houston, CLI, plugin)
- The next failover (or manual `bash /root/sync-iscsi-luns.sh`) maps it as a LUN
- No need to re-run Ansible or manually configure targetcli on both nodes

**ACL sync:** Initiator ACLs are synced from per-VLAN `acls-<vlan-name>.conf` files on shared storage. Seeded by `setup-client-iscsi-target.sh`. To add a new initiator without re-running Ansible:
1. Edit `/san-pool/cluster-config/iscsi/acls-<vlan-name>.conf` — add the IQN
2. Run `bash /root/sync-iscsi-luns.sh` (or wait for next failover)

### 3b. Per-VLAN iSCSI TPG Isolation

Each iSCSI-enabled client VLAN gets its own LIO Target Portal Group (TPG) with isolated portals, ACLs, and LUN mappings. This provides network-level isolation on top of existing CHAP and IQN ACL controls.

**Single-VLAN deployments:** One VLAN → one TPG (tpg1). Functionally identical to the old behavior. No extra configuration needed.

**Multi-VLAN deployments:** Each VLAN gets its own TPG (tpg1, tpg2, ...) with:
- One portal bound to that VLAN's VIP
- Independent ACL list (`acls-<vlan-name>.conf` on shared storage)
- Independent LUN namespace scoped to `iscsi_dataset`

**Per-VLAN variables** (set in `client_vlans` in `group_vars/all.yml`):
```yaml
- name: hypervisor
  services: [iscsi, ssh]
  iscsi_acls:            # required when multiple VLANs use iSCSI
    - "iqn.2025-01.lab.home:proxmox-a"
  iscsi_dataset: "iscsi/hypervisor"  # required when multiple VLANs use iSCSI
```
- `iscsi_acls` — per-VLAN initiator IQN whitelist. Falls back to `iscsi_client_acls` (iscsi.yml) for single-VLAN deployments.
- `iscsi_dataset` — ZFS dataset to scan for zvols. Falls back to `iscsi_client_zvol_dataset` (default: `iscsi`) for single-VLAN. Sub-datasets must be created in `zfs_datasets` (zfs.yml).

**Multi-VLAN template validation:** If `iscsi_vlans | length > 1`, the Jinja2 template fails clearly if any iSCSI VLAN is missing `iscsi_acls` or `iscsi_dataset`.

**Backstore naming:**
- Single-VLAN: `<zvol_name>` (unchanged — backward-compatible with existing setups)
- Multi-VLAN: `<vlan_name>-<zvol_name>` (prevents VMID collisions when multiple Proxmox clusters share the same VMID range)

**Key implementation notes:**
- `generate_node_acls=0` is set on every TPG — LIO does NOT auto-generate ACLs
- `setup-client-iscsi-target.sh` always re-runs (idempotent) to pick up VLAN changes
- `sync-iscsi-luns.sh` Phase 0 bakes in TPG→dataset/VLAN/prefix at Ansible deploy time (no runtime file dependency)
- ACL files: per-VLAN `acls-<vlan>.conf` + legacy union `acls.conf` (all VLANs combined)

**Upgrade from single-TPG to per-VLAN TPGs:**
1. Add `iscsi_acls` and `iscsi_dataset` to the iSCSI VLAN in `client_vlans` (or leave absent for single-VLAN fallback)
2. Create sub-dataset if needed: `zfs create san-pool/iscsi/hypervisor`
3. Re-run playbook: `ansible-playbook -i inventory.yml site.yml --tags services`
4. Verify: `targetcli ls /iscsi/` — one TPG per iSCSI VLAN, each with its VLAN's VIP
5. For multi-VLAN only: old un-prefixed backstores remain. Move zvols to the correct sub-dataset and re-run sync, or delete old backstores manually via `targetcli /backstores/block delete <name>`.

**Removing an iSCSI VLAN:** Remove it from `client_vlans`, re-run the playbook. The setup script warns about stale TPGs but does not auto-delete (requires manual cleanup: remove ACLs, portals, LUNs, then the TPG itself via `targetcli`).

### 4. Monitoring Changes

The monitoring role deploys these exporters on **all cluster nodes** (`hosts: cluster`):

| Exporter | Port | Purpose | Template/Source |
|----------|------|---------|----------------|
| `prometheus-node-exporter` | 9100 | System metrics | apt package |
| `prometheus-hacluster-exporter` | 9664 | Pacemaker/Corosync metrics | `ha-cluster-exporter-default.j2` |
| `zfs-scrub-exporter` (timer) | 9100 (textfile) | ZFS scrub state/timing | `zfs-scrub-exporter.{sh,service,timer}.j2` |
| `reboot-required-exporter` (timer) | 9100 (textfile) | Pending reboot flag | `reboot-required-exporter.{sh,service,timer}.j2` |
| `stonith-probe` (timer, storage nodes only) | 9100 (textfile) | STONITH agent reachability | `stonith-probe.{sh,service,timer}.j2` |
| `hwtemp-exporter` (timer) | 9100 (textfile) | NIC/HBA/SFP temperatures | `hwtemp-exporter.{sh,service,timer}.j2` |

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
hwtemp_monitoring_enabled: true       # toggles NIC/HBA/SFP temperature exporter (graceful no-op if no hwmon)
mellanox_mft_install: false           # installs MLNX_OFED MFT for ConnectX-3 chip temps (opt-in)
```

### 5. Per-Node vs. Global Configuration

**Per-node settings** (`host_vars/<hostname>.yml`):
- Disk layouts (`local_data_disks`) — must use stable `/dev/disk/by-id/` paths, never `/dev/sdX`
- IP addresses (`mgmt_ip`, `storage_ip`, `client_ips` dict keyed by VLAN name)
- iSCSI IQNs (`iscsi_target_iqn`, `iscsi_initiator_name`, `iscsi_peer_ip`, `iscsi_peer_iqn`)
- Optional `zfs_arc_max` override (otherwise computed as 50% of RAM at deploy time)
- Optional `slog_disk` and `special_disk` for NVMe SLOG/special vdevs

**Global settings** (`group_vars/all.yml`):
- VLAN configuration, `client_vlans` list (VIPs, subnets, per-VLAN services), `vip_cockpit`
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

### 6. Multi-OS Support (Debian 12 / Ubuntu 22.04/24.04 / Rocky Linux 9 / AlmaLinux 9)

The playbook supports **Debian 12**, **Ubuntu 22.04/24.04**, **Rocky Linux 9**, and **AlmaLinux 9** on all nodes. Each role uses a two-layer dispatch:

1. **OS-family vars** (`Debian.yml` / `RedHat.yml`) — base values for the whole family
2. **Distribution overrides** (`Ubuntu.yml`, `Ubuntu-22.yml`) — only values that differ from the family base (loaded second, wins on conflict)
3. **Task dispatch** uses `with_first_found`: `{{ ansible_distribution }}.yml` → `{{ ansible_os_family }}.yml` — roles without an `Ubuntu.yml` fall back to `Debian.yml` automatically

**Role structure pattern:**
```
roles/<role>/
├── tasks/
│   ├── main.yml        # Loads OS vars + overrides, dispatches to OS-specific tasks, then shared tasks
│   ├── Debian.yml      # Debian-specific: apt packages, repos, service names
│   ├── Ubuntu.yml      # Ubuntu overrides (only where Ubuntu differs from Debian)
│   └── RedHat.yml      # Rocky/AlmaLinux: dnf packages, repos, service names
├── vars/
│   ├── Debian.yml          # Debian package names, paths, service names
│   ├── Ubuntu.yml          # Ubuntu overrides (e.g. no zfs-dkms, empty ha_cluster_exporter_package)
│   ├── Ubuntu-22.yml       # Ubuntu 22.04-only overrides (e.g. PAM faillock no-op)
│   └── RedHat.yml          # Rocky/AlmaLinux package names, paths, service names
```

**AlmaLinux 9:** Binary-compatible with Rocky Linux 9. Uses existing `RedHat.yml` files — no new files needed.

**Ubuntu roles that fall back to Debian.yml (no Ubuntu.yml):**
`iscsi-target`, `iscsi-initiator`, `services`, `networking` — packages are identical.

**Key differences between supported OSes:**

| Area | Debian 12 | Ubuntu 22.04/24.04 | Rocky / AlmaLinux 9 |
|------|-----------|-------------------|---------------------|
| Package manager | `apt` | `apt` | `dnf` |
| Auto-updates | `unattended-upgrades` | `unattended-upgrades` | `dnf-automatic` |
| Default firewall | nftables | **ufw** (masked by playbook) | nftables (firewalld disabled) |
| PAM faillock | `pam-auth-update` | 22.04: no-op; 24.04: `pam-auth-update` | `authselect enable-feature` |
| ZFS source | `deb.debian.org` contrib + DKMS | **universe** (native kernel module) | OpenZFS ELRepo RPM + DKMS |
| ZFS packages | `zfsutils-linux`, `zfs-dkms`, `zfs-zed` | `zfsutils-linux`, `zfs-zed` (no DKMS) | `zfs`, `zfs-dkms` |
| Sanoid | apt | 24.04: apt; **22.04: manual install** | EPEL |
| ha_cluster_exporter | apt | **not in repos — manual install** | manual install |
| 45Drives plugins | yes | yes (dedicated Ubuntu repo) | yes (Rocky enterprise repo) |
| python3-kasa | apt | 24.04: apt; 22.04: pip fallback | pip |
| NTP servers | `*.debian.pool.ntp.org` | `ntp.ubuntu.com` | `*.rocky.pool.ntp.org` |
| Chrony config | `/etc/chrony/chrony.conf` | `/etc/chrony/chrony.conf` | `/etc/chrony.conf` |
| Reboot check | `/var/run/reboot-required` | `/var/run/reboot-required` | `needs-restarting -r` |

**When adding new tasks:**
- Use `ansible.builtin.package` for simple installs that share the same package name across all distros
- Use OS-specific task files when package names differ, repos need adding, or behaviour diverges
- Always check `vars/Debian.yml`, `vars/Ubuntu.yml`, and `vars/RedHat.yml` when adding packages
- Ubuntu overrides: only add values that **differ** from `Debian.yml` — do not duplicate
- See `docs/ubuntu-notes.md` for Ubuntu/AlmaLinux-specific caveats and workarounds

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
- Files with secrets: `group_vars/storage_nodes/` directory, `host_vars/*.yml`

The playbook has a **pre-flight validation play** (first play in `site.yml`) that fails immediately if any credential still contains the string `CHANGEME`. Vault these before deploying:
- `hacluster_password` (`group_vars/all.yml`)
- `iscsi_chap_password` (`group_vars/storage_nodes/iscsi.yml`)
- `iscsi_mutual_chap_password` (`group_vars/storage_nodes/iscsi.yml`)
- `stonith_nodes.<node>.password` (`group_vars/storage_nodes/cluster.yml`)

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

1. Update `stonith_nodes` dict in `group_vars/storage_nodes/cluster.yml`
2. Run playbook: `ansible-playbook -i inventory.yml site.yml --tags cluster`
3. Regenerate config: `ssh storage-a 'sudo bash /root/configure-stonith.sh'`
4. Verify: `pcs stonith status`
5. Test (careful!): `pcs stonith fence storage-b` (powers off node!)

### Adding or Changing ZFS Datasets

1. Update `zfs_datasets` in `group_vars/storage_nodes/zfs.yml`
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
- **Client VLANs:** Per-VLAN rules from `client_vlans[].services` — nfs→2049, smb→445, iscsi→3260, ssh→22

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
pcs resource move san-services storage-b
# Wait for resources to come up on storage-b
pcs resource clear san-services  # remove location constraint

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

- **2026-03-09**: Per-VLAN iSCSI TPG isolation — each iSCSI-enabled client VLAN now gets its own LIO TPG with isolated portals, ACLs, and LUN namespace. Per-VLAN `iscsi_acls` and `iscsi_dataset` fields added to `client_vlans`. Multi-VLAN backstore naming uses `<vlan>-<zvol>` prefix to prevent VMID collisions; single-VLAN keeps existing names (safe upgrade). `setup-client-iscsi-target.sh` always re-runs (idempotent). `sync-iscsi-luns.sh` rewritten: Phase 0 bakes in TPG→VLAN mapping, Phase 1 extended python3 parser emits per-TPG LUN/ACL state, Phase 2 per-TPG zvol scanning, Phase 3 per-VLAN ACL file sync. Template fails clearly at deploy time if multi-VLAN config is missing required fields.
- **2026-03-09**: Multi-VLAN client networks — replaced single `vlans.client` / `vip_nfs` / `vip_smb` / `vip_iscsi` with `client_vlans` list (per-VLAN VIP, subnet, services). Per-node `client_ip` → `client_ips` dict. Flat `san-services` Pacemaker group replaces per-service groups. Per-VLAN nftables rules auto-generated from `services` list. iSCSI portals bound to VIPs. SSH on client VLANs for Proxmox. `fence_check` pings all client IPs. `ip_nonlocal_bind=1` sysctl for HA VIP binding.
- **2026-03-09**: Cockpit management binding — systemd socket drop-in binds `cockpit.socket` to `mgmt_ip:9090` + `vip_cockpit:9090` only (defense-in-depth).
- **2026-03-08**: Optimized `sync-iscsi-luns.sh` — replaced per-item `targetcli ls` existence checks and individual write spawns with (1) a single `python3` invocation that parses `saveconfig.json` into bash associative arrays, and (2) a single batched `targetcli` stdin invocation for all changes. Reduces ~40 process spawns (15 zvols + 5 ACLs) to exactly 2 — expected runtime <5s vs previous 30–90s.
- **2026-03-06**: Added LIO ACL sync — `sync-iscsi-luns.sh` Phase 3 reads `/san-pool/cluster-config/iscsi/acls.conf` (one IQN per line) and ensures ACLs match on whichever node is active. `setup-client-iscsi-target.sh` seeds the file from `iscsi_client_acls`. Operators can edit the shared file directly without re-running Ansible.
- **2026-03-06**: Added iSCSI LUN auto-sync — `sync-iscsi-luns` Pacemaker resource auto-discovers zvols under `san-pool/iscsi/` after pool import and maps them as LIO LUNs. Handles zvols created by any method (Houston Cockpit, ZFS-over-iSCSI Proxmox plugin, manual `zfs create -V`). Cleans up stale backstores for deleted zvols. New `iscsi-group` resource group orders sync before `vip-iscsi`. Existing clusters must re-run `configure-pacemaker-resources.sh` and remove the old standalone `vip-iscsi` constraint.
- **2026-03-05**: Added `hwtemp-exporter` — NIC chip (sysfs hwmon), SFP/QSFP transceiver (ethtool), and HBA controller (FC + SAS allowlist) temperature monitoring; optional Mellanox MFT install (`mellanox_mft_install: false` opt-in) for ConnectX-3 mlx4 cards without hwmon; 5 new Prometheus alert rules (NICTemperatureHigh/Critical, SFPTemperatureHigh, HBATemperatureHigh, HWTempExporterFailed); MFT GPG fingerprint pinned in `group_vars/all.yml`
- **2026-03-04**: Added Ubuntu 22.04/24.04 and AlmaLinux 9 support — two-layer dispatch pattern (OS-family vars + distribution overrides), Ubuntu-specific files for ZFS (native modules, no DKMS), hardening (ufw disabled → nftables), monitoring (ha_cluster_exporter unavailable notice), pacemaker (kasa pip fallback), cockpit (45Drives opt-in); AlmaLinux uses existing RedHat.yml files unchanged; OS validation assertion added to site.yml; `docs/ubuntu-notes.md` added
- **2026-03-04**: Added per-node watchdog overrides — any `watchdog_*` variable can be set in `host_vars/<node>.yml` to use a different module or settings per node (e.g. `iTCO_wdt` on storage-a, `softdog` on storage-b); added commented examples to `host_vars/storage-{a,b}.yml` and per-node configuration section in `docs/watchdog.md`
- **2026-03-04**: Added watchdog support — kernel module loading (`watchdog_module`, defaults to `softdog`), boot persistence via `modules-load.d`, optional modprobe options via `watchdog-modprobe.conf.j2`, realtime daemon scheduling, and `docs/watchdog.md` covering hardware/software modules, STONITH relationship, verification, and troubleshooting
- **2026-03-01**: Added Rocky Linux 9 support alongside Debian 12 — every role now dispatches to OS-specific task/vars files (`{Debian,RedHat}.yml`) via `ansible_os_family`, covering package management, repo setup, service names, PAM configuration, monitoring paths, and reboot-required detection
- **2026-02-23**: Added `os-upgrade.yml` rolling upgrade playbook and `docs/os-upgrade.md` — automates pre-upgrade health checks, standby/failover, iSCSI verification, and post-upgrade rejoin
- **2026-02-20**: Comprehensive CLAUDE.md update — added architecture overview, playbook tags, new monitoring exporters (stonith-probe, reboot-required-exporter), ZFS tunable notes, Sanoid snapshot policy, complete firewall port table, NFS security doc, dataset best practices doc, HTML design/ops docs, full deployment workflow
- **2025-02-16**: Added Cockpit HA configuration (VIP + shared storage config sync)
- **2025-02-16**: Added mixed STONITH configuration support (per-node methods)
- **2025-02-16**: Added comprehensive monitoring (ZFS scrubs, cluster health)
- **2025-02-16**: Added ESPHome smart plug support for STONITH
- **Initial**: Base HA ZFS-over-iSCSI SAN playbook created
