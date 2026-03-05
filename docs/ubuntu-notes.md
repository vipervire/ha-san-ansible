# Ubuntu 22.04/24.04 and AlmaLinux 9 — Notes and Differences

This document covers OS-specific behaviour, limitations, and workarounds when deploying the HA SAN playbook on **Ubuntu 22.04 LTS**, **Ubuntu 24.04 LTS**, and **AlmaLinux 9**.

---

## AlmaLinux 9

AlmaLinux 9 is binary-compatible with Rocky Linux 9 and requires **no special handling**. It shares `ansible_os_family == "RedHat"`, so all existing `RedHat.yml` role files load automatically. Package names, repository URLs, service names, and PAM configuration are identical to Rocky Linux 9.

No new role files were created for AlmaLinux. Simply add your AlmaLinux nodes to `inventory.yml` like any other Rocky Linux node.

---

## Ubuntu 22.04 / 24.04

Ubuntu has `ansible_os_family == "Debian"`, so it loads Debian base variables first, then overlays Ubuntu-specific overrides from `vars/Ubuntu.yml` (and `vars/Ubuntu-22.yml` where needed). The sections below document where Ubuntu differs from Debian 12.

### Firewall — ufw replaced by nftables

Ubuntu ships `ufw` enabled by default. The playbook uses raw nftables with `flush ruleset`, which conflicts with ufw. The `hardening` role's `Ubuntu.yml` task file:

1. Installs nftables
2. Disables and **masks** ufw (preventing it from being re-enabled accidentally)
3. Stops firewalld if present

After deployment, nftables manages the firewall exclusively. Do not re-enable ufw on managed nodes.

**Verify:**
```bash
sudo systemctl status ufw          # Should be masked
sudo nft list ruleset | head -20   # Should show HA SAN ruleset
```

### ZFS — native kernel modules, no DKMS

Ubuntu ships ZFS kernel modules natively as part of `linux-modules-extra-*`. There is no need for DKMS or the `zfs-dkms` package. The `zfs/Ubuntu.yml` tasks:

1. Enable the `universe` repository
2. Install `linux-headers-$(uname -r)` and `zfsutils-linux` / `zfs-zed`
3. Skip `zfs-dkms` and `dpkg-dev` entirely

**Note:** If you see `zfs: command not found` after install, the universe repo may not be enabled or an apt cache refresh is needed. Re-run the playbook — the task is idempotent.

### Sanoid — Ubuntu 24.04+ only in apt

| Ubuntu version | Sanoid availability |
|----------------|-------------------|
| 24.04 LTS      | Available in universe repo — installed by apt |
| 22.04 LTS      | **Not in repos** — dependencies installed, manual install required |

**On Ubuntu 22.04**, if `sanoid_install: true`, the playbook installs all Perl dependencies but prints a debug message. To complete the install manually:

```bash
git clone https://github.com/jimsalterjrs/sanoid.git /opt/sanoid
ln -s /opt/sanoid/sanoid /usr/sbin/sanoid
ln -s /opt/sanoid/syncoid /usr/sbin/syncoid
mkdir -p /etc/sanoid
```

Then re-run the playbook to deploy the sanoid configuration (the timer deployment is handled by shared tasks once the binary exists).

### PAM faillock — Ubuntu 22.04 limitation

`pam-auth-update --enable faillock` silently fails on Ubuntu 22.04 because the `faillock` PAM profile is not available. On 22.04 the command is replaced with a no-op (`... || true`) via `vars/Ubuntu-22.yml`. The `faillock.conf` is still deployed to `/etc/security/faillock.conf`.

On **Ubuntu 24.04**, the faillock PAM profile is available and the standard command works correctly.

**Manual workaround for 22.04** (optional):
```bash
# Add to /etc/pam.d/common-auth manually:
auth required pam_faillock.so preauth
# auth [default=die] pam_faillock.so authfail
auth sufficient pam_faillock.so authsucc
```

### ha_cluster_exporter — not in Ubuntu repos

`prometheus-hacluster-exporter` is not packaged for Ubuntu. The monitoring role skips the apt install on Ubuntu and prints a notice. `ha_cluster_monitoring_enabled` can remain `true` — the exporter config and service tasks are also skipped when the package variable is empty.

**To install manually:**
1. Download the binary from [ha_cluster_exporter releases](https://github.com/ClusterLabs/ha_cluster_exporter/releases)
2. Place at `/usr/bin/ha_cluster_exporter`
3. Create `/etc/default/prometheus-hacluster-exporter` with the listen address
4. Create a systemd unit (see the Debian package as a reference)
5. Set `ha_cluster_monitoring_enabled: true` and re-run the monitoring play

### Cockpit — 45Drives plugins

45Drives Houston plugins (`cockpit-file-sharing`, `cockpit-identities`, `cockpit-navigator`) are installed on all supported OSes:

| OS | Repo URL | Notes |
|----|----------|-------|
| Debian 12 | `https://repo.45drives.com/debian bookworm main` | apt, signed-by keyring |
| Ubuntu 22.04/24.04 | `https://repo.45drives.com/enterprise/ubuntu jammy\|noble main` | apt, signed-by keyring |
| Rocky Linux 9 | `https://repo.45drives.com/rockylinux/el9` | dnf/yum repo, gpgcheck enabled |
| AlmaLinux 9 | `https://repo.45drives.com/rockylinux/el9` | binary-compatible with Rocky; uses the same repo |

Installation uses `ignore_errors: true` on all platforms in case a specific plugin is unavailable for the running release.

### python3-kasa — pip fallback on 22.04

`python3-kasa` is not available in Ubuntu 22.04 apt repos. The `pacemaker/Ubuntu.yml` task file:

1. Tries apt first (works on 24.04 if the package is present)
2. Falls back to `pip3 install python-kasa` if apt fails

Only triggered when at least one STONITH node uses `method: kasa`.

---

## Inventory example with mixed OS

```yaml
# inventory.yml
all:
  children:
    cluster:
      children:
        storage_nodes:
          hosts:
            storage-a:
              ansible_host: 10.20.20.1
              # Debian 12 node (default)
            storage-b:
              ansible_host: 10.20.20.2
              # Ubuntu 24.04 node
        quorum_node:
          hosts:
            quorum:
              ansible_host: 10.20.20.3
              # AlmaLinux 9 node
```

All three nodes work with the same `site.yml` playbook — OS-specific handling is automatic.

---

## Supported OS matrix

| OS | Version | ZFS | Sanoid | ha_cluster_exporter | 45Drives |
|----|---------|-----|--------|--------------------|----|
| Debian | 12 | DKMS | apt | apt | yes |
| Ubuntu | 22.04 | native | manual | manual | yes |
| Ubuntu | 24.04 | native | apt | manual | yes |
| Rocky Linux | 9 | ELRepo RPM | apt (EPEL) | manual | yes |
| AlmaLinux | 9 | ELRepo RPM | apt (EPEL) | manual | yes |
