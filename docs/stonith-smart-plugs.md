# STONITH Fencing with Smart Plugs

This guide covers setting up smart plug-based fencing for your HA cluster when IPMI is not available.

## ⚠️ Important Considerations

**Smart plugs are a budget alternative to IPMI but have limitations:**
- ✅ Better than no STONITH (critical for data integrity)
- ✅ Works for home labs and small deployments
- ✅ Much cheaper than IPMI-capable hardware ($15 vs $1000+)
- ⚠️ Dependent on network connectivity (same as IPMI over LAN)
- ⚠️ Consumer-grade reliability (not enterprise-rated)
- ⚠️ Requires manual fence agent installation in some cases

## Mixed STONITH Configurations

The playbook now supports **mixed fencing methods** where different nodes can use different STONITH mechanisms. This is useful when:
- Some servers have IPMI/BMC, others don't
- Migrating from one fencing method to another
- Budget allows IPMI for one node, smart plug for the other
- Testing different fencing technologies

### Configuration Example

```yaml
# In group_vars/storage_nodes.yml
stonith_nodes:
  storage-a:
    method: "ipmi"           # Enterprise server with BMC
    ip: "10.20.20.101"
    user: "bmcadmin"
    password: "CHANGEME"

  storage-b:
    method: "kasa"           # Consumer server with smart plug
    ip: "10.20.20.202"
```

The playbook will automatically:
- Install necessary fence agent dependencies (python3-kasa if any node uses Kasa)
- Generate correct `pcs stonith create` commands per node
- Configure firewall rules for all required protocols

**More examples:**
- Both IPMI: Set both nodes to `method: "ipmi"`
- Both smart plugs: Mix `kasa`, `esphome`, `tasmota` as needed
- Three nodes: Add third node with any method

See `docs/stonith-migration.md` for migrating from the old configuration format.

## Recommended Options

### Option 1: TP-Link Kasa HS105 (Easiest)
- **Local control** without cloud dependency
- **Reliable** and widely used in home automation
- **Cheap** (~$15-20 per plug)
- **Python library** for easy integration (`python3-kasa`)
- **No authentication** needed (network isolation provides security)
- **Best for:** Plug-and-play setup with minimal configuration

### Option 2: ESPHome-Flashed Plugs (Most Flexible)
- **Open source** firmware with active development
- **YAML configuration** (easier than Tasmota)
- **Excellent OTA updates** and debugging
- **Home Assistant integration** (optional)
- **Best for:** Users already in ESPHome ecosystem or wanting maximum control

### Option 3: Tasmota-Flashed Plugs (Most Mature)
- **Open source** firmware with large community
- **Feature-rich** web interface
- **No dependencies** on external services
- **MQTT support** (optional)
- **Best for:** Users wanting established open-source solution

### Hardware Requirements

**Minimum setup:**
- 2x TP-Link Kasa HS100/HS105/HS110 smart plugs
- Static IP addresses on management VLAN (10.20.20.0/24)
- Each storage node plugged into its own smart plug
- Plugs should be on UPS if possible (to survive brief power blips)

**Network topology:**
```
storage-a (10.20.20.1) ─── powered by ──→ Kasa plug A (10.20.20.201)
storage-b (10.20.20.2) ─── powered by ──→ Kasa plug B (10.20.20.202)
                                            ↑
                                            │
                                    Management VLAN switch
                                       (10.20.20.0/24)
```

**Critical:** Each node must be able to reach the OTHER node's plug, but not its own. Use firewall rules if needed.

## Installation Steps

### 1. Configure Smart Plugs

```bash
# Install Kasa CLI on your laptop/workstation
pip3 install python-kasa

# Discover plugs on your network
kasa discover

# Configure static IP for each plug
kasa --host 192.168.1.x --type plug --alias "storage-a-power" set-ip 10.20.20.201 255.255.255.0 10.20.20.1

kasa --host 192.168.1.y --type plug --alias "storage-b-power" set-ip 10.20.20.202 255.255.255.0 10.20.20.1

# Test control
kasa --host 10.20.20.201 --type plug on
kasa --host 10.20.20.201 --type plug state
kasa --host 10.20.20.201 --type plug off
```

### 2. Update Ansible Variables

**Edit `group_vars/storage_nodes.yml`:**

```yaml
# Option B: TP-Link Kasa smart plugs
stonith_method: "smart_plug"
stonith_smart_plug_type: "kasa"  # kasa, tasmota, or http
stonith_smart_plug:
  storage-a:
    ip: "10.20.20.201"   # Kasa plug controlling storage-a
  storage-b:
    ip: "10.20.20.202"   # Kasa plug controlling storage-b
```

Comment out the IPMI section if it was previously configured.

### 3. Install fence_kasa Agent

The `fence_kasa` agent is not in standard Debian packages yet. Install it manually on all cluster nodes:

```bash
# On storage-a, storage-b, and quorum node
ssh storage-a
sudo -i

# Install dependencies
apt-get install -y python3-kasa python3-pexpect

# Download fence_kasa from GitHub
wget -O /usr/sbin/fence_kasa \
  https://raw.githubusercontent.com/ClusterLabs/fence-agents/main/agents/kasa/fence_kasa.py

chmod +x /usr/sbin/fence_kasa

# Create symlink for Pacemaker
ln -sf /usr/sbin/fence_kasa /usr/lib/stonith/plugins/external/kasa

# Test the agent
fence_kasa --ip 10.20.20.201 --action status
# Should return: Status: ON

# Repeat on storage-b and quorum node
```

**Alternative: Ansible task to automate installation**

Add to `roles/pacemaker/tasks/main.yml` after the package installation:

```yaml
- name: Download fence_kasa agent
  ansible.builtin.get_url:
    url: https://raw.githubusercontent.com/ClusterLabs/fence-agents/main/agents/kasa/fence_kasa.py
    dest: /usr/sbin/fence_kasa
    mode: '0755'
    owner: root
    group: root
  when: stonith_method == 'smart_plug' and stonith_smart_plug_type == 'kasa'

- name: Create fence_kasa metadata link
  ansible.builtin.file:
    src: /usr/sbin/fence_kasa
    dest: /usr/lib/stonith/plugins/external/kasa
    state: link
  when: stonith_method == 'smart_plug' and stonith_smart_plug_type == 'kasa'
```

### 4. Run Ansible Playbook

```bash
# Deploy cluster configuration
ansible-playbook -i inventory.yml site.yml --tags pacemaker

# The playbook will generate /root/configure-stonith.sh on storage-a
ssh storage-a
sudo bash /root/configure-stonith.sh
```

### 5. Verify STONITH Configuration

```bash
# Check STONITH resources are created
pcs stonith status

# Expected output:
#  * fence-storage-a (stonith:fence_kasa): Started storage-b
#  * fence-storage-b (stonith:fence_kasa): Started storage-a

# Test status query (non-destructive)
pcs stonith show fence-storage-b
fence_kasa --ip 10.20.20.202 --action status

# View full cluster config
pcs config
```

### 6. Test Fencing (DESTRUCTIVE - BE CAREFUL!)

**⚠️ WARNING: This will POWER OFF the target node! Only test during maintenance window.**

```bash
# On storage-a, fence storage-b (storage-b will power off!)
pcs stonith fence storage-b

# Watch the plug turn off, wait 10 seconds, then manually power on:
kasa --host 10.20.20.202 --type plug on

# Verify in cluster logs
journalctl -u pacemaker -n 100 | grep -i fenc

# Expected log entries:
# - "Requesting Fencing of node storage-b"
# - "Operation stonith_admin status returned: 0 (OK)"
```

## Firewall Rules

The nftables rules already allow all management VLAN traffic, which includes smart plug communication:

```nftables
# In nftables.conf.j2 - already present
ip saddr {{ vlans.management.subnet }} tcp dport 22 accept   # SSH
ip saddr {{ vlans.management.subnet }} tcp dport 2224 accept # pcsd
```

Smart plugs use HTTP (port 9999 typically), which is outbound from cluster nodes, so no additional rules needed.

## Alternative: Open-Source Firmware Plugs

If you prefer open-source firmware, both Tasmota and ESPHome are excellent options.

### Option A: Tasmota-Flashed Plugs

**Hardware Setup:**
1. **Compatible plugs:** Sonoff S31, Gosund WP3, or any ESP8266/ESP32-based plug
2. **Flash Tasmota firmware** using [tuya-convert](https://github.com/ct-Open-Source/tuya-convert) or serial
3. **Configure Tasmota:**
   - Set static IP (10.20.20.201, 10.20.20.202)
   - Enable web interface
   - Optional: set password for authentication

**Ansible Configuration:**
```yaml
stonith_method: "smart_plug"
stonith_smart_plug_type: "tasmota"
stonith_smart_plug:
  storage-a:
    ip: "10.20.20.201"
    user: "admin"              # optional if password set
    password: "your-password"   # optional if password set
  storage-b:
    ip: "10.20.20.202"
    user: "admin"
    password: "your-password"
```

**Testing Tasmota Commands:**
```bash
# Power on
curl http://10.20.20.201/cm?cmnd=Power%20ON

# Power off
curl http://10.20.20.201/cm?cmnd=Power%20OFF

# Check status
curl http://10.20.20.201/cm?cmnd=Power
# Returns: {"POWER":"ON"}
```

### Option B: ESPHome-Flashed Plugs

**Why ESPHome?**
- Very popular in Home Assistant community
- YAML-based configuration (easier than Tasmota)
- Excellent OTA update support
- Native API with encryption (but we'll use HTTP for simplicity)
- Very active development

**Hardware Setup:**
1. **Compatible plugs:** Same as Tasmota (Sonoff S31, Gosund WP3, etc.)
2. **Flash ESPHome firmware** using web installer or ESPHome Dashboard
3. **ESPHome YAML configuration example:**

```yaml
# storage-a-power.yaml
esphome:
  name: storage-a-power
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: "YourSSID"
  password: "YourPassword"
  manual_ip:
    static_ip: 10.20.20.201
    gateway: 10.20.20.1
    subnet: 255.255.255.0

# Enable HTTP REST API (for STONITH)
web_server:
  port: 80
  auth:
    username: admin
    password: !secret web_password  # optional

# Enable logging
logger:

# Enable Home Assistant API (optional)
api:
  encryption:
    key: !secret api_key

ota:
  password: !secret ota_password

# Define the relay/switch
switch:
  - platform: gpio
    name: "Relay"
    id: relay
    pin: GPIO12
    restore_mode: ALWAYS_ON  # Critical: keep server powered on after reboot
```

**Ansible Configuration:**
```yaml
stonith_method: "smart_plug"
stonith_smart_plug_type: "esphome"
stonith_smart_plug:
  storage-a:
    ip: "10.20.20.201"
    switch_name: "relay"        # Must match ESPHome switch ID
    user: "admin"               # optional if web auth disabled
    password: "your-password"    # optional if web auth disabled
  storage-b:
    ip: "10.20.20.202"
    switch_name: "relay"
    user: "admin"
    password: "your-password"
```

**Testing ESPHome Commands:**
```bash
# Power on (POST request)
curl -X POST http://10.20.20.201/switch/relay/turn_on

# Power off (POST request)
curl -X POST http://10.20.20.201/switch/relay/turn_off

# Check status (GET request)
curl http://10.20.20.201/switch/relay
# Returns: {"id":"relay-relay","state":"ON","value":true}

# With authentication
curl -u admin:your-password -X POST http://10.20.20.201/switch/relay/turn_on
```

**Important ESPHome Notes:**
- Set `restore_mode: ALWAYS_ON` to prevent accidental power-off after plug reboot
- The HTTP REST API is automatically enabled when you add `web_server:` component
- Switch name in URL must match the `id:` in your ESPHome config
- Web authentication is optional but recommended for production

The playbook will automatically configure `fence_http` with the correct URLs for Tasmota or ESPHome.

## Troubleshooting

### STONITH Resource Fails to Start

```bash
# Check resource errors
pcs resource status fence-storage-b

# View detailed status
pcs resource debug-start fence-storage-b

# Common issues:
# 1. fence_kasa not installed: Install on ALL nodes
# 2. Python module missing: apt install python3-kasa python3-pexpect
# 3. Network unreachable: Verify plug IP and network connectivity
# 4. Plug not responding: Check plug is powered and on correct IP
```

### Test Connectivity from Cluster Nodes

```bash
# From storage-a, test reaching storage-b's plug
ssh storage-a
fence_kasa --ip 10.20.20.202 --action status

# From storage-b, test reaching storage-a's plug
ssh storage-b
fence_kasa --ip 10.20.20.201 --action status

# Both should return "Status: ON" if nodes are running
```

### Plug Not Responding

```bash
# Check plug is reachable
ping 10.20.20.201

# Check if plug is on correct VLAN
kasa --host 10.20.20.201 --type plug state

# Reset plug configuration (last resort)
# Hold button for 10+ seconds until LED blinks rapidly
```

### Fencing Takes Too Long

If fencing operations timeout, increase timeouts in STONITH configuration:

```bash
# Edit /root/configure-stonith.sh on storage-a
# Add longer timeouts:
pcs stonith create fence-storage-b fence_kasa \
  ip="10.20.20.202" \
  pcmk_host_list="storage-b" \
  pcmk_reboot_action="off" \
  pcmk_off_timeout="60s" \    # Increase from 30s
  pcmk_on_timeout="60s" \     # Increase from 30s
  op monitor interval=60s

# Re-run the script
bash /root/configure-stonith.sh
```

## Security Considerations

1. **Network isolation:** Smart plugs must ONLY be accessible from management VLAN
2. **No internet access:** Block smart plugs from reaching internet (prevents cloud firmware updates)
3. **Static IPs:** Always use static IPs, not DHCP
4. **UPS protection:** Put plugs on UPS to prevent accidental fencing during brief power blips
5. **Physical security:** Ensure plugs cannot be easily unplugged by unauthorized personnel

## Production Readiness

**Smart plugs are acceptable for production IF:**
- ✅ You understand the limitations and risks
- ✅ Network isolation is properly configured
- ✅ You have monitoring alerts for STONITH failures
- ✅ You've tested fencing multiple times
- ✅ You have documented recovery procedures

**Consider IPMI instead IF:**
- Your servers already have BMC (check for iLO, iDRAC, IPMI ports)
- You need enterprise-grade reliability
- Budget allows for proper server hardware

## Post-Fence Verification

### The Problem

When Pacemaker triggers a fence operation, it trusts the fence agent's return code. But the agent might report success even when the node isn't truly dead — for example, if the smart plug controls the wrong outlet, a UPS keeps the node running, or the device API reports success but the relay didn't actuate. If Pacemaker then starts resources on the surviving node while the "fenced" node is still running, both nodes may import the ZFS pool simultaneously → **split-brain → data corruption**.

### How Fence Verification Works

The playbook deploys a custom `fence_check` agent and uses **Pacemaker fencing levels** to add a mandatory post-fence verification step. Both the primary fence agent AND the verification agent must succeed before Pacemaker proceeds with failover:

```
Fencing level 1 for storage-b:
  1. fence-storage-b      (real agent: fence_kasa / fence_ipmilan / etc.)
  2. fence-verify-storage-b  (verification: fence_check)

Both must succeed → fencing complete → failover proceeds
Either fails → fencing BLOCKED → failover does NOT happen
```

The `fence_check` agent:
1. Waits 10 seconds for the node to fully power down
2. Queries the STONITH device for the target node's power state
3. Pings the target on management, storage, and client VLANs
4. Returns success only if **power is OFF** and **all pings fail**

### Configuration

Enabled by default (`stonith_fence_verify: true` in `roles/pacemaker/defaults/main.yml`). To disable:

```yaml
# group_vars/storage_nodes.yml
stonith_fence_verify: false
```

Re-run the playbook and then regenerate the STONITH config:
```bash
ansible-playbook -i inventory.yml site.yml --tags cluster
ssh storage-a 'sudo bash /root/configure-stonith.sh'
```

### Verifying the Topology

After running `configure-stonith.sh`, confirm the fencing levels are configured:

```bash
ssh storage-a 'pcs stonith level'
# Expected output:
#   Level 1 - storage-a: fence-storage-a,fence-verify-storage-a
#   Level 1 - storage-b: fence-storage-b,fence-verify-storage-b

ssh storage-a 'pcs stonith show fence-verify-storage-b'
```

### Manual Status Check (Non-Destructive)

```bash
# Check current power state and network reachability of peer
# (safe to run at any time — does NOT fence the node)
ssh storage-a 'fence_check -o status -n storage-b'

# Example output when both nodes are online (normal):
#   fence_check status for storage-b:
#     STONITH device (kasa @ 10.20.20.202): power=on
#     Management   10.20.20.2:  reachable
#     Storage      10.10.10.2:  reachable
#     Client       10.30.30.2:  reachable
#   Status: ON
```

### Troubleshooting a Blocked Failover

If `STONITHFenceVerifyFailed` fires and failover is blocked:

1. **Check why the node is still alive:**
   ```bash
   ssh storage-a 'fence_check -o status -n storage-b'
   # Look for which check is failing: power state, or which network
   ```

2. **Common causes:**
   - Smart plug controls the wrong outlet — verify the physically-connected outlet
   - UPS is keeping the node running — check UPS bypass status
   - IPMI command was accepted but BMC is unresponsive — check BMC management port
   - Network still reachable on storage/client VLANs after power cycle (e.g., network switch has the old ARP cached) — try waiting 30s and re-checking

3. **If the node is confirmed dead** (you've physically verified) and fencing is falsely blocked:
   ```bash
   # Clear the verification resource failure (allows Pacemaker to retry fencing)
   ssh storage-a 'pcs resource cleanup fence-verify-storage-b'
   ```

4. **Do NOT clear the failure** until you are certain the fenced node is not running with ZFS pool imported.

## Monitoring STONITH Status

The cluster monitoring setup includes a critical alert for STONITH status:

```yaml
# From docs/prometheus-alerts.yml
- alert: StonithDisabled
  expr: ha_cluster_pacemaker_stonith_enabled == 0
  for: 10m
  labels:
    severity: critical
    component: cluster
  annotations:
    summary: "STONITH is disabled in cluster"
```

This ensures you'll be notified via NTFY if STONITH becomes disabled for any reason.

## References

- [TP-Link Kasa Python Library](https://github.com/python-kasa/python-kasa)
- [fence_kasa Agent Source](https://github.com/ClusterLabs/fence-agents/tree/main/agents/kasa)
- [Tasmota Documentation](https://tasmota.github.io/docs/)
- [Pacemaker Fencing Guide](https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/2.0/html/Clusters_from_Scratch/ch07.html)
