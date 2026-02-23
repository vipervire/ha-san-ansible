# Playbook Review: HA ZFS-over-iSCSI SAN

## Overall Assessment

Well-architected playbook with clear separation of concerns, comprehensive variable layering (defaults -> group_vars -> host_vars), good use of Ansible patterns (run_once, delegate_to, handlers), and thorough documentation. The pre-flight validation, graceful shutdown service, and post-deploy guide are particularly thoughtful. Below are the issues found, organized by severity.

---

## Bugs (Will Cause Runtime Failures)

### 1. Missing handlers: `iscsi-target` and `iscsi-initiator` roles

Both roles use `notify:` but neither has a `handlers/main.yml` file:

- **`roles/iscsi-target/tasks/main.yml:33`** notifies `save lio config` — handler does not exist
- **`roles/iscsi-initiator/tasks/main.yml:14,21`** notifies `restart iscsid` — handler does not exist

Ansible will error at runtime when these handlers fire. Create handlers files for both roles:

```yaml
# roles/iscsi-target/handlers/main.yml
---
- name: save lio config
  ansible.builtin.command:
    cmd: targetcli saveconfig
  changed_when: true

# roles/iscsi-initiator/handlers/main.yml
---
- name: restart iscsid
  ansible.builtin.systemd:
    name: iscsid
    state: restarted
```

### 2. Broken `hosts.j2` template references nonexistent dict keys

**File:** `roles/common/templates/hosts.j2:7`

The template references `node.ring0_addr` and `node.ring1_addr`, but `corosync_nodes` only contains `name` and `nodeid`. This will fail with a Jinja2 `UndefinedError` at template rendering time.

The `corosync.conf.j2` template handles this correctly using `hostvars[node.name].mgmt_ip`.

**Fix:**
```jinja2
# Cluster nodes — management VLAN
{% for node in corosync_nodes %}
{{ hostvars[node.name].mgmt_ip }}    {{ node.name }}
{% endfor %}

# Storage interconnect (private)
{% for node in corosync_nodes %}
{% if hostvars[node.name].storage_ip is defined %}
{{ hostvars[node.name].storage_ip }}    {{ node.name }}-stor
{% endif %}
{% endfor %}
```

---

## Security Issues

### 3. Credentials exposed in Ansible output — missing `no_log: true`

Several tasks handle sensitive data without suppressing output:

| Task | File | Risk |
|------|------|------|
| `pcs host auth ... -p {{ hacluster_password }}` | `roles/pacemaker/tasks/main.yml:51-61` | Password in CLI args visible in `/proc/*/cmdline` and Ansible logs |
| Deploy `iscsid.conf` with CHAP passwords | `roles/iscsi-initiator/tasks/main.yml:16-21` | Template content logged on `-vvv` |
| Generate targetcli setup script with CHAP | `roles/iscsi-target/tasks/main.yml:17-20` | Template content logged on `-vvv` |
| Generate STONITH script with IPMI passwords | `roles/pacemaker/tasks/main.yml:166-171` | Template content logged on `-vvv` |

**Fix:** Add `no_log: true` to all tasks that handle credentials.

### 4. STONITH passwords visible in process listings

`roles/pacemaker/templates/configure-stonith.sh.j2` renders passwords into `/root/configure-stonith.sh`. When executed, `pcs stonith create ... passwd=<cleartext>` is visible in process listings. File mode 0700 is appropriate, but the template deployment task should also use `no_log: true`.

---

## Correctness Issues

### 5. Shadow Copy format only matches hourly snapshots

**File:** `roles/services/templates/smb.conf.j2:45`

```ini
shadow:format = autosnap_%Y-%m-%d_%H:%M:%S_hourly
```

Sanoid creates snapshots with `_hourly`, `_daily`, `_monthly`, and `_yearly` suffixes. Only hourly snapshots will appear as Windows Previous Versions.

**Fix:** Use `shadow:snapprefix` with a regex to match all Sanoid snapshot types:
```ini
vfs objects = shadow_copy2
shadow:snapdir = .zfs/snapshot
shadow:sort = desc
shadow:snapprefix = ^autosnap_
shadow:format = %%Y-%%m-%%d_%%H:%%M:%%S_
```

### 6. VIP resources don't specify network interface

**File:** `roles/pacemaker/templates/configure-resources.sh.j2`

IPaddr2 resources are created without a `nic=` parameter. With three VLANs, auto-detection may pick the wrong interface. Client VIPs (NFS/SMB/iSCSI) should bind to the client VLAN interface, and the Cockpit VIP to the management interface.

**Fix:**
```bash
pcs resource create vip-nfs ocf:heartbeat:IPaddr2 \
  ip="{{ vip_nfs }}" cidr_netmask={{ vip_cidr }} \
  nic="{{ net_client_parent }}.{{ vlans.client.id }}" \
  ...
```

### 7. Operator precedence issue in Syncoid/ZED `when` conditions

**File:** `roles/zfs/tasks/main.yml:170,178,187`

```yaml
when: syncoid_enabled | default(false) and sanoid_install | default(false)
```

Due to Jinja2 operator precedence, `and` binds tighter than `|`, so this evaluates as:
`syncoid_enabled | default(false and sanoid_install) | default(false)`.

**Fix:** Use list form:
```yaml
when:
  - syncoid_enabled | default(false)
  - sanoid_install | default(false)
```

---

## Reliability Improvements

### 8. Corosync restart handler is risky on a running cluster

**File:** `roles/pacemaker/handlers/main.yml`

A hard restart of corosync on a running cluster triggers membership recalculation and can cause brief resource disruption. Consider:

- Using `corosync-cfgtool -R` (reload) for supported config changes
- Implementing serial execution with a wait-for-rejoin before proceeding to the next node

### 9. `pacemaker_preferred_node` as hard-coded `delegate_to` target

Multiple tasks in `roles/pacemaker/tasks/main.yml` delegate to `pacemaker_preferred_node`. If that node is unreachable, the entire cluster play fails with no fallback. Consider dynamically selecting an available node or documenting this requirement prominently.

### 10. Missing validation for `corosync.conf` before restart

The corosync.conf template triggers a restart without any syntax validation. A broken config could take down the cluster ring.

---

## Minor / Style

### 11. `ignore_errors: true` vs `failed_when: false`

**File:** `roles/common/tasks/main.yml:43`

`ignore_errors: true` still marks the task as failed (just not fatally), producing red output and `...ignoring` messages. `failed_when: false` treats it as success and produces cleaner output.

### 12. `reload sysctl` handler defined in two roles

Both `hardening` and `networking` define `reload sysctl`. Since both are in the same play, the duplicate works but could cause confusion. Consider using unique names (`reload hardening sysctl`, `reload network sysctl`) or extracting shared handlers.

### 13. No `ansible.cfg`

The project lacks an `ansible.cfg`. Recommended settings for this project:

```ini
[defaults]
inventory = inventory.yml
retry_files_enabled = false
stdout_callback = yaml

[ssh_connection]
pipelining = true
```

SSH pipelining in particular can significantly speed up execution.

### 14. Pre-flight validation doesn't catch empty client CHAP password

`iscsi_client_chap_password` defaults to `""` in `roles/services/defaults/main.yml`. If `iscsi_client_chap_enabled: true` is set without changing the password, the empty password passes the pre-flight check since it doesn't contain `CHANGEME`.

---

## What's Done Well

- Pre-flight credential validation catching common deployment mistakes
- Pre-run degraded cluster health check with bypass flag (`-e skip_cluster_check=true`)
- Graceful standby-before-shutdown systemd service for planned resource migration
- Post-deploy guided wizard script (`post-deploy.sh`)
- Proper use of `run_once` + `delegate_to` for cluster-wide operations
- Comprehensive monitoring with atomic textfile writes (tmp + mv pattern)
- ZFS scrub wrapper with degraded-pool and min-interval guards
- Auto-discovery of iSCSI LUN paths by LUN number in `create-pool.sh`
- Clean variable layering with role defaults, group_vars directory split, and host_vars
- Reusable `deploy_timer_exporter.yml` include for the monitoring role
- ZED -> ntfy push notification handler for immediate pool event alerting
- Thorough CLAUDE.md with architecture overview, operational runbooks, and gotchas
