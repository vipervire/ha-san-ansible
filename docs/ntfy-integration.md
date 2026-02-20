# NTFY Integration for ZFS Scrub Alerts

This guide shows how to configure Prometheus Alertmanager to send ZFS scrub alerts to NTFY.

## Architecture

```
Storage Nodes (storage-a, storage-b)
  └─ node_exporter (port 9100)
      └─ textfile collector
          └─ zfs_scrub.prom (metrics updated every 5 min)
            ↓
Prometheus Server
  ├─ Scrapes metrics from storage nodes
  ├─ Evaluates alert rules (prometheus-alerts.yml)
  └─ Sends alerts to Alertmanager
      ↓
Alertmanager
  └─ Routes alerts to NTFY webhook
      ↓
NTFY (ntfy.sh or self-hosted)
  └─ Sends push notifications to your devices
```

## Prerequisites

1. **Prometheus server** scraping your storage nodes (on port 9100)
2. **Alertmanager** configured to receive alerts from Prometheus
3. **NTFY** topic (e.g., `https://ntfy.sh/your-san-alerts` or self-hosted)

## Step 1: Configure Prometheus

Add your storage nodes as scrape targets in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'storage-nodes'
    static_configs:
      - targets:
          - '10.20.20.1:9100'  # storage-a management IP
          - '10.20.20.2:9100'  # storage-b management IP
        labels:
          cluster: 'san-cluster'
          environment: 'production'
```

## Step 2: Add Alert Rules

Copy `docs/prometheus-alerts.yml` to your Prometheus rules directory and load it:

```yaml
# In prometheus.yml
rule_files:
  - "rules/prometheus-alerts.yml"
```

Reload Prometheus:
```bash
systemctl reload prometheus
# or
curl -X POST http://localhost:9090/-/reload
```

Verify rules are loaded:
```bash
# Check Prometheus UI: http://your-prometheus:9090/alerts
```

## Step 3: Configure Alertmanager for NTFY

### Option A: NTFY Cloud (ntfy.sh)

Create or update `/etc/alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'ntfy-default'
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 12h

  # Route critical alerts immediately, warnings with delay
  routes:
    - match:
        severity: critical
      receiver: 'ntfy-critical'
      group_wait: 0s
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: 'ntfy-warning'
      group_wait: 30s
      repeat_interval: 12h

receivers:
  - name: 'ntfy-critical'
    webhook_configs:
      - url: 'https://ntfy.sh/your-san-alerts-critical'
        send_resolved: true
        http_config:
          follow_redirects: true

  - name: 'ntfy-warning'
    webhook_configs:
      - url: 'https://ntfy.sh/your-san-alerts'
        send_resolved: true
        http_config:
          follow_redirects: true

  - name: 'ntfy-default'
    webhook_configs:
      - url: 'https://ntfy.sh/your-san-alerts'
        send_resolved: true

inhibit_rules:
  # Suppress warning if critical is firing for same alertname
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

**Important:** Replace `your-san-alerts` and `your-san-alerts-critical` with your actual NTFY topic names.

### Option B: Self-Hosted NTFY

If you're running your own NTFY server with authentication:

```yaml
receivers:
  - name: 'ntfy-critical'
    webhook_configs:
      - url: 'https://ntfy.example.com/san-alerts-critical'
        send_resolved: true
        http_config:
          follow_redirects: true
          authorization:
            type: Bearer
            credentials: 'your-ntfy-access-token'
```

## Step 4: Format Alert Messages for NTFY

Alertmanager webhook sends alerts in JSON format. NTFY expects specific headers for formatting. You have two options:

### Option A: Use ntfy-alertmanager Bridge (Recommended)

Install [ntfy-alertmanager](https://github.com/xenrox/ntfy-alertmanager) as a bridge:

```bash
# On your Alertmanager server or separate host
docker run -d \
  --name ntfy-alertmanager \
  -p 8080:8080 \
  -e NTFY_TOPIC=https://ntfy.sh/your-san-alerts \
  -e NTFY_PRIORITY=default \
  -e NTFY_TAGS=warning \
  xenrox/ntfy-alertmanager:latest
```

Then point Alertmanager to the bridge:

```yaml
receivers:
  - name: 'ntfy-critical'
    webhook_configs:
      - url: 'http://localhost:8080/alerts'  # ntfy-alertmanager bridge
        send_resolved: true
```

The bridge formats messages nicely and adds proper NTFY headers.

### Option B: Direct NTFY Webhook (Simple but Limited Formatting)

NTFY accepts webhook POSTs directly, but formatting is basic:

```yaml
receivers:
  - name: 'ntfy-critical'
    webhook_configs:
      - url: 'https://ntfy.sh'
        send_resolved: true
        http_config:
          follow_redirects: true
        # Send custom headers for NTFY
        # Note: Alertmanager doesn't support custom headers in webhook_configs easily
        # Consider using ntfy-alertmanager bridge instead
```

**Limitation:** Direct webhook has poor message formatting. Use the bridge for better UX.

## Step 5: Test Alerts

### Test 1: Manual Alert Trigger

Trigger a test alert by manually running the scrub exporter with a fake failure:

```bash
# On storage-a
ssh storage-a
sudo systemctl stop zfs-scrub-exporter.timer
sudo rm /var/lib/prometheus/node-exporter/zfs_scrub.prom
# Wait 5+ minutes for Prometheus to detect missing metrics
```

### Test 2: Simulate Overdue Scrub

Manually create metrics showing an old scrub:

```bash
echo 'zfs_scrub_last_run_timestamp_seconds{pool="san-pool"} 1609459200' | \
sudo tee /var/lib/prometheus/node-exporter/zfs_scrub.prom
# This timestamp is from Jan 1, 2021 - will trigger ZFSScrubOverdue
```

### Test 3: Send Test Alert from Alertmanager

```bash
# On Alertmanager server
amtool alert add alertname=ZFSScrubTest severity=warning
```

## Step 6: Subscribe to NTFY Notifications

### Mobile Apps
- **iOS**: [NTFY iOS App](https://apps.apple.com/us/app/ntfy/id1625396347)
- **Android**: [NTFY Android App](https://play.google.com/store/apps/details?id=io.heckel.ntfy)

### Web Browser
Visit: `https://ntfy.sh/your-san-alerts`

### Command Line
```bash
# Subscribe to alerts in terminal
ntfy sub your-san-alerts
```

## Alert Types You'll Receive

Based on the Prometheus alert rules:

**ZFS Storage Alerts:**
1. **ZFSScrubOverdue** (warning) - No scrub in 35+ days
2. **ZFSScrubNeverRun** (warning) - Pool never scrubbed
3. **ZFSScrubErrors** (critical) - Data corruption detected
4. **ZFSPoolDegraded** (critical) - Pool health degraded
5. **ZFSScrubExporterFailed** (warning) - Metrics collection issue
6. **ZFSScrubStalled** (warning) - Scrub running >12 hours
7. **ZFSPoolSplitBrain** (critical) - Pool imported on multiple nodes

**Cluster Health Alerts:**
8. **ClusterQuorumLost** (critical) - Cluster lost quorum
9. **ClusterNodeOffline** (critical) - Node offline for >2 minutes
10. **ClusterResourceFailed** (critical) - Resource has failures
11. **StonithDisabled** (critical) - STONITH fencing disabled
12. **ClusterSplitBrainRisk** (critical) - Multiple nodes reporting quorum
13. **ResourceMigrationThresholdReached** (critical) - Resource about to migrate
14. **ClusterResourceStopped** (warning) - Managed resource stopped
15. **CorosyncRingFaulty** (warning) - Corosync ring errors
16. **CorosyncMembershipChanges** (warning) - Frequent membership changes
17. **HAClusterExporterDown** (warning) - Cluster exporter not responding
18. **ResourceFailoverDetected** (info) - Normal failover event

**Alert Routing:**
- Critical alerts (3, 4, 7-13) → `ntfy-critical` topic (urgent notifications)
- Warning alerts (1, 2, 5, 6, 14-17) → `ntfy-warning` topic (standard notifications)
- Info alerts (18) → `ntfy-default` topic (awareness only)

## Advanced: Priority and Tags

If using ntfy-alertmanager bridge, configure per-severity routing:

```yaml
# In ntfy-alertmanager config
ntfy:
  topic: "san-alerts"
  priority_map:
    critical: "urgent"
    warning: "default"
    info: "low"
  tags_map:
    critical: "🚨,storage,critical"
    warning: "⚠️,storage,warning"
```

## Troubleshooting

### Metrics Not Appearing in Prometheus

```bash
# Check if exporter is running
systemctl status zfs-scrub-exporter.timer
systemctl list-timers zfs-scrub-exporter.timer

# Check metrics file exists
cat /var/lib/prometheus/node-exporter/zfs_scrub.prom

# Check node_exporter is serving metrics
curl http://10.20.20.1:9100/metrics | grep zfs_scrub
```

### Alerts Not Firing

```bash
# Check Prometheus can reach storage nodes
curl http://your-prometheus:9090/api/v1/targets

# Check alert rules are loaded
curl http://your-prometheus:9090/api/v1/rules

# Check if alert is pending/firing
curl http://your-prometheus:9090/api/v1/alerts
```

### NTFY Not Receiving Messages

```bash
# Test NTFY directly
curl -d "Test message" https://ntfy.sh/your-san-alerts

# Check Alertmanager logs
journalctl -u alertmanager -f

# Verify webhook is configured
amtool config show
```

## Example Alert Message

When `ZFSScrubOverdue` fires, you'll receive:

```
🚨 ZFS Scrub Overdue
Pool: san-pool
Node: storage-a (10.20.20.1)
Last scrub: 38 days ago

Monthly scrubs are recommended for data integrity.
Run: systemctl start zfs-scrub@san-pool.service
```

## Security Considerations

1. **NTFY Topic Names**: Use long, random topic names for public ntfy.sh
   - Bad: `san-alerts` (easily guessable)
   - Good: `san-alerts-8f3k2j9x` (random suffix)

2. **Self-Hosted NTFY**: Use authentication and HTTPS
3. **Alertmanager**: Restrict access to management VLAN only
4. **Sensitive Data**: Alert messages contain pool names and hostnames - ensure topics are private

## Dead-Man's Switch with Uptime Kuma

The `Watchdog` alert always fires (it uses `expr: vector(1)`), confirming the alerting pipeline
(Prometheus → Alertmanager → ntfy) is functioning end-to-end. If the Watchdog heartbeat stops
arriving, something in the pipeline is broken — even if no real alerts are firing.

### Setup

1. **Deploy Uptime Kuma** (self-hosted recommended):
   ```bash
   docker run -d --name uptime-kuma -p 3001:3001 -v uptime-kuma:/app/data louislam/uptime-kuma:1
   ```

2. **Create a Push monitor** in Uptime Kuma:
   - Type: **Push**
   - Heartbeat Interval: **5 minutes**
   - Note the push URL shown: `https://uptime-kuma.example.com/api/push/<token>`

3. **Add Watchdog receiver to Alertmanager**:
   ```yaml
   receivers:
     - name: 'uptime-kuma-watchdog'
       webhook_configs:
         - url: 'https://uptime-kuma.example.com/api/push/<token>?status=up&msg=OK'
           send_resolved: false
   ```

4. **Add Watchdog route** (before all other routes so it matches first):
   ```yaml
   routes:
     - match:
         alertname: Watchdog
       receiver: uptime-kuma-watchdog
       group_wait: 0s
       repeat_interval: 5m
     # ... existing routes below
   ```

5. **Configure Uptime Kuma notifications** — set Uptime Kuma to notify you (via ntfy, email, etc.)
   when the push heartbeat stops arriving.

Uptime Kuma will alert you via its own notification system if the Prometheus/Alertmanager pipeline
goes silent for more than one missed interval.

### Additional Alertmanager Inhibit Rules

These suppress downstream noise when a root-cause alert is firing. Add to `alertmanager.yml`:

```yaml
inhibit_rules:
  # Existing: suppress warnings when critical fires for same resource
  - source_match: { severity: 'critical' }
    target_match: { severity: 'warning' }
    equal: ['alertname', 'instance']

  # When a cluster node is offline, suppress resource/service alerts from that node
  - source_match: { alertname: 'ClusterNodeOffline' }
    target_match_re:
      alertname: 'ClusterResource.*|ZFSPool.*|NFSServer.*|SMBServer.*'
    equal: ['instance']

  # When quorum is lost, suppress resource transition and failover noise
  - source_match: { alertname: 'ClusterQuorumLost' }
    target_match_re:
      alertname: 'ClusterResource.*|ResourceMigration.*|Pacemaker.*|STONITH.*'
    equal: []
```

## References

- [NTFY Documentation](https://docs.ntfy.sh/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [ntfy-alertmanager Bridge](https://github.com/xenrox/ntfy-alertmanager)
- [Uptime Kuma](https://github.com/louislam/uptime-kuma)
