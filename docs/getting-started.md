# Getting Started

This guide walks through a first deployment of the HA ZFS-over-iSCSI SAN from scratch. Follow the steps in order — each step builds on the previous one.

**Estimated time:** 2–4 hours for configuration, plus hardware setup and post-deploy manual steps.

---

## Prerequisites

### Hardware
- **3 servers:** 2 storage nodes + 1 quorum node
- **Supported OS:** Debian 12, Ubuntu 22.04/24.04, Rocky Linux 9, or AlmaLinux 9 (mix allowed)
- **Storage nodes:** at least 2 matching disks per node for ZFS mirroring; enterprise SSDs recommended
- **Network:** at minimum a management NIC on every node; storage nodes need a second NIC (40GbE recommended) for iSCSI replication
- **STONITH:** IPMI/BMC on each storage node, OR a smart plug (Kasa, Tasmota, ESPHome) per node that can power-cycle it

### Software
- Ansible 2.14+ on your controller workstation
- SSH key access to all 3 nodes
- `sudo` configured for your admin user on all nodes (or root access)

### Clone the repository

```bash
git clone <repository-url> ha-san-ansible
cd ha-san-ansible
pip install ansible
ansible-galaxy install -r requirements.yml   # if present
```

---

## Step 1: Plan Your Network

Before editing any files, document your network layout. You need addresses for every node on every VLAN.

| VLAN | Purpose | Recommended Subnet |
|------|---------|-------------------|
| Storage | iSCSI between storage nodes | 10.10.10.0/24 (40GbE) |
| Management | Corosync, SSH, IPMI, Cockpit | 10.20.20.0/24 (1GbE) |
| Client VLANs | NFS/SMB/iSCSI to clients | 10.30.30.0/24, 10.40.40.0/24, … |

For each VLAN, assign:
- A static IP for `storage-a`
- A static IP for `storage-b`
- A floating VIP (claimed by whichever node is active)

Write these down — you will enter them in Steps 3 and 7.

---

## Step 2: Configure the Inventory

Edit `inventory.yml` and set the `ansible_host` for each node to its management IP.

```yaml
all:
  children:
    cluster:
      hosts:
        storage-a:
          ansible_host: 10.20.20.1
        storage-b:
          ansible_host: 10.20.20.2
        quorum:
          ansible_host: 10.20.20.3
```

Verify connectivity:

```bash
ansible -i inventory.yml all -m ping
```

All three nodes should return `pong`.

---

## Step 3: Configure Global Variables

Edit `group_vars/all.yml`.

**Cluster identity:**

```yaml
cluster_name: san-cluster
hacluster_password: !vault |   # vault-encrypt this (see Step 9)
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

**Admin user:**

```yaml
admin_user: storageadmin
admin_ssh_pubkey: "ssh-ed25519 AAAA... your-key-here"
```

**VLANs** — update IDs, subnets, and MTUs to match your switch configuration:

```yaml
vlans:
  storage:
    id: 10
    subnet: "10.10.10.0/24"
    mtu: 9000
  management:
    id: 20
    subnet: "10.20.20.0/24"
    mtu: 1500

vip_cockpit: "10.20.20.10"
vip_mgmt_cidr: 24
```

**Client VLANs** — add one entry per VLAN serving storage to clients:

```yaml
client_vlans:
  - name: enduser
    id: 30
    subnet: "10.30.30.0/24"
    mtu: 1500
    vip: "10.30.30.10"
    vip_cidr: 24
    services: [nfs, smb]
  - name: hypervisor
    id: 40
    subnet: "10.40.40.0/24"
    mtu: 9000
    vip: "10.40.40.10"
    vip_cidr: 24
    services: [iscsi, ssh]
    iscsi_acls:
      - "iqn.2025-01.lab.home:proxmox-a"
      - "iqn.2025-01.lab.home:proxmox-b"
    iscsi_dataset: "iscsi/hypervisor"
```

See `docs/variables.md` for the full list of per-VLAN fields.

**Corosync** — leave the defaults unless you have specific latency requirements. Do not set `corosync_token` below 3000.

**SSH allowed users** — add any additional admin accounts:

```yaml
ssh_allowed_users:
  - storageadmin
```

---

## Step 4: Configure Network Interfaces

Edit `group_vars/storage_nodes/network.yml` and set the parent interface names to match your hardware:

```yaml
net_storage_parent: "ens3f0"   # NIC for storage VLAN (iSCSI replication)
net_client_parent: "ens3f1"    # NIC for client VLANs
net_mgmt_interface: "eno1"     # Management NIC
```

To find interface names on a node:

```bash
ip link show
```

If you use bonding or have a single NIC, the same interface can serve multiple roles — but dedicated NICs are strongly recommended for storage.

---

## Step 5: Configure iSCSI Backend Replication

Edit `group_vars/storage_nodes/iscsi.yml`.

Set the IQN prefix to match your domain and year:

```yaml
iscsi_iqn_prefix: "iqn.2025-01.yourdomain.example"
```

Set CHAP credentials (vault-encrypt these in Step 9):

```yaml
iscsi_chap_user: "iscsi-repl-user"
iscsi_chap_password: "CHANGEME-vault-this"

iscsi_mutual_chap_user: "iscsi-target-user"
iscsi_mutual_chap_password: "CHANGEME-vault-this"
```

> **Important:** `iscsi_mutual_chap_password` must differ from `iscsi_chap_password`. iSCSI rejects identical bidirectional credentials.

For single-VLAN iSCSI setups, also set the client ACL list:

```yaml
iscsi_client_acls:
  - "iqn.2025-01.yourdomain.example:proxmox-a"
  - "iqn.2025-01.yourdomain.example:proxmox-b"
```

---

## Step 6: Configure STONITH Fencing

Edit `group_vars/storage_nodes/cluster.yml`.

Each storage node must have a fencing entry. Mixed methods are supported:

```yaml
stonith_nodes:
  storage-a:
    method: "ipmi"
    ip: "10.20.20.101"     # BMC IP
    user: "bmcadmin"
    password: "CHANGEME-vault-this"   # vault-encrypt this

  storage-b:
    method: "kasa"
    ip: "10.20.20.202"     # Smart plug IP
    # Kasa local protocol needs no credentials
```

Supported methods: `ipmi`, `kasa`, `tasmota`, `esphome`, `http`. See `docs/stonith-smart-plugs.md` for detailed setup per method.

> **Fencing latency note:** IPMI typically takes 20–30s; Kasa 5–15s. The ZFS resource start timeout (150s) accommodates these. If your fence agent is slower, increase `pcmk_reboot_timeout` and the `zfs-pool` start timeout before deploying.

---

## Step 7: Configure Per-Node Variables

### storage-a — `host_vars/storage-a.yml`

```yaml
storage_ip: "10.10.10.1"
mgmt_ip: "10.20.20.1"

client_ips:
  enduser: "10.30.30.1"
  hypervisor: "10.40.40.1"

ssh_listen_addresses:
  - "{{ mgmt_ip }}"

iscsi_target_iqn: "{{ iscsi_iqn_prefix }}:storage-a"
iscsi_initiator_name: "{{ iscsi_iqn_prefix }}:initiator-a"
iscsi_peer_ip: "10.10.10.2"
iscsi_peer_iqn: "{{ iscsi_iqn_prefix }}:storage-b"

local_data_disks:
  - device: /dev/disk/by-id/ata-YOUR_DISK_ID_HERE
    label: "data-a-0"
  # ... add remaining disks
```

### storage-b — `host_vars/storage-b.yml`

Same structure as storage-a with its own IPs, IQN suffix (`storage-b`, `initiator-b`), and peer pointing back to storage-a.

### quorum — `host_vars/quorum.yml`

```yaml
mgmt_ip: "10.20.20.3"
ssh_listen_addresses:
  - "{{ mgmt_ip }}"
```

### Finding disk paths

Always use persistent `/dev/disk/by-id/` paths — never `/dev/sdX` (reorders on reboot):

```bash
ls -la /dev/disk/by-id/ | grep -v part | grep -v wwn
```

Use `ata-*`, `scsi-*`, or `nvme-*` identifiers. Example:

```
ata-WDC_WD100EFAX-68LHPN0_XXXXXXXX -> ../../sda
```

If you have NVMe drives for SLOG (ZIL — accelerates sync writes) or special vdev, uncomment and populate in host_vars:

```yaml
slog_disk:
  device: /dev/disk/by-id/nvme-PLACEHOLDER_NVME_A
  label: "slog-a"
```

---

## Step 8: Configure Services (Optional)

### NFS exports — `group_vars/storage_nodes/services.yml`

```yaml
nfs_exports:
  - path: "/san-pool/nfs"
    clients: "10.30.30.0/24"
    options: "rw,sync,no_subtree_check,root_squash,sec=sys"
```

### SMB shares — `group_vars/storage_nodes/services.yml`

```yaml
smb_workgroup: "HOMELAB"
smb_shares:
  - name: shared
    path: "/san-pool/smb"
    read_only: false
    valid_users: "@smbusers"
    shadow_copy: true
```

### ZFS datasets — `group_vars/storage_nodes/zfs.yml`

Add or remove datasets from `zfs_datasets`. The playbook creates these after the pool exists. If you have multiple iSCSI VLANs, add sub-datasets matching each VLAN's `iscsi_dataset` value:

```yaml
zfs_datasets:
  - name: "san-pool/iscsi/hypervisor"
    properties:
      primarycache: metadata
```

---

## Step 9: Vault All Secrets

Never commit plain-text passwords. Encrypt each credential with ansible-vault:

```bash
ansible-vault encrypt_string 'yourpassword' --name 'hacluster_password'
```

Paste the output into the variable file, replacing the `CHANGEME` placeholder.

**Variables that block deployment if not vaulted:**

| Variable | File |
|----------|------|
| `hacluster_password` | `group_vars/all.yml` |
| `iscsi_chap_password` | `group_vars/storage_nodes/iscsi.yml` |
| `iscsi_mutual_chap_password` | `group_vars/storage_nodes/iscsi.yml` |
| `stonith_nodes.<node>.password` | `group_vars/storage_nodes/cluster.yml` |
| `chap_password` (per-initiator ACLs) | `group_vars/all.yml` (`client_vlans`) |

The pre-flight play in `site.yml` checks all of these and aborts if any still contain `CHANGEME`.

To use vault with the playbook, either:
- Pass `--ask-vault-pass` at runtime, or
- Configure a vault password file: `ansible.cfg` → `vault_password_file = ~/.vault_pass`

---

## Step 10: Run the Playbook

Do a dry-run first to review what will change:

```bash
ansible-playbook -i inventory.yml site.yml --check --diff -e skip_cluster_check=true
```

On the first deployment, skip the cluster pre-check (the cluster doesn't exist yet):

```bash
ansible-playbook -i inventory.yml site.yml -e skip_cluster_check=true
```

You can limit to specific roles using tags:

```bash
ansible-playbook -i inventory.yml site.yml --tags storage    # ZFS + iSCSI only
ansible-playbook -i inventory.yml site.yml --tags cluster    # Pacemaker only
ansible-playbook -i inventory.yml site.yml --tags services   # NFS/SMB/iSCSI services
```

---

## Step 11: Post-Deploy Manual Steps

The playbook stops before irreversible cluster configuration. Complete these steps in order on the primary node (`storage-a`):

### 1. Verify iSCSI sessions

On both storage nodes:

```bash
iscsiadm -m session
# Should show a session to the peer's storage IP and IQN
```

### 2. Review the ZFS pool creation script

```bash
cat /root/create-pool.sh
```

Verify that `REMOTE_DISKS` contains the expected `/dev/disk/by-path/` paths from the iSCSI session. If the paths look wrong, check iSCSI session status first.

### 3. Create the ZFS pool

Run on `storage-a` only:

```bash
bash /root/create-pool.sh
```

This creates a mirrored pool using local disks from `host_vars` and remote disks discovered via iSCSI. Review the output carefully — the script prints the `zpool create` command before running it.

### 4. Export the pool

Pacemaker must import the pool itself on first start:

```bash
zpool export san-pool
```

### 5. Configure STONITH

```bash
bash /root/configure-stonith.sh
```

### 6. Configure Pacemaker resources

```bash
bash /root/configure-pacemaker-resources.sh
```

### 7. Verify cluster status

```bash
pcs status
```

All resources should show `Started` on one node. Both nodes should be online.

---

## Step 12: Verify the Deployment

Run the read-only verification playbook:

```bash
ansible-playbook -i inventory.yml verify.yml
```

This checks:
- Cluster quorum and node health
- Resource placement and VIP reachability
- iSCSI session state on both nodes
- ZFS pool status
- NFS/SMB service availability

### Manual failover test

Before putting the cluster into production, verify failover works:

```bash
# On storage-a: put it into standby (graceful migration)
pcs node standby storage-a

# Observe resources migrating to storage-b
watch pcs status

# Bring storage-a back
pcs node unstandby storage-a
```

This validates STONITH configuration, resource ordering, and VIP failover without a hard reboot.

---

## What's Next

- `docs/ha-san-ops.html` — day-to-day operations runbook
- `docs/variables.md` — full variable reference
- `docs/stonith-smart-plugs.md` — per-method STONITH setup
- `docs/dataset-best-practices.md` — ZFS dataset layout recommendations
- `docs/nfs-client-config.md` — NFS client mount options
- `docs/cluster-monitoring.md` — Prometheus/Alertmanager setup
