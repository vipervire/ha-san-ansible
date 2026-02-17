# Development Guidelines for HA SAN Ansible Playbook

This file contains guidelines and reminders for maintaining and extending this Ansible playbook.

## Critical Checks When Making Changes

### 1. Firewall Rules (nftables)

**⚠️ IMPORTANT:** When adding new services, monitoring endpoints, or network-accessible components, **ALWAYS** check if firewall rules need updating.

**File to check:** `roles/hardening/templates/nftables.conf.j2`

**Common scenarios requiring firewall updates:**

- **Adding monitoring exporters** (e.g., node_exporter, ha_cluster_exporter, custom exporters)
  - Default ports: 9100 (node_exporter), 9664 (ha_cluster_exporter)
  - Add rules to management VLAN section

- **Adding web services** (e.g., Cockpit, Houston, monitoring dashboards)
  - Default ports: 9090 (Cockpit), 3000 (Grafana), 9090 (Prometheus)
  - Add rules to management VLAN section

- **Adding cluster services** (e.g., Pacemaker components, Corosync rings)
  - Ports: 2224 (pcsd), 5405 (Corosync), 21064 (Pacemaker)
  - Check if already present before adding

- **Adding storage protocols** (e.g., NFS, SMB, iSCSI)
  - NFS: 2049, 111
  - SMB: 445, 139
  - iSCSI: 3260
  - Add rules to client VLAN section

**Example firewall rule:**
```nftables
# Prometheus ha_cluster_exporter (Pacemaker/Corosync metrics)
ip saddr {{ vlans.management.subnet }} tcp dport 9664 accept
```

**Testing after changes:**
```bash
# Check nftables rules are applied
ssh storage-a
sudo nft list ruleset | grep <port-number>

# Test connectivity from expected source
curl http://10.20.20.1:<port>  # From management VLAN
```

### 2. STONITH Configuration

When modifying STONITH/fencing configuration:

- **File:** `roles/pacemaker/templates/configure-stonith.sh.j2`
- **Config:** `group_vars/storage_nodes.yml` (stonith_nodes dict)
- **Test:** Always test fencing in non-production before deploying
- **Document:** Update `docs/stonith-smart-plugs.md` if adding new fence agent types

**Supported methods:** ipmi, kasa, tasmota, esphome, http

### 3. Monitoring Changes

When adding new metrics or exporters:

- **Update documentation:** `docs/cluster-monitoring.md`, `docs/ntfy-integration.md`
- **Add alert rules:** `docs/prometheus-alerts.yml`
- **Check firewall:** See section 1 above
- **Test metrics:** Verify metrics are accessible from Prometheus server
- **Update README:** Add to "Monitoring and Alerting" section

### 4. Per-Node vs. Global Configuration

**Per-node settings** (in `host_vars/<hostname>.yml`):
- Disk layouts (`local_data_disks`)
- IP addresses (mgmt_ip, storage_ip, client_ip)
- iSCSI IQNs (initiator and target)

**Global settings** (in `group_vars/all.yml` or `group_vars/storage_nodes.yml`):
- VLAN configuration
- Virtual IPs (VIPs)
- Service ports
- Cluster settings
- STONITH configuration (now per-node in stonith_nodes dict)

### 5. Idempotency

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

### 6. Secrets Management

**Never commit plain text secrets!**

- Use `ansible-vault` for passwords
- Mark vault-required values with comments: `# ansible-vault encrypt_string`
- Files with secrets: `group_vars/storage_nodes.yml`, `host_vars/*.yml`

**Example:**
```yaml
hacluster_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...encrypted...
```

## Project Structure

```
.
├── inventory.yml           # Inventory file (hostnames, groups)
├── site.yml               # Main playbook
├── group_vars/
│   ├── all.yml           # Global variables (VLANs, VIPs, cluster name)
│   └── storage_nodes.yml # Storage-specific (ZFS, iSCSI, STONITH, NFS, SMB)
├── host_vars/
│   ├── storage-a.yml     # Node-specific (IPs, disks, IQNs)
│   ├── storage-b.yml
│   └── quorum.yml
├── roles/
│   ├── common/           # Base OS setup, packages, repos
│   ├── hardening/        # Security (nftables, SSH, sysctl)
│   ├── zfs/              # ZFS module, pool creation, scrub automation
│   ├── iscsi-target/     # LIO target setup (backend replication)
│   ├── iscsi-initiator/  # open-iscsi initiator (backend replication)
│   ├── pacemaker/        # Cluster setup, STONITH, resource configuration
│   ├── services/         # NFS, SMB, iSCSI client-facing target
│   ├── monitoring/       # node_exporter, ha_cluster_exporter, custom exporters
│   └── cockpit/          # 45Drives Houston + Cockpit
└── docs/
    ├── cluster-monitoring.md    # HA cluster monitoring guide
    ├── stonith-smart-plugs.md   # Smart plug fencing guide
    ├── stonith-migration.md     # Migration guide for STONITH config
    ├── ntfy-integration.md      # Prometheus + NTFY alerting
    └── prometheus-alerts.yml    # Example alert rules
```

## Common Tasks

### Adding a New Service

1. Create role in `roles/<service-name>/`
2. Add tasks, templates, handlers
3. Include role in `site.yml`
4. **Check firewall rules** (nftables.conf.j2)
5. Add configuration variables to appropriate group_vars
6. Document in README.md
7. Test deployment

### Adding Monitoring for a Component

1. Create metrics exporter script or install package
2. Deploy to appropriate nodes
3. **Add firewall rule for metrics port**
4. Add scrape config to Prometheus
5. Create alert rules in `docs/prometheus-alerts.yml`
6. Document in `docs/cluster-monitoring.md` or relevant doc
7. Test metrics collection and alerts

### Modifying STONITH Configuration

1. Update `group_vars/storage_nodes.yml` (stonith_nodes dict)
2. Run playbook: `ansible-playbook -i inventory.yml site.yml --tags pacemaker`
3. Regenerate config: `ssh storage-a 'sudo bash /root/configure-stonith.sh'`
4. Verify: `pcs stonith status`
5. Test (careful!): `pcs stonith fence storage-b` (powers off node!)

## Known Issues / Gotchas

### 1. iSCSI Disk Paths

- iSCSI disk paths (`/dev/disk/by-path/`) may vary between runs
- Always verify paths in `create-pool.sh` before running
- Template generates script but requires manual verification

### 2. STONITH Testing

- STONITH tests are destructive (power off nodes)
- Always test during maintenance windows
- Never test without understanding the impact
- `pcs stonith fence <node>` will POWER OFF the node!

### 3. ZFS Pool Import

- Pool must be exported before Pacemaker can manage it
- `zpool export san-pool` required after manual pool creation
- Pacemaker will refuse to start resources if pool is already imported

### 4. Firewall Port Requirements

- **Management VLAN (10.20.20.0/24):**
  - SSH: 22
  - Corosync: 5405 (UDP)
  - pcsd: 2224
  - Cockpit: 9090
  - node_exporter: 9100
  - ha_cluster_exporter: 9664

- **Storage VLAN (10.10.10.0/24):**
  - iSCSI (backend): 3260
  - Corosync ring1: 5405 (UDP)

- **Client VLAN (10.30.30.0/24):**
  - NFS: 2049, 111
  - SMB: 445
  - iSCSI (client): 3260

## Testing Procedures

### Full Deployment Test

```bash
# 1. Deploy configuration
ansible-playbook -i inventory.yml site.yml

# 2. Verify cluster formation
ssh storage-a 'pcs status'

# 3. Create and export pool
ssh storage-a 'sudo bash /root/create-pool.sh'
ssh storage-a 'sudo zpool export san-pool'

# 4. Configure STONITH
ssh storage-a 'sudo bash /root/configure-stonith.sh'

# 5. Configure Pacemaker resources
ssh storage-a 'sudo bash /root/configure-pacemaker-resources.sh'

# 6. Verify all resources running
ssh storage-a 'pcs status'
```

### Monitoring Test

```bash
# 1. Check exporters are running
ssh storage-a 'systemctl status prometheus-node-exporter'
ssh storage-a 'systemctl status prometheus-hacluster-exporter'
ssh storage-a 'systemctl status zfs-scrub-exporter.timer'

# 2. Test metrics endpoints
curl http://10.20.20.1:9100/metrics | head
curl http://10.20.20.1:9664/metrics | head
curl http://10.20.20.1:9100/metrics | grep zfs_scrub

# 3. Configure Prometheus scrape targets
# 4. Verify metrics in Prometheus UI
# 5. Test alert rules
```

### Failover Test

```bash
# 1. Check current resource location
pcs status

# 2. Planned failover
pcs resource move san-resources storage-b
# Wait for failover to complete
pcs resource clear san-resources

# 3. Verify resources on new node
pcs status
zpool status san-pool  # On storage-b
```

## References

- [Pacemaker Documentation](https://clusterlabs.org/pacemaker/doc/)
- [Corosync Documentation](https://clusterlabs.org/corosync/doc/)
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)
- [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/)
- [NTFY Documentation](https://docs.ntfy.sh/)

## Change Log

- **2025-02-16**: Added mixed STONITH configuration support (per-node methods)
- **2025-02-16**: Added comprehensive monitoring (ZFS scrubs, cluster health)
- **2025-02-16**: Added ESPHome smart plug support for STONITH
- **Initial**: Base HA ZFS-over-iSCSI SAN playbook created
