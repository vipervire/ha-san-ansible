# Watchdog Support

The hardening role includes optional support for a system watchdog — a hardware or software
mechanism that forces a node reboot if the operating system stops responding. In a Pacemaker
cluster this is a form of **self-fencing**: if a node's kernel hangs or panics, the watchdog
expires and resets the node, freeing its resources for the surviving peer to take over.

The watchdog daemon is disabled by default (`watchdog_enabled: false`) to avoid accidental
reboots on misconfigured systems. Enable it only after verifying your chosen watchdog module
is stable in your environment.

---

## How It Works

```
  ┌──────────────────────────────────────────────────────┐
  │  OS (running normally)                               │
  │                                                      │
  │   watchdog daemon                                    │
  │      │  pets /dev/watchdog every interval seconds    │
  │      └──────────────────────────────────────────┐   │
  │                                                  ▼   │
  │   kernel watchdog driver (/dev/watchdog)             │
  │      • resets hardware countdown on each pet         │
  │      • if countdown reaches 0 → force reboot         │
  └──────────────────────────────────────────────────────┘

  If the OS hangs:
    - watchdog daemon stops running
    - /dev/watchdog stops receiving pets
    - countdown expires → node reboots
    - Pacemaker on surviving node imports ZFS pool and resumes services
```

The watchdog daemon (`watchdog` package, `/usr/sbin/watchdog`) also monitors:
- System load average (`max-load-1`)
- Free memory (`min-memory`)

If any check fails, the daemon deliberately stops petting the device, triggering a reboot.

---

## Configuration

Global defaults are set in `group_vars/all.yml`. Any variable can be overridden
per-node in `host_vars/<node>.yml` — useful when your two storage nodes have
different hardware (e.g. storage-a has an Intel TCO watchdog, storage-b only has softdog).

### Minimal — software watchdog (recommended for testing)

```yaml
# group_vars/all.yml (applies to all nodes)
watchdog_enabled: true
watchdog_module: softdog
```

### Hardware watchdog — Intel server boards

```yaml
watchdog_enabled: true
watchdog_module: iTCO_wdt
watchdog_device: /dev/watchdog
watchdog_timeout: 60
watchdog_interval: 10
```

### Hardware watchdog — IPMI/BMC

```yaml
watchdog_enabled: true
watchdog_module: ipmi_watchdog
watchdog_device: /dev/watchdog
watchdog_timeout: 60
watchdog_interval: 10
```

### Per-node configuration (mixed hardware)

When your nodes have different hardware, set the global enable in `group_vars/all.yml`
and override the module (and any other settings) per-node in `host_vars/`:

```yaml
# group_vars/all.yml — enable on all storage nodes
watchdog_enabled: true
watchdog_timeout: 60
watchdog_interval: 10
```

```yaml
# host_vars/storage-a.yml — enterprise server with Intel TCO hardware watchdog
watchdog_module: iTCO_wdt
```

```yaml
# host_vars/storage-b.yml — consumer server, fall back to software watchdog
watchdog_module: softdog
watchdog_module_options:
  soft_margin: "60"
```

You can also enable the watchdog on only one node by overriding `watchdog_enabled`:

```yaml
# group_vars/all.yml
watchdog_enabled: false          # off by default

# host_vars/storage-a.yml
watchdog_enabled: true           # override: enable only on storage-a
watchdog_module: iTCO_wdt
```

All `watchdog_*` variables follow standard Ansible variable precedence:
`host_vars` overrides `group_vars` overrides `defaults/main.yml`.

### Full configuration reference

| Variable | Default | Description |
|---|---|---|
| `watchdog_enabled` | `false` | Master enable/disable toggle |
| `watchdog_module` | (none) | Kernel module to load (`softdog`, `iTCO_wdt`, `ipmi_watchdog`, …) |
| `watchdog_module_options` | `{}` | Extra modprobe key/value options (see below) |
| `watchdog_device` | `/dev/watchdog` | Watchdog device path |
| `watchdog_timeout` | `60` | Seconds before reboot if daemon stops petting |
| `watchdog_interval` | `10` | Seconds between keep-alive pats |
| `watchdog_max_load` | `24.0` | 1-minute load average ceiling; breach triggers reboot |
| `watchdog_min_memory` | `1` | Minimum free memory in pages (1 page ≈ 4 KB) |

---

## Kernel Modules

### `softdog` — software watchdog (default)

Available on every x86 system without special hardware. Uses a kernel timer instead of a
dedicated hardware circuit. Sufficient for most HA deployments.

```yaml
watchdog_module: softdog
```

Useful `softdog` module options (set via `watchdog_module_options`):

| Option | Default | Description |
|---|---|---|
| `soft_margin` | `60` | Watchdog timeout in seconds (match `watchdog_timeout`) |
| `soft_noboot` | `0` | Set to `1` to log instead of reboot (testing only) |
| `nowayout` | `0` | Set to `1` to prevent daemon from closing device cleanly |

Example — extend timeout and prevent clean close:

```yaml
watchdog_module: softdog
watchdog_module_options:
  soft_margin: "120"
  nowayout: "1"
watchdog_timeout: 120
watchdog_interval: 30
```

> **Warning:** `nowayout=1` means `systemctl stop watchdog` will NOT prevent a reboot.
> The only way to stop the countdown is to reboot. Use `nowayout=0` during initial setup
> and testing.

### `iTCO_wdt` — Intel TCO hardware watchdog

Present on most Intel server chipsets (ICH/PCH series). Backed by dedicated hardware
circuits on the motherboard — survives a kernel panic that would stop softdog.

```yaml
watchdog_module: iTCO_wdt
```

Common options:

| Option | Default | Description |
|---|---|---|
| `heartbeat` | `-1` (auto) | Watchdog timeout in seconds |
| `nowayout` | `0` | Prevent clean close |

Verify the module is available:

```bash
modinfo iTCO_wdt
```

If your system uses SMBus access instead of I/O port (some newer chipsets):

```yaml
watchdog_module_options:
  turn_SMI_watchdog_clear_off: "1"
```

### `ipmi_watchdog` — IPMI BMC watchdog

Uses the server's BMC (Baseboard Management Controller) to implement the watchdog. Requires
an IPMI-capable server and the `ipmi_si` driver. Survives complete OS crashes.

```yaml
watchdog_module: ipmi_watchdog
```

Common options:

| Option | Default | Description |
|---|---|---|
| `action` | `reset` | Action on timeout: `reset`, `power_off`, `power_cycle` |
| `timeout` | `60` | BMC timeout in seconds |
| `pretimeout` | `0` | Seconds before timeout to send IPMI pre-timeout event |
| `nowayout` | `0` | Prevent clean close |

### `sp5100_tco` — AMD server board watchdog

For AMD EPYC / Ryzen-based servers with FCH/SP5100 chipset:

```yaml
watchdog_module: sp5100_tco
```

---

## Relationship to STONITH

The watchdog and STONITH are complementary, not alternatives:

| Mechanism | Who initiates | When used |
|---|---|---|
| STONITH (ipmi, kasa, etc.) | Surviving peer | Peer declares node dead (Corosync timeout) |
| Watchdog | Node itself | Node's own OS hangs or kernel panics |

A node that crashes hard (kernel panic, memory corruption) may never send Corosync
messages that trigger STONITH. The watchdog catches this case and forces a reset,
allowing the survivor to eventually import the ZFS pool after the normal STONITH
timeout expires.

**Defense-in-depth:** With both STONITH and watchdog enabled, a node can be fenced
externally (STONITH) AND reset itself (watchdog) — whichever fires first wins.

---

## Pacemaker Awareness

Pacemaker supports a `have-watchdog` cluster property. When set to `true` and SBD is
configured, Pacemaker coordinates with the SBD daemon (which uses the watchdog device)
for disk-based fencing. This playbook does **not** configure SBD — it uses direct
STONITH agents instead.

If you later add SBD to the cluster:

```bash
pcs property set have-watchdog=true
```

The watchdog device configured by this role (`/dev/watchdog`) can be reused by SBD.
See the Pacemaker documentation for full SBD integration details.

---

## Verification

After enabling and running the playbook:

```bash
# Check module is loaded
lsmod | grep -E 'softdog|iTCO|ipmi_watchdog'

# Check device exists
ls -la /dev/watchdog*

# Check daemon is running
systemctl status watchdog

# Check daemon log
journalctl -u watchdog -n 50

# Check module persistence
cat /etc/modules-load.d/watchdog.conf

# Check module options (if configured)
cat /etc/modprobe.d/<module>.conf
```

### Testing the watchdog (with caution)

> **WARNING:** The following test causes an immediate kernel panic and reboot.
> Only run on a node that is in Pacemaker standby and during a maintenance window.

```bash
# Put node in standby first (move resources to peer)
pcs node standby storage-b

# Trigger kernel panic to test watchdog reboot
echo c > /proc/sysrq-trigger
# Node should reboot within watchdog_timeout seconds

# After reboot, remove standby
pcs node unstandby storage-b
```

For a non-destructive smoke test, check that the watchdog device is being petted:

```bash
# This shows the watchdog is open and being serviced (returns immediately)
wdctl /dev/watchdog
```

---

## Troubleshooting

### `modprobe: FATAL: Module softdog not found`

The kernel does not include the softdog module. Check:

```bash
# Check kernel config
grep CONFIG_SOFT_WATCHDOG /boot/config-$(uname -r)
# Should show: CONFIG_SOFT_WATCHDOG=m

# Try loading with verbose output
modprobe -v softdog
```

On Rocky Linux 9, ensure the matching `kernel-modules-extra` package is installed:

```bash
dnf install kernel-modules-extra-$(uname -r)
```

### `watchdog.service` fails to start

Check if another process already holds `/dev/watchdog`:

```bash
fuser /dev/watchdog
lsof /dev/watchdog
```

Pacemaker or SBD may already be using the device. You cannot have two consumers of
`/dev/watchdog` simultaneously.

### Daemon starts but node reboots unexpectedly

1. Lower `watchdog_max_load` — your normal load may be exceeding the threshold
2. Increase `watchdog_timeout` and `watchdog_interval` to reduce sensitivity
3. Check available memory — `watchdog_min_memory: 1` is nearly zero; increase if nodes run low on memory
4. Review daemon logs: `journalctl -u watchdog`

### Cannot stop watchdog cleanly (nowayout=1)

If `nowayout=1` was set, the device cannot be closed without a reboot. To recover
without rebooting, unload the module (only possible if nowayout was compiled as a
module parameter, not built-in):

```bash
rmmod softdog   # will fail if nowayout=1 and device is open
```

In this state, the only safe recovery is a planned reboot or letting the timeout expire.
