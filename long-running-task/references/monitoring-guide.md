# Monitoring and Alerting Guide

## Table of Contents

1. [Key Metrics Overview](#key-metrics-overview)
2. [Storage Health Monitoring](#storage-health-monitoring)
3. [Task Execution Metrics](#task-execution-metrics)
4. [Resource Isolation Metrics](#resource-isolation-metrics)
5. [Alert Configuration](#alert-configuration)
6. [Dashboard Setup](#dashboard-setup)

## Key Metrics Overview

### Metric Categories

| Category | Purpose | Criticality |
|----------|---------|-------------|
| Storage Health | Ensure checkpoint data persistence | High |
| Task Execution | Track long-running task behavior | Medium |
| Resource Isolation | Monitor parallel agent conflicts | Medium |
| Performance | Measure system responsiveness | Low |

## Storage Health Monitoring

### Core Metrics

#### Storage Usage

```
Metric: checkpoint_storage_usage_bytes
Type: Gauge
Labels: [storage_backend, retention_tier]
Description: Total bytes used for checkpoint storage
```

**Thresholds**:
- Warning: 80% of allocated storage
- Critical: 90% of allocated storage

#### Creation Success Rate

```
Metric: checkpoint_creation_success_rate
Type: Counter (success/total)
Calculation: success_count / total_attempts over 5m window
Description: Percentage of successful checkpoint creations
```

**Thresholds**:
- Target: > 99.9%
- Warning: < 99.5%
- Critical: < 99%

#### Recovery Latency

```
Metric: checkpoint_recovery_duration_seconds
Type: Histogram
Buckets: [0.1, 0.5, 1, 2, 5, 10, 30]
Labels: [recovery_type]
Description: Time to restore from checkpoint
```

**Thresholds**:
- P95: < 2 seconds
- P99: < 5 seconds

### Storage Health Checklist

- [ ] Monitor storage usage trend (predict capacity exhaustion)
- [ ] Alert on creation failures immediately
- [ ] Track recovery latency distribution
- [ ] Monitor garbage collection effectiveness

## Task Execution Metrics

### Core Metrics

#### Task Duration Distribution

```
Metric: task_duration_seconds
Type: Histogram
Buckets: [60, 300, 900, 1800, 3600, 7200, 14400]
Labels: [task_type, complexity]
Description: Duration of completed tasks
```

#### Checkpoint Frequency

```
Metric: checkpoints_per_task
Type: Histogram
Buckets: [1, 5, 10, 20, 50, 100]
Labels: [task_type]
Description: Number of checkpoints created per task
```

#### Recovery Operations

```
Metric: recovery_operations_total
Type: Counter
Labels: [recovery_type, reason]
Description: Count of recovery operations performed
```

### Recovery Reason Classification

| Reason Code | Description | Action |
|-------------|-------------|--------|
| `user_request` | User initiated recovery | None |
| `session_disconnect` | Network/session interruption | Investigate if frequent |
| `error_recovery` | Task execution error | Review error patterns |
| `resource_conflict` | Port/file lock conflict | Review isolation config |

### Task Execution Checklist

- [ ] Track task duration by type and complexity
- [ ] Monitor checkpoint frequency patterns
- [ ] Classify recovery reasons for insights
- [ ] Identify tasks with abnormal checkpoint ratios

## Resource Isolation Metrics

### Core Metrics

#### Sub-Agent Memory

```
Metric: subagent_memory_bytes
Type: Gauge
Labels: [agent_id, task_id]
Description: Peak memory usage of sub-agents
```

**Thresholds**:
- Warning: 80% of allocated limit
- Critical: 95% of allocated limit

#### File Lock Conflicts

```
Metric: file_lock_conflicts_total
Type: Counter
Labels: [file_path, conflict_type]
Description: Count of file lock conflicts between agents
```

#### Port Conflicts

```
Metric: port_conflicts_total
Type: Counter
Labels: [port_number, agent_id]
Description: Count of port allocation conflicts
```

### Resource Isolation Checklist

- [ ] Monitor per-agent memory consumption
- [ ] Track file lock contention hotspots
- [ ] Identify frequently conflicting ports
- [ ] Set up isolation boundary alerts

## Alert Configuration

### Alert Priority Levels

| Level | Response Time | Notification |
|-------|---------------|--------------|
| P1 Critical | Immediate | Page + SMS |
| P2 Warning | < 15 minutes | Slack/Teams |
| P3 Info | < 1 hour | Email digest |

### Sample Alert Rules (Prometheus)

```yaml
groups:
  - name: checkpoint_storage
    rules:
      - alert: CheckpointStorageNearCapacity
        expr: checkpoint_storage_usage_bytes / checkpoint_storage_limit_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Checkpoint storage at {{ $value | humanizePercentage }}"
          description: "Storage backend {{ $labels.storage_backend }} is approaching capacity"

      - alert: CheckpointStorageCritical
        expr: checkpoint_storage_usage_bytes / checkpoint_storage_limit_bytes > 0.9
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Checkpoint storage critically full"
          description: "Immediate action required to prevent checkpoint failures"

  - name: checkpoint_operations
    rules:
      - alert: CheckpointCreationFailureRate
        expr: rate(checkpoint_creation_failures_total[5m]) / rate(checkpoint_creation_total[5m]) > 0.01
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Checkpoint creation failure rate elevated"
          description: "{{ $value | humanizePercentage }} of checkpoint creations failing"

      - alert: CheckpointRecoverySlow
        expr: histogram_quantile(0.95, rate(checkpoint_recovery_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Checkpoint recovery P95 latency high"
          description: "P95 recovery time is {{ $value }}s (target: <2s)"
```

### Alert Response Playbook

#### Storage Capacity Alert

1. Check current usage trend
2. Identify large checkpoint sources
3. Consider:
   - Increasing storage allocation
   - Adjusting retention policy
   - Enabling more aggressive compression

#### Creation Failure Alert

1. Check storage backend health
2. Verify network connectivity
3. Review recent changes to checkpoint configuration
4. Check for permission issues

#### Recovery Latency Alert

1. Check storage backend performance
2. Verify network latency
3. Consider local cache expansion
4. Review checkpoint size trends

## Dashboard Setup

### Grafana Dashboard Panels

#### Panel 1: Storage Overview

```
Type: Stat + Gauge
Metrics:
  - Current usage (gauge)
  - Usage trend (sparkline)
  - Days until full (projection)
```

#### Panel 2: Operation Health

```
Type: Time series
Metrics:
  - Creation success rate (line)
  - Recovery latency P50/P95/P99 (lines)
  - Recovery operations by type (stacked area)
```

#### Panel 3: Task Execution

```
Type: Heatmap + Bar chart
Metrics:
  - Task duration distribution (heatmap)
  - Checkpoints per task (bar chart by task type)
```

#### Panel 4: Resource Conflicts

```
Type: Table + Pie chart
Metrics:
  - Top 10 conflicting files (table)
  - Conflict type distribution (pie)
  - Memory usage by agent (table)
```

### Sample Dashboard JSON Structure

```json
{
  "dashboard": {
    "title": "Claude Code Checkpoint Monitoring",
    "panels": [
      {
        "title": "Storage Health",
        "type": "gauge",
        "targets": [
          {
            "expr": "checkpoint_storage_usage_bytes / checkpoint_storage_limit_bytes",
            "legendFormat": "Usage"
          }
        ],
        "thresholds": {
          "mode": "absolute",
          "steps": [
            {"color": "green", "value": 0},
            {"color": "yellow", "value": 0.8},
            {"color": "red", "value": 0.9}
          ]
        }
      }
    ]
  }
}
```

## Integration with Existing Systems

### Prometheus Integration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'claude-code-checkpoints'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'
```

### Datadog Integration

```yaml
# datadog.yaml
checks:
  - name: claude_code_checkpoints
    instances:
      - host: localhost
        port: 9090
        metrics:
          - checkpoint_*
```

### Custom Webhook Integration

```yaml
# webhook config for alerts
webhooks:
  - name: slack
    url: https://hooks.slack.com/services/XXX
    triggers:
      - alert: CheckpointStorageCritical
        template: |
          {
            "text": "Critical: Checkpoint storage at capacity",
            "attachments": [{
              "color": "danger",
              "fields": [
                {"title": "Usage", "value": "{{ .Value }}", "short": true},
                {"title": "Backend", "value": "{{ .Labels.storage_backend }}", "short": true}
              ]
            }]
          }
```
