# NFS Security: Authentication Levels

## NFS Security Flavors

NFSv4 supports multiple authentication/integrity flavors via the `sec=` option:

| Flavor | Auth | Integrity | Privacy | Notes |
|--------|------|-----------|---------|-------|
| `sec=sys` | UID/GID only | None | None | Default; trust the client OS |
| `sec=krb5` | Kerberos | None | None | Identity proven; no signing |
| `sec=krb5i` | Kerberos | Signed | None | Prevents data tampering in transit |
| `sec=krb5p` | Kerberos | Signed | Encrypted | Full protection; highest CPU cost |

## Why `sec=sys` Is Used Here

This playbook uses `sec=sys` (the default) for NFS exports. This is intentional:

**Why it's acceptable:**
- The client VLAN (`vlans.client.subnet`) is a dedicated, isolated network
- Only trusted clients (Proxmox hypervisors, known workstations) have access
- iSCSI replication runs on a completely separate storage VLAN
- nftables firewall restricts NFS ports to the client subnet only
- No untrusted or internet-facing clients can reach the NFS VLAN

**What `sec=sys` trusts:**
- The client kernel presents the UID/GID of the requesting process
- Any root user on a client has root access to the NFS share (unless `root_squash` is set)
- A compromised or rogue client on the client VLAN can impersonate any UID

**Mitigations applied:**
- `root_squash` is set by default in the NFS exports (maps root→nobody)
- Client VLAN access is restricted to specific trusted subnets
- Storage VLAN is completely separate from the client-accessible network

## Why Not Kerberos?

Enabling `sec=krb5` would provide cryptographic identity verification, but requires:

1. **A Kerberos realm** — MIT Kerberos (`krb5-kdc`) or FreeIPA running on the network
2. **Service principal** — the NFS server needs a `nfs/hostname@REALM` keytab
3. **Client enrollment** — every NFS client must be enrolled as a Kerberos client and have a valid ticket
4. **Proxmox limitation** — Proxmox VE does not support NFSv4 Kerberos out of the box. Enabling it
   requires configuring Proxmox hosts as Kerberos clients, installing `krb5-user`, and managing
   keytab distribution — this is not feasible in most homelab/SMB environments without a full
   identity management infrastructure

**Kerberos is appropriate when:**
- You have FreeIPA or Active Directory providing Kerberos services
- Clients are domain-joined machines with managed credentials
- The client VLAN is not fully trusted (e.g., shared with other departments)
- Regulatory compliance requires in-transit encryption (`sec=krb5p`)

## Hardening `sec=sys` Deployments

If you choose to stay with `sec=sys`, apply these mitigations:

```bash
# In /etc/exports (managed via Ansible):
# root_squash — prevents root on client from having root access server-side
# all_squash — maps ALL users to anonymous uid/gid (for public shares only)
# no_subtree_check — more reliable (subtree_check has known issues)

/san-pool/nfs  10.30.30.0/24(rw,sync,no_subtree_check,root_squash)
```

Avoid `no_root_squash` unless you have a specific operational need (e.g., NFS-root diskless clients).

## References

- [Linux NFS HOWTO — Security](https://nfs.sourceforge.net/nfs-howto/ar01s06.html)
- [RFC 7530 — NFSv4](https://datatracker.ietf.org/doc/html/rfc7530)
- [MIT Kerberos Documentation](https://web.mit.edu/kerberos/krb5-latest/doc/)
