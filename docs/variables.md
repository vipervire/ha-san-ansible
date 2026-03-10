# Variables Reference

This document lists all configurable variables for the HA ZFS-over-iSCSI SAN playbook. Variables are organized by file and concern.

**Variable Precedence** (lowest → highest): role `defaults/main.yml` < `group_vars/all.yml` < `group_vars/storage_nodes/*.yml` < `host_vars/<node>.yml`

---

## Cluster & Identity

Defined in `group_vars/all.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `cluster_name` | `san-cluster` | string | Corosync cluster name |
| `hacluster_password` | — | string | **Vault required.** Password for the `hacluster` OS user used by Pacemaker/pcsd |
| `admin_user` | `storageadmin` | string | Ansible and SSH admin username created on all nodes |
| `admin_ssh_pubkey` | — | string | SSH public key for `admin_user`. Replace the placeholder before deploying |

---

## Network — VLANs & Interfaces

Defined in `group_vars/all.yml` (VLAN topology) and `group_vars/storage_nodes/network.yml` (interface names).

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `vlans.storage.id` | `10` | int | Storage VLAN tag (iSCSI replication) |
| `vlans.storage.subnet` | `10.10.10.0/24` | string | Storage VLAN subnet |
| `vlans.storage.mtu` | `9000` | int | MTU — use 9000 for jumbo frames |
| `vlans.management.id` | `20` | int | Management VLAN tag |
| `vlans.management.subnet` | `10.20.20.0/24` | string | Management VLAN subnet |
| `vlans.management.mtu` | `1500` | int | MTU for management VLAN |
| `vip_cockpit` | `10.20.20.10` | string | Floating VIP for Cockpit on management VLAN |
| `vip_mgmt_cidr` | `24` | int | CIDR prefix for management VIP |
| `net_storage_parent` | `ens3f0` | string | Physical interface for storage VLAN (40GbE recommended) |
| `net_client_parent` | `ens3f1` | string | Physical interface for client VLANs |
| `net_mgmt_interface` | `eno1` | string | Management interface (Corosync ring0, SSH) |

### TCP Tunables

Defined in `group_vars/storage_nodes/network.yml` as a dict `tcp_tunables`. These are written to `/etc/sysctl.d/`.

| Key | Default | Description |
|-----|---------|-------------|
| `net.core.rmem_max` | `16777216` | Socket receive buffer maximum |
| `net.core.wmem_max` | `16777216` | Socket send buffer maximum |
| `net.ipv4.tcp_rmem` | `4096 87380 16777216` | TCP receive buffer min/default/max |
| `net.ipv4.tcp_wmem` | `4096 65536 16777216` | TCP send buffer min/default/max |
| `net.ipv4.tcp_congestion_control` | `bbr` | Congestion control algorithm |
| `net.core.default_qdisc` | `fq` | Queue discipline (required for BBR) |

### Client VLANs

`client_vlans` is a list in `group_vars/all.yml`. Each entry represents a network serving storage to clients.

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Short name used as a key throughout the playbook |
| `id` | yes | VLAN tag |
| `subnet` | yes | CIDR subnet |
| `mtu` | no | MTU (default 1500) |
| `vip` | yes | Floating VIP address for this VLAN |
| `vip_cidr` | yes | CIDR prefix for VIP |
| `services` | yes | List of services: `nfs`, `smb`, `iscsi`, `ssh` |
| `description` | no | Human-readable label |
| `parent_interface` | no | Override parent NIC; defaults to `net_client_parent` |
| `iscsi_acls` | yes (multi-VLAN iSCSI) | List of initiator IQNs allowed on this VLAN's TPG. Entries are plain strings or dicts with `iqn`, `chap_user`, `chap_password`. Not required when `generate_node_acls: true` |
| `iscsi_dataset` | yes (multi-VLAN iSCSI) | ZFS dataset path for this VLAN's zvols (e.g. `iscsi/hypervisor`). Falls back to `iscsi_client_zvol_dataset` in single-VLAN deployments |
| `generate_node_acls` | no | Set `true` to allow any initiator (required for Proxmox ZFS-over-iSCSI plugin). Disables per-IQN ACL enforcement |
| `iscsi_chap_user` | no | Per-TPG CHAP username (only used when `generate_node_acls: true`) |
| `iscsi_chap_password` | no | **Vault recommended.** Per-TPG CHAP password (only used when `generate_node_acls: true`) |

---

## Corosync

Defined in `group_vars/all.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `corosync_token` | `4000` | int (ms) | Time before a silent node is declared dead. Do not set below 3000 |
| `corosync_consensus` | `4800` | int (ms) | Time to reach quorum consensus (set to 1.2× token) |
| `corosync_join` | `1000` | int (ms) | Maximum time for a new member to join |
| `corosync_max_messages` | `20` | int | Maximum messages sent per token rotation |
| `corosync_transport` | `knet` | string | Transport: `knet` (default, multi-ring) or `udpu` |
| `corosync_crypto_cipher` | `aes256` | string | Encryption cipher for cluster traffic |
| `corosync_crypto_hash` | `sha256` | string | Hash algorithm for cluster authentication |
| `corosync_nodes` | (see below) | list | Node definitions. Each entry: `name`, `nodeid`. Ring addresses are derived from `mgmt_ip` and `storage_ip` in host_vars |

---

## SSH & Security

Defined in `group_vars/all.yml` and `roles/hardening/defaults/main.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `ssh_port` | `22` | int | SSH listen port |
| `ssh_allowed_users` | `[storageadmin]` | list | Users permitted to SSH in (AllowUsers directive) |
| `ssh_listen_addresses` | `[]` | list | IPs SSH listens on. Empty = all interfaces. Override per-host in host_vars |
| `auditd_enabled` | `false` | bool | Enable auditd to monitor sensitive files and privilege escalation |
| `syslog_remote_enabled` | `false` | bool | Enable rsyslog remote forwarding |
| `syslog_remote_host` | — | string | Syslog server IP or hostname |
| `syslog_remote_port` | `514` | int | Syslog server port |
| `syslog_remote_protocol` | `udp` | string | Transport protocol: `udp` or `tcp` |
| `firewall_rate_limit_enabled` | `true` | bool | Rate-limit Cockpit (9090) and pcsd (2224) against brute-force |
| `nfs_v3_enabled` | `false` | bool | Open NFSv3 firewall ports (111/tcp+udp, 20048). Must match `nfs_v3_enabled` in services role |
| `fortyfive_drives_gpg_fingerprint` | (pinned) | string | GPG fingerprint for 45Drives repo key. Verify out-of-band before changing |
| `mellanox_gpg_fingerprint` | (pinned) | string | GPG fingerprint for Mellanox repo key. Used when `mellanox_mft_install: true` |

### Watchdog

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `watchdog_enabled` | `false` | bool | Enable hardware/software watchdog daemon |
| `watchdog_module` | `softdog` | string | Kernel module: `softdog`, `iTCO_wdt`, `ipmi_watchdog`, `sp5100_tco` |
| `watchdog_module_options` | `{}` | dict | Extra modprobe options (e.g. `{soft_margin: "60"}`) |
| `watchdog_device` | `/dev/watchdog` | string | Watchdog device path |
| `watchdog_timeout` | `60` | int (s) | Watchdog timeout — reboot if not pet within this interval |
| `watchdog_interval` | `10` | int (s) | How often the daemon pets the watchdog |
| `watchdog_max_load` | `24.0` | float | Reboot if 1-minute load exceeds this value |
| `watchdog_min_memory` | `1` | int (MB) | Reboot if free memory drops below this value |

All watchdog variables can be overridden per-node in `host_vars/<node>.yml`.

---

## ZFS

Defined in `group_vars/storage_nodes/zfs.yml` and `roles/zfs/defaults/main.yml`.

### Pool & Dataset Options

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `zfs_pool_name` | `san-pool` | string | ZFS pool name |
| `zfs_pool_ashift` | `12` | int | Sector size hint (12 = 4K, 13 = 8K). Match to physical sector size |
| `zfs_pool_options` | (see below) | dict | Pool-level properties set at creation |
| `zfs_pool_options.multihost` | `on` | string | Prevents dual-import. Required for HA safety |
| `zfs_pool_options.autotrim` | `on` | string | Automatic TRIM for SSDs |
| `zfs_pool_options.autoreplace` | `on` | string | Auto-replace when a disk reappears in a faulted vdev slot |
| `zfs_dataset_options` | (see below) | dict | Default properties applied to all datasets |
| `zfs_datasets` | (list) | list | Datasets to create. Each entry: `name`, `properties` dict |
| `zfs_arc_max` | 50% RAM | int (bytes) | ARC size limit. Computed at deploy time. Override per-host in host_vars |

### ZFS Modprobe Tunables

Written to `/etc/modprobe.d/zfs.conf`. Defined as `zfs_modprobe_options` dict.

| Key | Default | Description |
|-----|---------|-------------|
| `zfs_vdev_scheduler` | `none` | I/O scheduler (none = let ZFS manage) |
| `zfs_txg_timeout` | `30` | TXG flush interval in seconds |
| `zfs_resilver_min_time_ms` | `27000` | Minimum resilver time per pass (OpenZFS recommendation) |
| `zfs_resilver_delay` | `0` | Delay between resilver passes |
| `zfs_nocacheflush` | `1` | Skip drive write-cache flush. **Only safe on enterprise SSDs with hardware PLP.** |

### ZFS Scrub

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `zfs_scrub_enabled` | `true` | bool | Enable scheduled scrubs |
| `zfs_scrub_schedule` | `*-*-01 02:00:00` | string | systemd calendar expression |
| `zfs_scrub_randomized_delay_sec` | `1800` | int | Random delay added to prevent simultaneous scrubs on both nodes |
| `zfs_scrub_skip_on_degraded` | `true` | bool | Skip scrub if pool is degraded |
| `zfs_scrub_min_interval_days` | `25` | int | Minimum days between scrubs |

### Sanoid Snapshot Policy

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `sanoid_install` | `true` | bool | Install and configure Sanoid |
| `sanoid_datasets` | (list) | list | Datasets to snapshot. Each entry: `name`, `template`, `recursive` |
| `sanoid_templates` | (dict) | dict | Snapshot retention templates. Keys: template names. Values: `hourly`, `daily`, `monthly`, `yearly`, `autosnap`, `autoprune` |

### ZED ntfy Notifications

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `zed_ntfy_enabled` | `false` | bool | Enable ZED → ntfy push notifications for pool events |
| `zed_ntfy_url` | `http://ntfy.example.com` | string | Base URL of your ntfy server |
| `zed_ntfy_topic` | `/ha-san-alerts` | string | ntfy topic path |

### Syncoid Replication

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `syncoid_enabled` | `false` | bool | Enable Syncoid offsite replication (requires `sanoid_install: true`) |
| `syncoid_schedule` | `*-*-* 03:00:00` | string | systemd calendar expression |
| `syncoid_targets` | `[]` | list | Replication targets. Each entry: `source` (dataset), `destination` (user@host:dataset) |

---

## iSCSI — Backend Replication

Defined in `group_vars/storage_nodes/iscsi.yml`. These variables control the peer-to-peer iSCSI sessions between storage nodes.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `iscsi_iqn_prefix` | `iqn.2025-01.lab.home` | string | IQN prefix for all iSCSI names in the cluster. Change to match your domain and date |
| `iscsi_chap_enabled` | `true` | bool | Enable CHAP on backend replication sessions |
| `iscsi_chap_user` | `iscsi-repl-user` | string | CHAP username for backend replication |
| `iscsi_chap_password` | — | string | **Vault required.** CHAP password for backend replication |
| `iscsi_mutual_chap_enabled` | `true` | bool | Enable mutual CHAP (initiator verifies target) |
| `iscsi_mutual_chap_user` | `iscsi-target-user` | string | Mutual CHAP username |
| `iscsi_mutual_chap_password` | — | string | **Vault required.** Mutual CHAP password. **Must differ from `iscsi_chap_password`** — iSCSI rejects identical bidirectional credentials |
| `iscsi_replacement_timeout` | `5` | int (s) | Seconds before a LUN is marked faulted after path loss |
| `iscsi_noop_out_interval` | `2` | int (s) | Heartbeat frequency |
| `iscsi_noop_out_timeout` | `2` | int (s) | Heartbeat timeout |
| `iscsi_login_timeout` | `10` | int (s) | Initial login timeout |
| `iscsi_queue_depth` | `32` | int | iSCSI queue depth per session (SATA SSD: 32–64, NVMe: 64–128) |
| `iscsi_cmds_max` | `128` | int | Maximum outstanding commands per session |

---

## iSCSI — Client-Facing Target

Defined in `group_vars/storage_nodes/iscsi.yml` and `roles/services/defaults/main.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `iscsi_client_iqn` | `{{ iscsi_iqn_prefix }}:san-client-target` | string | IQN of the client-facing LIO target |
| `iscsi_client_acls` | (list) | list | Single-VLAN fallback: initiator IQNs allowed to connect. Ignored when `iscsi_acls` is defined on the iSCSI VLAN entry in `client_vlans`. Entries are plain strings or dicts with `iqn`, `chap_user`, `chap_password` |
| `iscsi_client_zvol_dataset` | `iscsi` | string | Single-VLAN fallback: ZFS dataset containing zvols to export as LUNs. Overridden per-VLAN by `iscsi_dataset` in `client_vlans` |

**CHAP on client-facing targets is implicit** — credentials present = `authentication=1`; no credentials = `authentication=0`. No toggle needed.

**Encrypted credential storage:** CHAP credentials are never written plaintext to disk. Ansible deploys `/root/.iscsi-chap.env.enc` (AES-256-CBC, keyed to `/etc/machine-id`). Rotation: re-run `ansible-playbook --tags services`.

Per-VLAN iSCSI fields (`iscsi_acls`, `iscsi_dataset`, `generate_node_acls`, `iscsi_chap_user`, `iscsi_chap_password`) are documented in the [Client VLANs](#client-vlans) section above.

---

## NFS & SMB Services

Defined in `group_vars/storage_nodes/services.yml` and `roles/services/defaults/main.yml`.

### NFS

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `nfs_exports` | (list) | list | NFS export definitions. Each entry: `path`, `clients`, `options` |
| `nfs_v3_enabled` | `false` | bool | Enable NFSv3 (also opens firewall ports 111, 20048). Keep `false` unless clients require it |
| `nfs_v4_enabled` | `true` | bool | Enable NFSv4 |
| `nfs_v41_enabled` | `true` | bool | Enable NFSv4.1 (pNFS) |
| `nfs_threads` | `8` | int | Number of nfsd threads |
| `nfs_grace_time` | `45` | int (s) | Grace period for client state reclaim after failover |
| `nfs_lease_time` | `45` | int (s) | NFSv4 lease duration |

### SMB

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `smb_workgroup` | `HOMELAB` | string | Samba workgroup name |
| `smb_server_string` | `HA SAN SMB` | string | Server description shown to clients |
| `smb_encrypt` | `required` | string | SMB encryption: `required`, `desired`, or `off` |
| `smb_min_protocol` | `SMB3` | string | Minimum protocol version |
| `smb_signing` | `required` | string | Packet signing: `required`, `desired`, or `disabled` |
| `smb_shares` | (list) | list | Share definitions. Each entry: `name`, `path`, `read_only`, `valid_users`, `shadow_copy` |

---

## Pacemaker & STONITH

Defined in `group_vars/storage_nodes/cluster.yml` and `roles/pacemaker/defaults/main.yml`.

### Pacemaker Resource Configuration

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `pacemaker_preferred_node` | `storage-a` | string | Soft node preference (not a hard pin — resources migrate on failure) |
| `pacemaker_resource_stickiness` | `200` | int | Score added to keep resources on their current node |
| `pacemaker_standby_drain_seconds` | `15` | int | Seconds to wait for resource migration after `pcs node standby` before stopping Pacemaker |
| `stonith_fence_verify` | `true` | bool | After fencing, verify power-off via STONITH device and VLAN pings before allowing failover. Prevents split-brain on flaky fence agents |

### STONITH Nodes

`stonith_nodes` is a dict in `group_vars/storage_nodes/cluster.yml` keyed by node hostname.

| Field | Required | Description |
|-------|----------|-------------|
| `method` | yes | Fencing method: `ipmi`, `kasa`, `tasmota`, `esphome`, `http` |
| `ip` | yes | BMC IP or smart plug IP |
| `user` | no | Username (IPMI, ESPHome, Tasmota, HTTP) |
| `password` | no | **Vault recommended.** Password (IPMI, ESPHome, Tasmota, HTTP) |
| `switch_name` | no | ESPHome relay name (ESPHome only) |
| `power_off_url` | no | HTTP path for power-off (HTTP method only) |
| `power_on_url` | no | HTTP path for power-on (HTTP method only) |
| `status_url` | no | HTTP path for power status (HTTP method only) |
| `status_on_regex` | no | Regex matching the "on" state in HTTP status response |
| `status_off_regex` | no | Regex matching the "off" state in HTTP status response |

See `docs/stonith-smart-plugs.md` for per-method setup guides.

---

## Monitoring

Defined in `group_vars/storage_nodes/cluster.yml` and `roles/monitoring/defaults/main.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `ha_cluster_monitoring_enabled` | `true` | bool | Enable ha_cluster_exporter (Pacemaker/Corosync metrics) on port 9664 |
| `ha_cluster_exporter_port` | `9664` | int | ha_cluster_exporter listen port |
| `ha_cluster_exporter_address` | `{{ mgmt_ip }}` | string | ha_cluster_exporter listen address |
| `smart_monitoring_enabled` | `true` | bool | Enable SMART disk health textfile exporter |
| `zfs_scrub_monitoring_enabled` | `true` | bool | Enable ZFS scrub metrics textfile exporter |
| `ras_monitoring_enabled` | `false` | bool | Enable rasdaemon ECC memory error monitoring (requires ECC RAM) |
| `hwtemp_monitoring_enabled` | `true` | bool | Enable NIC/HBA/SFP temperature monitoring via sysfs hwmon |
| `mellanox_mft_install` | `false` | bool | Install NVIDIA Mellanox Firmware Tools for ConnectX-3 chip temperatures |
| `node_exporter_tls_enabled` | `false` | bool | Enable TLS on node_exporter (port 9100) |
| `node_exporter_cert_file` | — | string | Path to TLS cert on Ansible controller (requires `node_exporter_tls_enabled: true`) |
| `node_exporter_key_file` | — | string | Path to TLS key on Ansible controller (requires `node_exporter_tls_enabled: true`) |
| `alertmanager_ntfy_url` | `""` | string | If set, generates `alertmanager.yml` with ntfy webhook integration |

---

## Cockpit

Defined in `roles/cockpit/defaults/main.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `cockpit_tls_cert_file` | — | string | Path to custom TLS cert on Ansible controller. If unset, Cockpit uses a self-signed cert |
| `cockpit_tls_key_file` | — | string | Path to custom TLS key on Ansible controller |

---

## Networking Role

Defined in `roles/networking/defaults/main.yml`. This role is **opt-in** and disabled by default.

> **Warning:** Enabling `networking_manage` on a running system can disrupt network connectivity. Review templates before enabling.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `networking_manage` | `false` | bool | Master switch. Set `true` to allow the networking role to manage interfaces |
| `networking_backend` | `systemd-networkd` | string | Network backend to configure |
| `networking_restart_on_change` | `false` | bool | Allow interface restart when configuration changes |

---

## NTP

Defined in `group_vars/all.yml`.

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| `ntp_servers` | `[]` | list | NTP server addresses. Empty = OS-specific defaults (Debian pool, Rocky pool) |

---

## Per-Node Variables (host_vars)

Each storage node has `host_vars/<node>.yml`. The quorum node only needs `mgmt_ip`.

### storage-a / storage-b

| Variable | Required | Description |
|----------|----------|-------------|
| `storage_ip` | yes | Node IP on the storage VLAN (10.10.10.x) |
| `mgmt_ip` | yes | Node IP on the management VLAN (10.20.20.x) |
| `client_ips` | yes | Dict of per-VLAN IPs, keyed by `client_vlans[].name` (e.g. `enduser: 10.30.30.1`) |
| `ssh_listen_addresses` | no | Override SSH listen addresses for this node (typically just `mgmt_ip`) |
| `iscsi_target_iqn` | yes | IQN this node exports as a target (e.g. `{{ iscsi_iqn_prefix }}:storage-a`) |
| `iscsi_initiator_name` | yes | IQN this node uses as an initiator (e.g. `{{ iscsi_iqn_prefix }}:initiator-a`) |
| `iscsi_peer_ip` | yes | Peer storage node's IP on the storage VLAN |
| `iscsi_peer_iqn` | yes | Peer storage node's target IQN |
| `local_data_disks` | yes | List of data disk definitions. Each entry: `device` (persistent `/dev/disk/by-id/` path), `label` |
| `slog_disk` | no | SLOG (ZIL) device: `device` (by-id path), `label`. Accelerates sync writes |
| `special_disk` | no | Special vdev device: `device` (by-id path), `label` |
| `zfs_arc_max` | no | Override ARC size for this node in bytes (default: 50% RAM) |
| `watchdog_enabled` | no | Per-node watchdog override |
| `watchdog_module` | no | Per-node watchdog module override |
| `watchdog_module_options` | no | Per-node watchdog module options override |

> **Disk paths:** Always use `/dev/disk/by-id/` paths. Never use `/dev/sdX` (reorders on reboot) or `wwn-*` paths (hardware-specific, not portable). Use `ata-*`, `scsi-*`, or `nvme-*` identifiers.
>
> Discover paths: `ls -la /dev/disk/by-id/ | grep -v part | grep -v wwn`

### quorum

| Variable | Required | Description |
|----------|----------|-------------|
| `mgmt_ip` | yes | Node IP on the management VLAN |
| `ssh_listen_addresses` | no | Override SSH listen addresses |

---

## Vault-Required Variables

The pre-flight play in `site.yml` blocks deployment if any of these contain `CHANGEME`:

| Variable | File |
|----------|------|
| `hacluster_password` | `group_vars/all.yml` |
| `iscsi_chap_password` | `group_vars/storage_nodes/iscsi.yml` |
| `iscsi_mutual_chap_password` | `group_vars/storage_nodes/iscsi.yml` |
| `stonith_nodes.<node>.password` | `group_vars/storage_nodes/cluster.yml` |
| Per-initiator `chap_password` in `iscsi_acls` | `group_vars/all.yml` (client_vlans) |

Encrypt with: `ansible-vault encrypt_string 'yourpassword' --name 'variable_name'`
