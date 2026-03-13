# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible playbook for HA ZFS-over-iSCSI SAN (Debian 12, Ubuntu 22.04/24.04, Rocky Linux 9, AlmaLinux 9). Two storage nodes + quorum. Pacemaker active/passive HA; ZFS mirrors local disks with peer's iSCSI-exported disks.

## Key File Locations

| What | Where |
|------|-------|
| Global vars (VLANs, VIPs, Corosync, SSH, NTP) | `group_vars/all.yml` |
| Storage node vars, split by concern | `group_vars/storage_nodes/*.yml` |
| Per-node IPs, IQNs, disk paths | `host_vars/storage-[ab].yml` |
| Firewall template | `roles/hardening/templates/nftables.conf.j2` |
| STONITH script template | `roles/pacemaker/templates/configure-stonith.sh.j2` |
| iSCSI config manager CLI (setup/sync/save/strip subcommands) | `roles/services/templates/iscsi-config-manager.py.j2` |
| iSCSI deploy-time VLAN/TPG config (rendered to `/root/iscsi-vlan-config.json`) | `roles/services/templates/iscsi-vlan-config.json.j2` |
| iSCSI config sync path/service units | `roles/services/templates/iscsi-config-sync.{path,service}.j2` |
| Pacemaker resource config | `roles/pacemaker/templates/configure-resources.sh.j2` |

## Inventory Groups

- `cluster` — all 3 nodes (common, hardening, pacemaker, monitoring plays)
- `storage_nodes` — storage-a + storage-b only (ZFS, iSCSI, services, cockpit plays)

## Playbook Commands

```bash
ansible-playbook -i inventory.yml site.yml                    # full deploy
ansible-playbook -i inventory.yml site.yml --tags storage     # ZFS + iSCSI only
ansible-playbook -i inventory.yml site.yml --tags cluster     # Pacemaker only
ansible-playbook -i inventory.yml site.yml --tags services    # NFS/SMB/iSCSI services
ansible-playbook -i inventory.yml site.yml --tags monitoring  # exporters only
ansible-playbook -i inventory.yml site.yml --check --diff     # dry run
ansible-playbook -i inventory.yml verify.yml                  # post-deploy health check (read-only)
```

The first three plays in `site.yml` use `tags: always` — credential validation, cluster pre-check, and OS assertion run regardless of any `--tags` filter. Skip the cluster pre-check with `-e skip_cluster_check=true` on first deploy.

## Testing

```bash
cd molecule/default
molecule converge   # deploy to Docker containers
molecule verify     # run read-only checks
molecule test       # full converge + verify + destroy cycle
```

Molecule only tests `common` and `hardening` — ZFS, iSCSI, and Pacemaker require real hardware and are excluded from automated testing.

## Development Rules

### Firewall — Always Check When Adding Services

Any new network-accessible component needs a rule in `roles/hardening/templates/nftables.conf.j2`.

Port assignments:
- Management (10.20.20.0/24): SSH 22, Corosync 5405/udp, pcsd 2224, Cockpit 9090, node_exporter 9100, ha_cluster_exporter 9664
- Storage (10.10.10.0/24): iSCSI 3260, Corosync ring1 5405/udp
- Client VLANs: auto-generated from `client_vlans[].services` — nfs→2049, smb→445, iscsi→3260, ssh→22

### Multi-OS Dispatch Pattern

Two-layer dispatch: load `{{ ansible_os_family }}.yml` vars first, then overlay `{{ ansible_distribution }}.yml` (skip_missing: true). Task files use `with_first_found`: `{{ ansible_distribution }}.yml` → `{{ ansible_os_family }}.yml`.

- `Ubuntu.yml`: only values that **differ** from `Debian.yml` — never duplicate entries
- AlmaLinux: uses `RedHat.yml` unchanged (binary-compatible with Rocky)
- Roles without `Ubuntu.yml` fall back to `Debian.yml` automatically

### Idempotency

All tasks must be idempotent:
- Use `creates:` with command/shell tasks
- For `pcs` commands: `failed_when: result.rc != 0 and 'already exists' not in result.stderr`
- Use `changed_when: false` for read-only commands

### Secrets

Never commit plain text secrets. All passwords must be `ansible-vault` encrypted.

The pre-flight play in `site.yml` blocks deployment if any credential contains `CHANGEME`. Vault before deploying:
- `hacluster_password` (`group_vars/all.yml`)
- `iscsi_chap_password`, `iscsi_mutual_chap_password` (`group_vars/storage_nodes/iscsi.yml`)
- `stonith_nodes.<node>.password` (`group_vars/storage_nodes/cluster.yml`)

`iscsi_mutual_chap_password` must differ from `iscsi_chap_password` — iSCSI rejects identical bidirectional credentials.

### iSCSI Per-VLAN TPG Isolation

Each iSCSI-enabled client VLAN gets its own LIO TPG. Per-VLAN fields in `client_vlans` (`group_vars/all.yml`):

```yaml
- name: hypervisor
  services: [iscsi, ssh]
  iscsi_acls:                          # required when multiple VLANs use iSCSI
    - "iqn.2025-01.lab.home:proxmox-a"  # plain string = ACL-only, no CHAP
    - iqn: "iqn.2025-01.lab.home:proxmox-b"  # dict = per-initiator CHAP
      chap_user: "proxmox-b"
      chap_password: !vault |...       # vault-encrypt in production
  iscsi_dataset: "iscsi/hypervisor"    # required when multiple VLANs use iSCSI
```

- Single-VLAN: both fields optional (fall back to `iscsi_client_acls` / `iscsi_client_zvol_dataset`)
- Backstore naming: single-VLAN = `<zvol>`, multi-VLAN = `<vlan>-<zvol>` (prevents VMID collisions)
- Template fails at deploy time if multi-VLAN config is missing required fields
- `generate_node_acls: true` — sets `generate_node_acls=1` + `demo_mode_write_protect=0` on the VLAN's TPG; required for Proxmox ZFS-over-iSCSI plugin (which doesn't support per-IQN ACLs). When set, `iscsi_acls` is not required and no individual ACL entries are created. `iscsi-config-manager.py sync` skips ACL sync for this TPG.
- Per-TPG CHAP (for `generate_node_acls` VLANs): add `iscsi_chap_user` + `iscsi_chap_password` at VLAN level — all initiators share one credential.
- **CHAP is implicit** — no toggle needed. Credentials present = `authentication=1`; no credentials = `authentication=0`.

### iSCSI CHAP Encrypted Credentials

CHAP credentials are never stored in plaintext on disk. Ansible deploys `/root/.iscsi-chap.env.enc` (mode 0600, encrypted with OpenSSL AES-256-CBC using `/etc/machine-id` as passphrase). `iscsi-config-manager.py` decrypts this file at runtime using `openssl enc -d` and parses the JSON result. The JSON structure is: `{"per_initiator": {"tpg:iqn": {"user": "...", "pass": "..."}}, "per_tpg": {"tpg_num": {"user": "...", "pass": "..."}}}` — keyed `"tpg:iqn"` for per-initiator ACL-mode VLANs and by TPG number string for `generate_node_acls` VLANs.

- **Machine-specific**: the encrypted file is useless on any other node
- **Password rotation**: re-run `ansible-playbook ... --tags services` → new encrypted file deployed → re-run setup script
- **Global CHAP defaults removed**: `iscsi_client_chap_enabled` / `iscsi_client_chap_user` / `iscsi_client_chap_password` no longer exist. Credentials live at initiator or VLAN level only.

### iSCSI LUN Auto-Sync

`sync-iscsi-luns` Pacemaker resource (`iscsi-config-manager.py sync`) restores the exact client-facing LIO config (LUN numbers, ACLs, CHAP) from shared ZFS storage after each pool import. LUN numbers are preserved across failovers — no reordering. TPG→dataset mappings are baked into `/root/iscsi-vlan-config.json` at deploy time (no runtime file dependency).

The client config is split from the node-local `saveconfig.json`:
- **Node-local `saveconfig.json`**: backend target only (disk backstores for ZFS replication peer)
- **`/san-pool/cluster-config/iscsi/client-saveconfig.json`**: client-facing target (zvol LUNs, ACLs, CHAP) — authoritative source of truth

A systemd path unit (`iscsi-config-sync.path`) watches `saveconfig.json` for changes and runs `iscsi-config-manager.py save` in near-real-time. `ExecStop` on the sync service calls `save` during planned failover.

To add an initiator ACL without re-running Ansible: `targetcli /iscsi/<iqn>/tpg<N>/acls create <initiator_iqn> && targetcli saveconfig` — the path watcher will persist the change to shared storage automatically.

### Pacemaker Resource Ordering

```
zfs-pool → san-services (portblock-<vlan>-<svc>... → vip-<vlan>... → sync-iscsi-luns → nfs-server → smb-server → portunblock-<vlan>-<svc>...)
zfs-pool → cockpit-group (vip-cockpit → cockpit-service)
```

Portblock resources DROP packets on each VIP:port (iSCSI=3260, NFS=2049, SMB=445) while services start, preventing clients from receiving TCP RST during the startup window. Portunblock lifts the DROP rules and sends TCP tickles when all services are ready. Uses `ocf:heartbeat:portblock` (from `resource-agents`, already installed). Requires `iptables-nft` (present on Debian 12 and RHEL 9 by default as a transitive dependency).

### Config Files on Shared Storage

NFS/SMB configs are symlinked to shared ZFS storage for failover sync:
- `/etc/exports` → `/san-pool/cluster-config/nfs/exports`
- `/etc/samba/smb.conf` → `/san-pool/cluster-config/samba/smb.conf`

Always edit the shared storage location, not the local path.

### Disk Paths

Always use `/dev/disk/by-id/` paths in `host_vars/`. Never `/dev/sdX` (reorders on reboot) or `wwn-*` (hardware-specific).

### ZFS Pool Creation Is Manual

The playbook stops before `zpool create`. Run the generated `/root/create-pool.sh` (auto-discovers iSCSI paths from `/dev/disk/by-path/`), then `zpool export san-pool` before Pacemaker starts.

### STONITH

- `pcs stonith fence <node>` powers off immediately — only run in maintenance windows
- Post-fence verification (`stonith_fence_verify: true`): fence agent + `fence_check` ping all VLANs; both must succeed before failover proceeds
- ZFS resource start timeout (150s) must exceed max fencing latency (ipmi: 20–30s, kasa: 5–15s). Adjust `pcmk_reboot_timeout` in `configure-stonith.sh.j2` and `op start timeout` in `configure-resources.sh.j2` if needed.

### Adding Monitoring Exporters

Textfile exporters write `.prom` to `/var/lib/prometheus/node-exporter/` for collection by node_exporter on port 9100. ha_cluster_exporter uses port 9664.

When adding a new exporter:
1. Add script + systemd templates to `roles/monitoring/templates/`
2. Add deploy tasks to `roles/monitoring/tasks/main.yml`
3. Add firewall rule if using its own port
4. Add alert rules to `docs/prometheus-alerts.yml`

### Adding New Services or Roles

1. Create role, add to `site.yml` with the correct hosts group and tag
2. Check firewall rules (`roles/hardening/templates/nftables.conf.j2`)
3. Add vars to appropriate `group_vars/` file
4. Implement OS-specific task/vars files following the multi-OS dispatch pattern
5. All optional features must default to `false` — never enable by default if it could affect a running cluster

### ZFS Tunables Warning

`zfs_nocacheflush: 1` bypasses drive write-cache flush — **only safe on enterprise SSDs with hardware power-loss protection (PLP)**. Never enable on consumer drives or spinning disks. Configured in `group_vars/storage_nodes/zfs.yml`.

`zfs_arc_max` is computed as 50% of RAM at deploy time. Override per-host in `host_vars/<node>.yml`.
