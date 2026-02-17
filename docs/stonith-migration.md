# Migrating to Per-Node STONITH Configuration

If you previously configured STONITH using the old `stonith_method` global variable, this guide will help you migrate to the new per-node structure that supports mixed fencing methods.

## What Changed?

**Old approach (pre-v2.0):** All nodes used the same STONITH method (either all IPMI or all smart plugs)

**New approach (v2.0+):** Each node can use a different STONITH method, configured individually

## Migration Examples

### Example 1: IPMI-only Setup

**Old Structure:**
```yaml
stonith_method: "ipmi"
stonith_ipmi:
  storage-a:
    ip: "10.20.20.101"
    user: "admin"
    password: "secret"
  storage-b:
    ip: "10.20.20.102"
    user: "admin"
    password: "secret"
```

**New Structure:**
```yaml
stonith_nodes:
  storage-a:
    method: "ipmi"
    ip: "10.20.20.101"
    user: "admin"
    password: "secret"
  storage-b:
    method: "ipmi"
    ip: "10.20.20.102"
    user: "admin"
    password: "secret"
```

### Example 2: Smart Plug Setup (Kasa)

**Old Structure:**
```yaml
stonith_method: "smart_plug"
stonith_smart_plug_type: "kasa"
stonith_smart_plug:
  storage-a:
    ip: "10.20.20.201"
  storage-b:
    ip: "10.20.20.202"
```

**New Structure:**
```yaml
stonith_nodes:
  storage-a:
    method: "kasa"
    ip: "10.20.20.201"
  storage-b:
    method: "kasa"
    ip: "10.20.20.202"
```

### Example 3: Smart Plug Setup (ESPHome)

**Old Structure:**
```yaml
stonith_method: "smart_plug"
stonith_smart_plug_type: "esphome"
stonith_smart_plug:
  storage-a:
    ip: "10.20.20.201"
    switch_name: "relay"
    user: "admin"
    password: "secret"
  storage-b:
    ip: "10.20.20.202"
    switch_name: "relay"
    user: "admin"
    password: "secret"
```

**New Structure:**
```yaml
stonith_nodes:
  storage-a:
    method: "esphome"
    ip: "10.20.20.201"
    switch_name: "relay"
    user: "admin"
    password: "secret"
  storage-b:
    method: "esphome"
    ip: "10.20.20.202"
    switch_name: "relay"
    user: "admin"
    password: "secret"
```

## Step-by-Step Migration

### 1. Backup Your Configuration

```bash
cp group_vars/storage_nodes.yml group_vars/storage_nodes.yml.backup
```

### 2. Update group_vars/storage_nodes.yml

Find the STONITH section (starts around line 128) and replace it according to the examples above.

**Key changes:**
- Remove `stonith_method`, `stonith_ipmi`, `stonith_smart_plug`, and `stonith_smart_plug_type` variables
- Add new `stonith_nodes` dict
- For IPMI: use `method: "ipmi"`
- For Kasa: use `method: "kasa"`
- For Tasmota: use `method: "tasmota"`
- For ESPHome: use `method: "esphome"`
- For generic HTTP: use `method: "http"`

### 3. Run Ansible Playbook

```bash
ansible-playbook -i inventory.yml site.yml --tags pacemaker
```

This will:
- Deploy the updated STONITH configuration script
- Install necessary fence agent dependencies (e.g., python3-kasa if using Kasa)

### 4. Regenerate STONITH Resources

SSH to your preferred node (usually storage-a) and run the configuration script:

```bash
ssh storage-a
sudo bash /root/configure-stonith.sh
```

### 5. Verify Configuration

```bash
# Check STONITH resources are created
pcs stonith status

# Expected output (example for mixed setup):
#  * fence-storage-a (stonith:fence_ipmilan): Started storage-b
#  * fence-storage-b (stonith:fence_kasa): Started storage-a

# Check detailed configuration
pcs stonith show fence-storage-a
pcs stonith show fence-storage-b

# Test status queries (non-destructive)
pcs stonith status fence-storage-a
pcs stonith status fence-storage-b
```

## Now You Can Mix Methods!

The main benefit of the new structure is **mixed fencing**. You can now configure different methods per node:

```yaml
stonith_nodes:
  storage-a:
    method: "ipmi"        # Server with BMC
    ip: "10.20.20.101"
    user: "admin"
    password: "secret"

  storage-b:
    method: "kasa"        # Server with smart plug
    ip: "10.20.20.202"
```

This is useful when:
- One server has IPMI, the other doesn't
- Migrating from smart plugs to IPMI (or vice versa)
- Budget allows enterprise hardware for one node only
- Testing different fencing technologies

## Troubleshooting

### Issue: "stonith_nodes is undefined" error

**Cause:** You forgot to add the `stonith_nodes` variable

**Fix:** Add the `stonith_nodes` dict to `group_vars/storage_nodes.yml` as shown in the examples above

### Issue: STONITH resource fails to create

**Cause 1:** Wrong method name

**Fix:** Valid method names are: `ipmi`, `kasa`, `tasmota`, `esphome`, `http`

**Cause 2:** Missing required parameters

**Fix:** Check required parameters for your method:
- IPMI: requires `ip`, `user`, `password`
- Kasa: requires `ip`
- ESPHome: requires `ip`, `switch_name`
- Tasmota: requires `ip`
- HTTP: requires `ip`, `user`, `password`, `power_off_url`, `power_on_url`, `status_url`

### Issue: python3-kasa not installed

**Cause:** Using Kasa but package wasn't installed

**Fix:**
```bash
# Manual install
ssh storage-a
sudo apt-get install python3-kasa

# Or re-run playbook
ansible-playbook -i inventory.yml site.yml --tags pacemaker
```

### Issue: Old STONITH resources still present

**Cause:** Old fence agents weren't removed when regenerating config

**Fix:**
```bash
# List all STONITH resources
pcs stonith status

# Remove old ones manually
pcs stonith delete fence-storage-a
pcs stonith delete fence-storage-b

# Re-run configuration script
bash /root/configure-stonith.sh
```

## Rollback

If you encounter issues and need to revert:

```bash
# Restore backup
cp group_vars/storage_nodes.yml.backup group_vars/storage_nodes.yml

# Revert code changes (if you modified the playbook)
git checkout HEAD -- roles/pacemaker/templates/configure-stonith.sh.j2
git checkout HEAD -- roles/pacemaker/tasks/main.yml

# Re-run playbook
ansible-playbook -i inventory.yml site.yml --tags pacemaker

# Regenerate STONITH configuration
ssh storage-a
sudo bash /root/configure-stonith.sh
```

## Benefits of New Structure

1. **Flexibility**: Mix and match fencing methods per node
2. **Clarity**: All parameters in one place per node
3. **Simplicity**: One dict instead of multiple conditional variables
4. **Extensibility**: Easy to add new fencing methods in future
5. **Self-documenting**: `method` field makes configuration intent obvious

## Questions?

- **Can I still use the old format?** No, the new template requires `stonith_nodes`
- **Do I need to reconfigure my cluster?** Yes, run `/root/configure-stonith.sh` after migrating
- **Will this break my existing cluster?** Not immediately - old STONITH resources continue working until you regenerate
- **Can I migrate one node at a time?** No - update all nodes in `stonith_nodes` dict at once
- **What if I have three nodes?** Add all three to `stonith_nodes`, each with their own method

## References

- Smart plug setup guide: `docs/stonith-smart-plugs.md`
- Main README: `README.md`
- Pacemaker STONITH documentation: https://clusterlabs.org/pacemaker/doc/
