HA SAN Ansible Playbook — Improvement Recommendations
Context
This is an analysis of the existing ha-san-ansible playbook — a production-grade ZFS-over-iSCSI HA SAN on Debian 12 using Pacemaker/Corosync. The playbook is already well-engineered. The recommendations below identify gaps and enhancement opportunities across five dimensions. Items are roughly ordered by impact within each section.

Security
S1 — TLS for Prometheus Metrics Endpoints (Medium)
node_exporter (9100) and ha_cluster_exporter (9664) serve plaintext HTTP. Anyone on the management VLAN can read metrics without authentication. Add --web.config.file to both exporters to enable TLS and/or basic auth. Critical files:

roles/monitoring/tasks/main.yml
roles/monitoring/templates/ha-cluster-exporter-default.j2
roles/hardening/templates/nftables.conf.j2 (already restricts by subnet, which partially mitigates this)
S2 — LIO Target saveconfig.json File Permissions (Medium)
/etc/iscsi/iscsid.conf is correctly deployed at mode: 0600. However, /etc/target/saveconfig.json (which stores the LIO target configuration including CHAP credentials in plaintext) should also be verified at mode: 0600. The iscsi-target role does set permissions on this file per CLAUDE.md, but it's worth confirming the task explicitly sets 0600 rather than relying on targetcli's default.

roles/iscsi-target/tasks/main.yml
S3 — No Secrets Rotation Mechanism (Medium)
There's no playbook or procedure for rotating credentials (iSCSI CHAP, hacluster password, STONITH passwords) without a full re-run. A targeted rotate-secrets.yml playbook or a --tags secrets path would reduce operational risk when credentials need to change.

S4 — No Audit Logging (auditd) (Medium)
The hardening role covers firewall, SSH, sysctl, and PAM faillock, but deploys no auditd. Critical paths like /etc/corosync/, /etc/pacemaker/, iSCSI configurations, and su/sudo usage should be audited. Relevant role: roles/hardening/.

S5 — NFS Uses sec=sys (Host-Trust Only) (Low/Informational)
Documented in docs/nfs-security.md but worth surfacing: sec=sys trusts UIDs from clients. If a client is compromised, any UID can be spoofed. Consider sec=krb5p for any shares exposed to untrusted clients, or add a note to exports forcing root_squash and all_squash where appropriate.

S6 — iSCSI Client Target Has No Per-Initiator ACLs (High)
The client-facing iSCSI target (served on vip_iscsi) is configured by the services role, but there's no template for LIO ACLs or CHAP on the client target. Any initiator on the client VLAN can connect. Per-initiator ACLs + CHAP should be added.

roles/services/tasks/main.yml (currently no iSCSI target config)
S7 — pcsd Uses Self-Signed TLS (Low)
pcs host auth uses self-signed certificates for the pcsd daemon. For production use, replace with CA-signed certificates or at minimum pin the expected certificate fingerprint in the automation.

S8 — No Rate Limiting for Cockpit/pcsd (Low)
PAM faillock protects local logins but Cockpit (9090) and pcsd (2224) have no brute-force protection at the network level. fail2ban or nftables rate-limiting on these ports would reduce exposure.

S9 — No Cockpit TLS Certificate Management (Medium)
Cockpit serves HTTPS on port 9090 but the playbook doesn't manage its TLS certificate — Cockpit generates a self-signed cert on first run. For production use, the certificate should be replaced with one from an internal CA or Let's Encrypt. A post-deploy task to install a certificate (or a reverse proxy template) would complete the security posture.

roles/cockpit/tasks/main.yml
S10 — No Central Syslog Forwarding (Low)
PAM faillock audits, nftables drop logs (nft-drop: prefix), Corosync, and Pacemaker events are all logged locally. There is no rsyslog or journald forwarding configured to a central SIEM or log aggregator. For production environments, forward at minimum auth, kern, and daemon facilities.

roles/hardening/tasks/main.yml
Performance
P1 — SLOG Not Wired into Pool Creation Script (High)
slog_disk and special_disk variables are defined in host_vars/ and commented-out placeholders exist, but roles/iscsi-initiator/templates/create-pool.sh.j2 does not include them in the zpool create command. SLOG dramatically accelerates sync writes (NFS sync, database commits). This is the highest-impact missing performance feature.

roles/iscsi-initiator/templates/create-pool.sh.j2
P2 — Single-Path iSCSI Backend (Medium)
The storage VLAN uses a single NIC (net_storage_parent). If that NIC or its switch port fails, the iSCSI mirror leg goes offline and the pool degrades. Adding a second storage NIC + bonding or a second iSCSI session via multipathd would eliminate this single point of failure.

group_vars/storage_nodes.yml (network interface vars)
roles/iscsi-initiator/templates/iscsid.conf.j2
P3 — TCP Congestion Control Not Set (Medium)
tcp_tunables sets buffer sizes but doesn't set net.ipv4.tcp_congestion_control. For 40GbE with high BDP (Bandwidth-Delay Product), bbr outperforms the default cubic. Add to group_vars/storage_nodes.yml:


tcp_tunables:
  net.ipv4.tcp_congestion_control: bbr
  net.core.default_qdisc: fq
P4 — ZFS Prefetch Not Tuned for Mixed Workloads (Medium)
For the iscsi dataset (VM storage), ZFS's sequential prefetch can hurt random-access IOPS. Consider zfs_prefetch_disable: 1 for the iscsi dataset workload profile, or document this in docs/dataset-best-practices.md.

group_vars/storage_nodes.yml (zfs_modprobe_options)
P5 — iSCSI Session Queue Depth Conservative (Low)
node.session.queue_depth = 32 is a safe default but modern enterprise SSDs can handle 64–128. For NVMe-backed iSCSI, increasing this would reduce queue saturation under bursty VM I/O.

roles/iscsi-initiator/templates/iscsid.conf.j2
P6 — No L2ARC Configuration (Low)
No provision for L2ARC (read cache) devices. For read-heavy NFS/SMB workloads with available NVMe, L2ARC can significantly reduce latency. A l2arc_disk variable in host_vars/ and integration into the pool creation script would mirror the SLOG pattern.

P7 — NFS Client Mount Options Not Documented (Low)
nfs.conf.j2 tunes the server side but there's no documentation of recommended client-side mount options (e.g., rsize=1048576,wsize=1048576,nconnect=4,proto=tcp) for 40GbE. Add to docs/.

Stability
ST1 — No SMART Disk Monitoring (High)
smartmontools is not deployed. Disk health monitoring is table stakes for production storage. A failing drive gives weeks of warning via SMART attributes before failure. Add smartd deployment to roles/monitoring/ or roles/common/.

ST2 — ZED Events Not Wired to Alerting (High)
zfs-zed is enabled and running, but ZED event handlers (/etc/zfs/zed.d/) are not configured. Pool degradation, checksum errors, and resilver completion should trigger ntfy/email alerts. The docs/ntfy-integration.md documents the alerting stack but ZED is not connected to it.

roles/zfs/tasks/main.yml
docs/ntfy-integration.md
ST3 — No Hardware Watchdog Configuration (Medium)
If the kernel panics or hangs, without a hardware watchdog the node won't reset itself, and the peer can't fence it because it's unresponsive but not dead. /dev/watchdog should be configured via watchdog package or iTCO_wdt kernel module and integrated with Pacemaker's pacemaker-watchdog (SBD-style).

ST4 — Pacemaker Health Check Resources Missing (Medium)
No ocf:pacemaker:HealthCPU or ocf:pacemaker:HealthIOWait resources. These trigger preventive failover when a node becomes overloaded before clients experience failures. Useful for catching runaway ZFS scrubs or resilvers.

roles/pacemaker/templates/configure-resources.sh.j2
ST5 — NFS Grace Period Not Configured (Medium)
After failover, NFS clients must reclaim open files within the server's grace period. nfs_grace_period and nfs_leasetime are not configured in nfs.conf.j2. The defaults (90s grace, 90s lease) may be too long for client experience. Should be documented and tuned.

roles/services/templates/nfs.conf.j2
ST6 — No Clock Drift Alert (Medium)
Corosync token timeouts fail if clock skew exceeds safe bounds. Chrony is deployed ✓, but there's no Prometheus alert for node_timex_offset_seconds exceeding a threshold. Add to docs/prometheus-alerts.yml.

ST7 — iSCSI Session Recovery After Partition Not Documented (Low)
The ops runbook (ha-san-ops.html) exists but there's no specific procedure for recovering iSCSI sessions after a storage VLAN partition and re-join. After partition, ZFS may need to resilver and iscsid may need a session reset. A recovery checklist would be valuable.

ST8 — No Memory Error Monitoring (Low)
No rasdaemon or mcelog deployment. ECC memory errors are an early warning for hardware failure that can lead to data corruption. Add to roles/monitoring/.

Flexibility
F1 — SLOG/Special Vdev Variables Defined But Not Implemented (High)
Mirrors P1 above from a flexibility perspective. The host_vars variables and template stubs exist, signaling user intent, but the feature isn't functional. Users will expect it to work.

F2 — Everything Hardcoded to san-pool (Medium)
Pool name is parameterized via zfs_pool_name but many paths, scripts, and templates assume a single pool. Supporting multiple pools (e.g., separate NVMe tier and HDD tier) would require significant refactoring. Document this limitation explicitly or add multi-pool support with a zfs_pools list variable.

F3 — Custom Pacemaker Resources Require Template Edits (Medium)
Adding a custom resource (e.g., a VIP for a custom app) requires editing configure-resources.sh.j2. A pacemaker_extra_resources variable list in group_vars/ with shell command strings would allow extensions without template modifications.

roles/pacemaker/templates/configure-resources.sh.j2
F4 — No Networking Role (Medium)
Interface configuration (VLANs, bonds, MTU for jumbo frames on storage VLAN) is left entirely to manual setup. A networking role using systemd-networkd or ifupdown templates would complete the automation story, especially important for jumbo frames (MTU 9000) on the storage interconnect which significantly impacts iSCSI throughput.

F5 — NFSv3 Enabled by Default (Low)
NFSv4-only simplifies the firewall significantly (no portmapper, mountd, or statd ports — just 2049) and removes legacy security surface. Make NFS version configurable with NFSv4-only as the recommended default, with NFSv3 opt-in.

roles/services/templates/nfs.conf.j2
roles/hardening/templates/nftables.conf.j2
F6 — No Syncoid (Offsite Replication) Support (Low)
Sanoid is configured for local snapshots. Syncoid (part of the same project) enables ZFS snapshot replication to a remote host. A syncoid_targets variable and systemd service template would add off-site backup capability with minimal additional code.

roles/zfs/tasks/main.yml
F7 — No Roles defaults/main.yml (Low)
Roles have no defaults/main.yml files. All variables live in group_vars/. Without defaults, users can't discover what's configurable without reading both the tasks and the group_vars simultaneously. Adding defaults/main.yml to each role with documented variables improves discoverability.

User Experience
U1 — Post-Deployment Verification Playbook Missing (High)
There's no verify.yml or test.yml that confirms the deployed state: cluster quorum, resource status, iSCSI sessions active, ZFS pool healthy, all exporters responding, all firewall ports accessible. This is the most common pain point after initial deployment.

U2 — iSCSI Path FIXME in create-pool.sh Requires Manual Edit (High)
The generated script contains FIXME placeholders for remote iSCSI device paths that the user must manually fill in. The script could auto-discover paths from iscsiadm -m session -P3 and populate them automatically (with user confirmation), reducing a fragile manual step.

roles/iscsi-initiator/templates/create-pool.sh.j2
U3 — Pre-Run Cluster Health Check Missing (Medium)
Re-running the playbook against a live cluster without checking its current state risks disrupting services (e.g., restarting Corosync on the active node). A pre-flight play that checks pcs status and warns if the cluster is degraded would prevent accidental outages.

site.yml (add to pre-flight play)
U4 — No Alerting Deployment (ntfy) Despite Documentation (Medium)
docs/ntfy-integration.md is thorough, but the actual Alertmanager configuration and ntfy routing isn't deployed by the playbook. Users who follow the monitoring setup end up with metrics but no alerts. Deploying at minimum an example alertmanager.yml template would close this gap.

U5 — No Upgrade Procedure (Medium)
No documentation or playbook for upgrading the cluster components (ZFS DKMS, Debian kernel, Pacemaker, Corosync) while maintaining HA. The correct sequence (standby node B → upgrade → failover → upgrade A) is non-obvious and critical to get right.

U6 — Role Variable Documentation (Low)
Variables are spread across group_vars/all.yml, group_vars/storage_nodes.yml, and host_vars/. Each role's configurable variables should be listed in a comment block at the top of defaults/main.yml or in README.md per role. Reduces time to first deployment.

U7 — Molecule or CI Testing Framework (Low)
No automated testing. Changes to firewall rules, NFS config, or Pacemaker templates can't be validated without deploying to real hardware. Even a basic Molecule scenario with mock inventory would catch template errors and missing variables before deployment.

U8 — Example host_vars with Realistic Disk IDs (Low)
Current host_vars/ use PLACEHOLDER disk names. Providing example entries (e.g., ata-Samsung_SSD_870_EVO_..., nvme-WD_Black_SN850X_...) with comments explaining how to find real disk IDs (ls -la /dev/disk/by-id/) would reduce initial setup friction.

Summary Table
ID	Area	Item	Impact
S2	Security	LIO saveconfig.json permissions verification	Medium
S6	Security	iSCSI client target has no ACLs	High
P1	Performance	SLOG not wired into create-pool.sh	High
ST1	Stability	No SMART disk monitoring	High
ST2	Stability	ZED not wired to alerting	High
F1	Flexibility	SLOG/special vdev variables not implemented	High
U1	UX	No post-deployment verification playbook	High
U2	UX	create-pool.sh iSCSI FIXME is manual	High
S1	Security	TLS for Prometheus endpoints	Medium
S3	Security	No secrets rotation mechanism	Medium
S4	Security	No auditd	Medium
P2	Performance	Single-path iSCSI backend	Medium
P3	Performance	TCP congestion control not set (BBR)	Medium
ST3	Stability	No hardware watchdog	Medium
ST4	Stability	No Pacemaker health check resources	Medium
ST5	Stability	NFS grace period not configured	Medium
ST6	Stability	No clock drift alert	Medium
F3	Flexibility	Custom Pacemaker resources require edits	Medium
F4	Flexibility	No networking role (MTU/bond/VLAN)	Medium
U3	UX	No pre-run cluster health check	Medium
U4	UX	No alerting deployment (ntfy)	Medium
U5	UX	No upgrade procedure	Medium
S9	Security	No Cockpit TLS certificate management	Medium
S10	Security	No central syslog forwarding	Low
Verification
This is a recommendations document, not an implementation plan. No code changes are proposed. To validate any implemented recommendation:

Security items: nft list ruleset, openssl s_client, curl -k https://, file permission checks
Performance items: zpool status (SLOG shown as log vdev), fio benchmarks, iperf3
Stability items: smartctl -a /dev/sdX, zpool events, pcs status
Flexibility items: test with modified group_vars/, dry-run ansible-playbook --check --diff
UX items: run verify.yml against a test deployment