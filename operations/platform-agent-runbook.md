---
layout: default
title: Platform Agent Operations Runbook
parent: Operations
nav_order: 14
---
{% raw %}

# Platform Agent Operations Runbook

Complete operations guide for deploying, monitoring, and troubleshooting Exploop Platform Agents.

**Last Updated:** 2026-01-26

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Quick Diagnostics](#quick-diagnostics)
- [Deployment](#deployment)
- [Configuration Reference](#configuration-reference)
- [Monitoring & Alerting](#monitoring--alerting)
- [Troubleshooting](#troubleshooting)
- [Common Operations](#common-operations)
- [Scaling Guide](#scaling-guide)
- [Incident Response](#incident-response)
- [Maintenance Procedures](#maintenance-procedures)

---

## Overview

### What is a Platform Agent?

Platform Agents are Rediver-managed scan agents that provide shared scanning capacity to tenants. Unlike tenant-deployed agents, platform agents are:

- **Centrally managed** by platform administrators
- **Multi-tenant** - serve multiple tenants with isolation
- **Auto-scaling** capable based on queue depth
- **Monitored** via the Admin UI and CLI

### Key Components

| Component | Purpose |
|-----------|---------|
| **Agent Binary** | Executes scan jobs |
| **Lease Manager** | Maintains heartbeat with API |
| **Job Poller** | Long-polls for new jobs |
| **Reporter** | Reports job progress and results |

### Communication Flow

```
                    Platform Agent
                    ┌─────────────────────────────────┐
                    │                                  │
┌─────────┐  Lease  │  ┌─────────────────────────┐   │
│         │ ◄───────┼──┤     Lease Manager       │   │
│         │  (30s)  │  │  PUT /platform/lease    │   │
│         │         │  └─────────────────────────┘   │
│         │         │                                  │
│   API   │  Poll   │  ┌─────────────────────────┐   │
│ Server  │ ◄───────┼──┤     Job Poller          │   │
│         │ (long)  │  │  POST /platform/poll    │   │
│         │         │  └─────────────────────────┘   │
│         │         │                                  │
│         │ Report  │  ┌─────────────────────────┐   │
│         │ ◄───────┼──┤     Job Reporter        │   │
│         │         │  │  POST /platform/jobs/*  │   │
└─────────┘         │  └─────────────────────────┘   │
                    │                                  │
                    └─────────────────────────────────┘
```

---

## Quick Diagnostics

### Health Check Commands

```bash
# Check agent container status
docker ps | grep platform-agent

# Check agent logs (last 100 lines)
docker logs --tail 100 exploop-platform-agent

# Check agent metrics
curl -s http://localhost:9100/metrics | grep.exploop_agent

# Check API connectivity
curl -s https://api.exploop.io/health

# List all agents via CLI
exploop-admin get agents

# Get specific agent status
exploop-admin describe agent <agent-name>

# Get cluster overview
exploop-admin cluster-info
```

### Quick Status Verification

```bash
# Verify agent is online
exploop-admin get agents | grep <agent-name>

# Expected output:
# NAME              STATUS   REGION      JOBS    LAST SEEN
# agent-us-east-1   online   us-east-1   3/10    12s

# If status is 'offline', check:
# 1. Agent container running
# 2. Network connectivity to API
# 3. API key/credentials valid
```

---

## Deployment

### Prerequisites

- Docker 20.10+ or Kubernetes 1.21+
- Network access to Rediver API (port 443)
- Bootstrap token or pre-assigned API key
- Minimum 2 CPU cores, 4GB RAM per agent

### Docker Deployment

#### Using Bootstrap Token (Recommended)

```bash
# 1. Create bootstrap token
exploop-admin create token --max-uses=1 --expires=1h

# 2. Run agent container
docker run -d \
  --name exploop-platform-agent \
  --restart unless-stopped \
  -e API_URL=https://api.exploop.io \
  -e BOOTSTRAP_TOKEN=<token-from-step-1> \
  -e REGION=us-east-1 \
  -e CAPABILITIES=sast,sca,secrets \
  -e MAX_CONCURRENT_JOBS=5 \
  -v agent-data:/home/exploop/.exploop \
  --memory=4g \
  --cpus=2 \
  exploopio/agent:platform

# 3. Verify registration
docker logs exploop-platform-agent | grep "registered"
exploop-admin get agents
```

#### Using Pre-assigned API Key

```bash
# 1. Create agent record
exploop-admin create agent \
  --name=agent-us-east-1 \
  --region=us-east-1 \
  --capabilities=sast,sca,secrets \
  --max-jobs=5

# Save the API key from output!

# 2. Run agent container
docker run -d \
  --name exploop-platform-agent \
  --restart unless-stopped \
  -e API_URL=https://api.exploop.io \
  -e API_KEY=ragent_xxxxx \
  -v agent-data:/home/exploop/.exploop \
  --memory=4g \
  --cpus=2 \
  exploopio/agent:platform
```

### Kubernetes Deployment

See [Platform Admin Guide - Kubernetes Deployment](../guides/platform-admin.md#kubernetes-deployment) for full manifests.

**Quick Helm Install:**

```bash
helm install platform-agent exploop/platform-agent \
  --namespace.exploop \
  --create-namespace \
  --set apiUrl=https://api.exploop.io \
  --set bootstrapToken=<token> \
  --set replicaCount=3 \
  --set agent.region=us-east-1
```

### Verification Checklist

After deployment, verify:

- [ ] Agent appears in `exploop-admin get agents`
- [ ] Status shows `online`
- [ ] Capabilities are correct
- [ ] Jobs counter increments when jobs are assigned
- [ ] Logs show successful heartbeats

---

## Configuration Reference

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `API_URL` | Yes | - | Rediver API URL |
| `API_KEY` | No* | - | Agent API key (if pre-assigned) |
| `BOOTSTRAP_TOKEN` | No* | - | Bootstrap token for registration |
| `REGION` | No | auto | Agent region identifier |
| `CAPABILITIES` | No | sast,sca | Comma-separated scanner capabilities |
| `MAX_CONCURRENT_JOBS` | No | 5 | Maximum parallel jobs |
| `LOG_LEVEL` | No | info | Log verbosity (debug, info, warn, error) |
| `HEARTBEAT_INTERVAL` | No | 30s | Heartbeat frequency |
| `POLL_TIMEOUT` | No | 30s | Long-poll timeout |

*Either `API_KEY` or `BOOTSTRAP_TOKEN` is required.

### Resource Recommendations

| Workload | CPU | Memory | Disk |
|----------|-----|--------|------|
| Light (1-2 concurrent) | 1 core | 2GB | 10GB |
| Medium (3-5 concurrent) | 2 cores | 4GB | 20GB |
| Heavy (6-10 concurrent) | 4 cores | 8GB | 50GB |

### Network Requirements

| Direction | Port | Protocol | Destination |
|-----------|------|----------|-------------|
| Outbound | 443 | HTTPS | API server |
| Outbound | 443 | HTTPS | Git providers (GitHub, GitLab) |
| Outbound | 443 | HTTPS | Package registries (npm, PyPI) |

---

## Monitoring & Alerting

### Key Metrics

| Metric | Type | Alert Threshold |
|--------|------|-----------------|
| .exploop_agent_status` | Gauge | != 1 (online) |
| .exploop_agent_jobs_running` | Gauge | > max_jobs * 0.9 |
| .exploop_agent_jobs_completed_total` | Counter | Rate = 0 for 10m |
| .exploop_agent_jobs_failed_total` | Counter | Rate > 5/min |
| .exploop_agent_heartbeat_latency_seconds` | Histogram | p99 > 5s |
| .exploop_agent_lease_ttl_seconds` | Gauge | < 30s |

### Prometheus Alerts

```yaml
groups:
  - name: platform-agents
    rules:
      - alert: PlatformAgentOffline
        expr:.exploop_agent_status != 1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Platform agent {{ $labels.agent_name }} is offline"

      - alert: PlatformAgentHighLoad
        expr:.exploop_agent_jobs_running /.exploop_agent_max_jobs > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent {{ $labels.agent_name }} at 90%+ capacity"

      - alert: PlatformAgentHighFailureRate
        expr: rate.exploop_agent_jobs_failed_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent {{ $labels.agent_name }} failure rate elevated"

      - alert: PlatformAgentLeaseExpiring
        expr:.exploop_agent_lease_ttl_seconds < 30
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Agent {{ $labels.agent_name }} lease about to expire"
```

### Grafana Dashboard

Key panels to include:

1. **Agent Status Overview**: Table of all agents with status
2. **Jobs per Agent**: Stacked bar chart by agent
3. **Queue Depth**: Line chart of pending jobs
4. **Success Rate**: Gauge showing completion rate
5. **Latency Distribution**: Histogram of job durations
6. **Resource Usage**: CPU/Memory per agent

### Log Patterns to Monitor

```bash
# Successful patterns
"lease renewed successfully"
"job completed"
"agent registered"

# Warning patterns
"lease renewal failed, retrying"
"job timeout, will retry"
"high memory usage"

# Error patterns (alert on these)
"failed to connect to API"
"authentication failed"
"lease expired"
"job failed permanently"
```

---

## Troubleshooting

### Agent Not Starting

**Symptom:** Container starts but agent doesn't appear in admin UI.

**Diagnosis:**
```bash
# Check container logs
docker logs exploop-platform-agent 2>&1 | head -50

# Common errors:
# "invalid bootstrap token" - Token expired or revoked
# "failed to connect" - Network issue
# "authentication failed" - Invalid API key
```

**Solutions:**

| Error | Solution |
|-------|----------|
| Invalid bootstrap token | Create new token with `exploop-admin create token` |
| Connection refused | Check API_URL, verify network connectivity |
| Authentication failed | Verify API key, check if admin disabled agent |
| TLS handshake failed | Ensure API URL uses HTTPS, check certificates |

### Agent Shows Offline

**Symptom:** Agent was online but now shows offline.

**Diagnosis:**
```bash
# Check if agent process is running
docker ps | grep platform-agent

# Check recent logs
docker logs --tail 50 exploop-platform-agent

# Check when agent was last seen
exploop-admin describe agent <name>
```

**Solutions:**

1. **Container crashed**: Restart with `docker restart exploop-platform-agent`
2. **Network partition**: Check connectivity to API
3. **Lease expired**: Agent will auto-recover on restart
4. **API overloaded**: Check API health, scale API if needed

### Jobs Not Being Processed

**Symptom:** Jobs stuck in pending/queued state.

**Diagnosis:**
```bash
# Check queue status
exploop-admin cluster-info

# Check for available agents
exploop-admin get agents | grep online

# Check job details
exploop-admin describe job <job-id>
```

**Solutions:**

| Cause | Solution |
|-------|----------|
| No online agents | Start/restart agents |
| All agents at capacity | Scale up agents or increase max_jobs |
| No matching capabilities | Deploy agents with required capabilities |
| Agent draining | Uncordon agents: `exploop-admin uncordon agent <name>` |

### High Job Failure Rate

**Symptom:** Many jobs failing with errors.

**Diagnosis:**
```bash
# Check failed jobs
exploop-admin get jobs --status=failed

# Get failure details
exploop-admin describe job <job-id>

# Check agent logs during failure
docker logs --since 1h exploop-platform-agent | grep -i error
```

**Common Causes:**

1. **Out of memory**: Increase agent memory limit
2. **Disk full**: Clean up or increase disk space
3. **Git authentication**: Verify repository access tokens
4. **Scanner issues**: Check scanner versions and configs

### Memory/CPU Issues

**Symptom:** Agent consuming excessive resources.

**Diagnosis:**
```bash
# Check container stats
docker stats exploop-platform-agent

# Check system resources
top -p $(pgrep -f .exploop.*agent")
```

**Solutions:**

1. **Reduce MAX_CONCURRENT_JOBS** to limit parallel work
2. **Increase container limits** if system has capacity
3. **Enable swap** for memory spikes (not recommended for production)
4. **Scale horizontally** with more agents instead of more jobs per agent

---

## Common Operations

### Starting/Stopping Agent

```bash
# Stop gracefully (finishes current jobs)
docker stop --time=300 exploop-platform-agent

# Force stop (abandons current jobs)
docker kill exploop-platform-agent

# Start
docker start exploop-platform-agent

# Restart
docker restart exploop-platform-agent
```

### Draining for Maintenance

```bash
# 1. Drain agent (stop accepting new jobs)
exploop-admin drain agent <agent-name>

# 2. Wait for current jobs to complete
watch -n 5 'exploop-admin describe agent <agent-name> | grep Jobs'

# 3. Once jobs = 0, perform maintenance
docker stop exploop-platform-agent
# ... maintenance ...
docker start exploop-platform-agent

# 4. Resume operations
exploop-admin uncordon agent <agent-name>
```

### Updating Agent Version

```bash
# 1. Drain agent
exploop-admin drain agent <agent-name>

# 2. Wait for jobs to complete

# 3. Stop and remove container
docker stop exploop-platform-agent
docker rm exploop-platform-agent

# 4. Pull new image
docker pull exploopio/agent:platform

# 5. Start with same config
docker run -d ... # (same docker run command as before)

# 6. Uncordon
exploop-admin uncordon agent <agent-name>
```

### Rolling Update (Kubernetes)

```bash
# Update image version
kubectl set image deployment/exploop-platform-agent \
  agent=exploopio/agent:platform-v2.0.0 \
  -n exploop

# Watch rollout
kubectl rollout status deployment/exploop-platform-agent -n exploop
```

### Rotating Agent Credentials

```bash
# 1. Create new agent record
exploop-admin create agent --name=agent-us-east-1-new --region=us-east-1

# 2. Update agent container with new key
# (update API_KEY environment variable)

# 3. Drain old agent
exploop-admin drain agent agent-us-east-1

# 4. Delete old agent record
exploop-admin delete agent agent-us-east-1
```

---

## Scaling Guide

### Horizontal Scaling

Add more agent replicas to handle increased load:

**Docker Compose:**
```yaml
services:
  platform-agent:
    image: exploopio/agent:platform
    deploy:
      replicas: 5  # Increase this
```

**Kubernetes:**
```bash
kubectl scale deployment/exploop-platform-agent --replicas=5 -n exploop
```

### Vertical Scaling

Increase resources per agent:

```bash
docker run -d \
  --cpus=4 \          # Increase CPU
  --memory=8g \       # Increase memory
  -e MAX_CONCURRENT_JOBS=10 \  # More parallel jobs
  exploopio/agent:platform
```

### Auto-Scaling (Kubernetes)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: platform-agent-hpa
  namespace: exploop
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: exploop-platform-agent
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name:.exploop_agent_jobs_running
        target:
          type: AverageValue
          averageValue: 4  # Scale when avg > 4 jobs
```

### Capacity Planning

| Queue Depth | Agents Needed | Config |
|-------------|---------------|--------|
| < 50 jobs/day | 1-2 | 5 jobs each |
| 50-200 jobs/day | 3-5 | 5 jobs each |
| 200-500 jobs/day | 5-10 | 10 jobs each |
| > 500 jobs/day | 10+ | Auto-scale |

---

## Incident Response

### Severity Levels

| Level | Description | Response Time |
|-------|-------------|---------------|
| **SEV1** | All agents offline, no job processing | Immediate |
| **SEV2** | High failure rate (>50%), degraded service | 15 minutes |
| **SEV3** | Single agent offline, capacity reduced | 1 hour |
| **SEV4** | Performance degradation, slow jobs | Next business day |

### SEV1: All Agents Offline

1. **Verify**: `exploop-admin get agents` - all showing offline?
2. **Check API**: `curl https://api.exploop.io/health`
3. **If API down**: Escalate to API team
4. **If API up**: Check agent logs for common errors
5. **Recovery**: Restart all agent containers
6. **Verify**: Confirm agents return to online status

### SEV2: High Failure Rate

1. **Identify**: `exploop-admin get jobs --status=failed`
2. **Check patterns**: Same scanner? Same tenant? Same error?
3. **Check logs**: Look for common errors in agent logs
4. **Mitigate**: Drain affected agents if isolated
5. **Fix**: Address root cause (scanner bug, resource exhaustion)
6. **Retry**: `exploop-admin retry job <id>` for failed jobs

### Post-Incident

1. **Document**: What happened, timeline, impact
2. **Root cause**: Determine why it happened
3. **Prevention**: What changes prevent recurrence
4. **Monitor**: Add alerts for early detection

---

## Maintenance Procedures

### Daily Checks

```bash
# Cluster health
exploop-admin cluster-info

# Agent status
exploop-admin get agents

# Failed jobs (last 24h)
exploop-admin get jobs --status=failed
```

### Weekly Tasks

1. Review audit logs for anomalies
2. Check resource utilization trends
3. Review and rotate bootstrap tokens
4. Verify backup procedures

### Monthly Tasks

1. Update agent images to latest version
2. Review and optimize resource allocation
3. Test incident response procedures
4. Review and update alerts/thresholds

### Certificate Renewal

If using custom certificates:

```bash
# 1. Update certificate files
# 2. Restart agents to pick up new certs
docker restart exploop-platform-agent

# Or for Kubernetes:
kubectl rollout restart deployment/exploop-platform-agent -n exploop
```

---

## Related Documentation

- [Platform Admin Guide](../guides/platform-admin.md) - CLI reference
- [Admin UI Guide](../admin-ui/user-guide.md) - Web interface guide
- [Platform Agents Feature](../features/platform-agents.md) - Feature overview
- [Monitoring Guide](./MONITORING.md) - General monitoring setup
{% endraw %}
