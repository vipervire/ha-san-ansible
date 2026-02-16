# Cluster Health Monitoring Guide

This guide covers monitoring your Pacemaker/Corosync HA cluster using **ha_cluster_exporter** with Prometheus and NTFY alerts.

## Overview

The playbook deploys **ha_cluster_exporter** on all cluster nodes (storage-a, storage-b, quorum) to expose Pacemaker and Corosync metrics to Prometheus.

**What's monitored:**
- Cluster quorum status (split-brain detection)
- Node online/offline status
- Resource health and location (failover detection)
- STONITH/fencing status
- Corosync ring health (network connectivity)
- Resource migration and failure counts

## Metrics Endpoint

ha_cluster_exporter runs on each node:
- **Port**: 9664
- **Listens on**: Management VLAN only (security)
- **Service**: `prometheus-hacluster-exporter`
- **Metrics path**: `http://<node-ip>:9664/metrics`

## Quick Verification

```bash
# Check exporter is running on all nodes
ssh storage-a "systemctl status prometheus-hacluster-exporter"
ssh storage-b "systemctl status prometheus-hacluster-exporter"
ssh quorum "systemctl status prometheus-hacluster-exporter"

# View metrics from active node
curl http://10.20.20.1:9664/metrics | grep ha_cluster

# Key metrics to check
curl http://10.20.20.1:9664/metrics | grep -E "(quorate|pacemaker_nodes|pacemaker_resources|stonith_enabled)"
```

## Prometheus Configuration

Add to your Prometheus server's `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'ha_cluster'
    scrape_interval: 30s
    static_configs:
      - targets:
          - '10.20.20.1:9664'  # storage-a
          - '10.20.20.2:9664'  # storage-b
          - '10.20.20.3:9664'  # quorum
        labels:
          cluster: 'san-cluster'
          environment: 'production'
```

Reload Prometheus:
```bash
systemctl reload prometheus
```

Verify targets are UP:
```bash
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job=="ha_cluster")'
```

## Key Metrics Reference

### Quorum and Membership

```prometheus
# Cluster has quorum (1=yes, 0=no) - CRITICAL metric
ha_cluster_corosync_quorate

# Total expected votes vs current votes
ha_cluster_corosync_quorum_votes

# Per-node voting power
ha_cluster_corosync_member_votes
```

### Node Status

```prometheus
# Node status (member, online/offline/standby)
ha_cluster_pacemaker_nodes{node="storage-a",type="member",status="online"}

# Node attributes
ha_cluster_pacemaker_node_attributes
```

### Resource Status

```prometheus
# Resource state (which node is active)
ha_cluster_pacemaker_resources{resource="san-resources",role="Started",node="storage-a"}

# Resource failure counts
ha_cluster_pacemaker_fail_count{resource="san-resources",node="storage-a"}

# Migration threshold
ha_cluster_pacemaker_migration_threshold
```

### STONITH Status

```prometheus
# STONITH enabled (1=yes, 0=no)
ha_cluster_pacemaker_stonith_enabled
```

### Corosync Rings

```prometheus
# Ring health (1=healthy, 0=faulty)
ha_cluster_corosync_rings{node="storage-a",ring_id="0"}

# Ring errors
ha_cluster_corosync_ring_errors
```

## Alert Rules

The playbook includes comprehensive alert rules in `docs/prometheus-alerts.yml`. Load them on your Prometheus server:

```yaml
# In prometheus.yml
rule_files:
  - "rules/prometheus-alerts.yml"
```

### Critical Alerts

- **ClusterQuorumLost** - Cluster lost quorum (30s threshold)
- **ClusterNodeOffline** - Node offline for >2 minutes
- **ClusterResourceFailed** - Resource has failures
- **StonithDisabled** - STONITH disabled (unsafe!)
- **ClusterSplitBrainRisk** - Multiple nodes reporting quorum
- **ResourceMigrationThresholdReached** - Resource about to migrate

### Warning Alerts

- **ClusterResourceStopped** - Managed resource stopped (5 min)
- **CorosyncRingFaulty** - Ring has errors (2 min)
- **CorosyncMembershipChanges** - Frequent changes (5 min)
- **HAClusterExporterDown** - Exporter not responding (3 min)

### Info Alerts

- **ResourceFailoverDetected** - Normal failover event (awareness only)

## Common Scenarios

### Scenario 1: Planned Failover

```bash
# Trigger failover
pcs resource move san-resources storage-b

# Watch metrics change
watch -n 1 'curl -s http://10.20.20.1:9664/metrics | grep pacemaker_resources'

# Expected alerts:
# - ResourceFailoverDetected (info) - normal

# Clear constraint
pcs resource clear san-resources
```

### Scenario 2: Node Offline (Unplanned)

**What happens:**
1. Node loses connectivity or crashes
2. After 2 minutes: `ClusterNodeOffline` alert fires
3. Pacemaker detects node down (~4-5 seconds)
4. Resources migrate to surviving node
5. `ResourceFailoverDetected` alert fires
6. Quorum maintained (2 of 3 nodes still up)

**Metrics to check:**
```bash
# Node status
curl http://10.20.20.2:9664/metrics | grep 'pacemaker_nodes.*storage-a'

# Quorum status (should still be 1)
curl http://10.20.20.2:9664/metrics | grep 'corosync_quorate'

# Resource location (should show storage-b)
curl http://10.20.20.2:9664/metrics | grep 'pacemaker_resources.*san-resources'
```

### Scenario 3: Network Partition

**Single ring failure:**
- `CorosyncRingFaulty` warning alert
- Cluster continues on remaining ring
- Fix network, ring recovers automatically

**Both rings fail (split-brain risk):**
- Cluster partitions based on quorum votes
- Majority partition (2+ nodes) keeps quorum
- Minority partition loses quorum → stops resources
- `ClusterQuorumLost` critical alert on minority
- STONITH fences nodes in minority partition

### Scenario 4: Resource Failure

**What happens:**
1. Resource fails to start or crashes
2. Pacemaker attempts restart (per resource policy)
3. After threshold failures: migration to other node
4. `ClusterResourceFailed` critical alert
5. `ResourceMigrationThresholdReached` if at limit

**Response:**
```bash
# Check resource status
pcs resource status san-resources

# Check failure count
pcs resource failcount show san-resources

# Check logs
journalctl -u pacemaker -n 100

# Clear failcount after fixing issue
pcs resource cleanup san-resources
```

## Grafana Dashboard

Import the official HA Cluster dashboard for visualization:

1. Go to Grafana → Dashboards → Import
2. Enter dashboard ID: **12229**
3. Select your Prometheus datasource
4. Customize variables:
   - cluster: `san-cluster`
   - node: `storage-a|storage-b|quorum`

**Dashboard includes:**
- Cluster quorum status
- Node online/offline visualization
- Resource location and status
- Ring health indicators
- Failure count graphs
- Historical failover events

## Integration with ZFS Monitoring

Cluster monitoring complements ZFS monitoring for comprehensive visibility:

| Metric | ZFS Exporter | Cluster Exporter | Combined Insight |
|--------|--------------|------------------|------------------|
| **Pool owner** | `zfs_scrub_pool_imported` | `ha_cluster_pacemaker_resources` | Detect split-brain or lag |
| **Node health** | Pool health on active node | All nodes' cluster status | Correlate storage and cluster failures |
| **Failover** | Pool import changes | Resource location changes | Full failover timeline |

**Example combined alert:**
```yaml
# Alert if ZFS shows pool on both nodes AND cluster shows split
- alert: ConfirmedSplitBrain
  expr: |
    sum(zfs_scrub_pool_imported{pool="san-pool"}) > 1
    and count(ha_cluster_corosync_quorate == 1) > 1
  for: 30s
  labels:
    severity: critical
  annotations:
    summary: "Confirmed split-brain: both ZFS and cluster report dual ownership"
```

## Troubleshooting

### Exporter Not Starting

```bash
# Check service status
systemctl status prometheus-hacluster-exporter

# Check logs
journalctl -u prometheus-hacluster-exporter -n 50

# Common issues:
# - Pacemaker not running: systemctl start pacemaker
# - Port conflict: check port 9664 availability
# - Permission errors: exporter needs to run as root (default)
```

### Missing Metrics

```bash
# Test local metrics endpoint
curl http://localhost:9664/metrics

# If empty or errors:
# 1. Verify cluster is running: pcs status
# 2. Check crm_mon works: crm_mon -1
# 3. Check corosync: corosync-quorumtool
# 4. Restart exporter: systemctl restart prometheus-hacluster-exporter
```

### Prometheus Not Scraping

```bash
# Check Prometheus targets page
# http://your-prometheus:9090/targets

# If target is DOWN:
# 1. Verify exporter is listening: ss -tlnp | grep 9664
# 2. Check firewall: nftables on management VLAN should allow 9664
# 3. Test from Prometheus server: curl http://10.20.20.1:9664/metrics
# 4. Check Prometheus scrape_configs in prometheus.yml
```

### Alerts Not Firing

```bash
# Check Prometheus alerts page
# http://your-prometheus:9090/alerts

# Verify rules loaded:
curl http://localhost:9090/api/v1/rules | jq '.data.groups[] | select(.name=="ha_cluster_alerts")'

# If rules missing:
# 1. Check rule_files in prometheus.yml
# 2. Validate YAML syntax: promtool check rules prometheus-alerts.yml
# 3. Reload Prometheus: systemctl reload prometheus
```

## Best Practices

1. **Monitor the monitors**: Set up `HAClusterExporterDown` alert
2. **Baseline normal behavior**: Observe metrics during normal operation
3. **Test failovers**: Practice planned failovers and verify alerts
4. **Correlate events**: Use Grafana to overlay ZFS and cluster metrics
5. **Tune alert thresholds**: Adjust for values based on your network latency
6. **Document procedures**: Create runbooks for each alert type
7. **Review regularly**: Check historical metrics for trends

## Security Considerations

- Exporter runs on management VLAN only (10.20.20.0/24)
- No authentication on exporter (network-level security)
- Read-only operations (cannot modify cluster)
- Metrics may contain hostnames and resource names
- Use firewall rules to restrict Prometheus server access

## References

- [ha_cluster_exporter GitHub](https://github.com/ClusterLabs/ha_cluster_exporter)
- [Metrics Documentation](https://github.com/ClusterLabs/ha_cluster_exporter/blob/main/doc/metrics.md)
- [Grafana Dashboard #12229](https://grafana.com/grafana/dashboards/12229-ha-cluster-details/)
- [Pacemaker Documentation](https://clusterlabs.org/pacemaker/doc/)
- [Corosync Documentation](https://clusterlabs.org/corosync/doc/)
