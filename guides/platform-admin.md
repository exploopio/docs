---
layout: default
title: Platform Administration
parent: Platform Guides
nav_order: 20
---
# Platform Administration Guide

This guide covers how to set up and manage the Rediver platform, including bootstrapping admin credentials, managing platform agents, and using the admin CLI.

---

## Overview

The Rediver platform has two types of administration:

| Type | Purpose | Tools |
|------|---------|-------|
| **Tenant Admin** | Manage assets, scans, users within a tenant | Web UI, Tenant API |
| **Platform Admin** | Manage platform infrastructure, agents, quotas | Admin CLI, Admin API |

This guide focuses on **Platform Administration**.

---

## Initial Setup (Bootstrap)

When deploying Rediver for the first time, you need to create the first admin user. This is done using the `bootstrap-admin` tool which connects directly to the database.

### Prerequisites

1. Database migrations have been applied
2. API container running (or direct database access)

### Bootstrap First Admin

The `bootstrap-admin` tool is included in the API Docker image. Since it needs database access, you run it from within the API container.

#### Option 1: Using docker-compose exec (Recommended)

```bash
# Run bootstrap-admin from within the API container
# The container already has DB_HOST, DB_USER, etc. configured
docker-compose exec app ./bootstrap-admin \
  -email "admin@yourcompany.com" \
  -role "super_admin"

# Note: The service name is 'app' in docker-compose.prod.yml
# Adjust if your service has a different name
```

#### Option 2: Using docker exec

```bash
# If using plain docker (not compose)
docker exec -it rediver-app ./bootstrap-admin \
  -email "admin@yourcompany.com" \
  -role "super_admin"
```

#### Option 3: Using standalone admin-cli image

```bash
# For environments where API container isn't accessible
# Must have network access to the database
docker run --rm \
  --network your-network \
  --entrypoint /usr/local/bin/bootstrap-admin \
  -e DATABASE_URL="postgres://user:pass@db:5432/rediver?sslmode=disable" \
  -e ADMIN_EMAIL="admin@yourcompany.com" \
  rediverio/admin-cli:latest
```

#### Option 4: Kubernetes Job (Recommended for K8s)

```yaml
# bootstrap-admin-job.yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-admin-config
  namespace: rediver
type: Opaque
stringData:
  ADMIN_EMAIL: "admin@yourcompany.com"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: bootstrap-admin
  namespace: rediver
spec:
  ttlSecondsAfterFinished: 300  # Clean up after 5 minutes
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: bootstrap-admin
          image: rediverio/api:latest
          command: ["./bootstrap-admin"]
          args: ["-role", "super_admin"]
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: rediver-db-credentials
                  key: host
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: rediver-db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: rediver-db-credentials
                  key: password
            - name: DB_NAME
              value: rediver
            - name: DB_SSLMODE
              value: require
            - name: ADMIN_EMAIL
              valueFrom:
                secretKeyRef:
                  name: bootstrap-admin-config
                  key: ADMIN_EMAIL
```

```bash
# Apply the job
kubectl apply -f bootstrap-admin-job.yaml

# Watch for completion and get the API key from logs
kubectl logs -f job/bootstrap-admin -n rediver

# Clean up (or wait for ttlSecondsAfterFinished)
kubectl delete job bootstrap-admin -n rediver
```

#### Option 5: kubectl exec (Quick method for K8s)

```bash
# If API pod is already running
kubectl exec -it deploy/rediver-api -n rediver -- \
  ./bootstrap-admin -email "admin@yourcompany.com" -role "super_admin"
```

#### Option 6: Using Binary (development)

```bash
# Only for local development with direct DB access
./bootstrap-admin \
  -db "postgres://user:pass@localhost:5432/rediver?sslmode=disable" \
  -email "admin@yourcompany.com" \
  -role "super_admin"
```

### Bootstrap Output

```
=== Bootstrap Admin Created ===
  ID:    550e8400-e29b-41d4-a716-446655440000
  Email: admin@yourcompany.com
  Role:  super_admin

API Key (save this, it won't be shown again):
  radm_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6

Configure the CLI:
  export REDIVER_API_KEY=radm_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
  export REDIVER_API_URL=https://your-api-url

  # Or save to config file:
  rediver-admin config set-context prod --api-url=https://your-api-url --api-key=radm_...
  rediver-admin config use-context prod

Test the connection:
  rediver-admin cluster-info
```

> **Important**: Save the API key immediately! It cannot be retrieved later.

### Admin Roles

| Role | Permissions |
|------|-------------|
| `super_admin` | Full access: manage admins, agents, tokens, all operations |
| `ops_admin` | Manage agents and tokens, view queue stats |
| `viewer` | Read-only access to platform status |

---

## Admin CLI (rediver-admin)

The `rediver-admin` CLI provides kubectl-style commands for platform management.

### Installation

#### Binary Installation

```bash
# Download from releases
curl -LO https://github.com/rediverio/api/releases/latest/download/rediver-admin-linux-amd64
chmod +x rediver-admin-linux-amd64
sudo mv rediver-admin-linux-amd64 /usr/local/bin/rediver-admin

# Verify installation
rediver-admin version
```

#### Using Docker

```bash
# Create alias for convenience
alias rediver-admin='docker run --rm -it \
  -e REDIVER_API_URL=$REDIVER_API_URL \
  -e REDIVER_API_KEY=$REDIVER_API_KEY \
  -v ~/.rediver:/root/.rediver \
  rediverio/admin-cli:latest'

# Use normally
rediver-admin get agents
```

### Configuration

The CLI supports three configuration methods (in priority order):

#### 1. Command Line Flags (Highest Priority)

```bash
rediver-admin --api-url=https://api.rediver.io --api-key=radm_xxx get agents
```

#### 2. Environment Variables

```bash
export REDIVER_API_URL=https://api.rediver.io
export REDIVER_API_KEY=radm_a1b2c3d4e5f6...
rediver-admin get agents
```

#### 3. Config File (~/.rediver/config.yaml)

```bash
# Create context
rediver-admin config set-context prod \
  --api-url=https://api.rediver.io \
  --api-key=radm_a1b2c3d4e5f6...

# Or use key file for security
echo "radm_a1b2c3d4e5f6..." > ~/.rediver/prod-key
chmod 600 ~/.rediver/prod-key
rediver-admin config set-context prod \
  --api-url=https://api.rediver.io \
  --api-key-file=~/.rediver/prod-key

# Switch contexts
rediver-admin config use-context prod

# List contexts
rediver-admin config get-contexts
```

**Config file format:**

```yaml
# ~/.rediver/config.yaml
apiVersion: admin.rediver.io/v1
kind: Config
current-context: prod

contexts:
  - name: prod
    context:
      api-url: https://api.rediver.io
      api-key-file: ~/.rediver/prod-key

  - name: staging
    context:
      api-url: https://api.staging.rediver.io
      api-key-file: ~/.rediver/staging-key

  - name: local
    context:
      api-url: http://localhost:8080
      api-key: radm_local_dev_key
```

### Command Reference

#### Global Flags

All commands support these global flags:

| Flag | Short | Description |
|------|-------|-------------|
| `--output` | `-o` | Output format: `json`, `yaml`, `wide`, `name` |
| `--api-url` | | Override API URL from config |
| `--api-key` | | Override API key from config |
| `--context` | `-c` | Use specific context from config |
| `--verbose` | `-v` | Enable verbose output |

### Common Commands

#### Cluster Overview

```bash
# Get platform status
rediver-admin cluster-info

# Output:
# Platform Cluster Info
# =====================
#
# Agents:
#   Total:    5
#   Online:   4
#   Offline:  1
#   Drained:  0
#
# Capacity:
#   Total:    50 jobs
#   Used:     12 jobs (24.0%)
#   Free:     38 jobs
#
# Job Queue:
#   Pending:    3
#   Running:    12
#   Completed:  156 (24h)
#   Failed:     2 (24h)
```

#### Agent Management

```bash
# List agents
rediver-admin get agents
rediver-admin get agents -o wide
rediver-admin get agents -o json

# Get specific agent
rediver-admin describe agent agent-us-east-1

# Create agent (for manual registration)
rediver-admin create agent \
  --name=agent-us-east-1 \
  --region=us-east-1 \
  --capabilities=sast,sca,secrets \
  --max-jobs=10

# Maintenance operations
rediver-admin drain agent agent-us-east-1      # Stop accepting new jobs
rediver-admin uncordon agent agent-us-east-1   # Resume operations
rediver-admin delete agent agent-us-east-1     # Remove agent
```

#### Bootstrap Token Management

```bash
# List tokens
rediver-admin get tokens
rediver-admin get tokens -o wide

# Create token for agent registration
rediver-admin create token --max-uses=5 --expires=24h

# Output:
# token/tok-abc123 created
#
# Token Details:
#   ID:        tok-abc123
#   Max Uses:  5
#   Expires:   2026-01-27 15:04:05
#
# Bootstrap Token (save this, it won't be shown again):
#   abc123.xxxxxxxxxxxxxxxx
#
# Use this token to register a platform agent:
#   ./agent -platform -bootstrap-token=abc123.xxxxxxxxxxxxxxxx -api-url=https://api.rediver.io

# Revoke token
rediver-admin revoke token tok-abc123 --reason="No longer needed"
```

#### Admin User Management (super_admin only)

```bash
# List admins
rediver-admin get admins

# Create new admin
rediver-admin create admin --email=ops@company.com --role=ops_admin

# Output includes the new admin's API key
```

#### Job Queue Monitoring

```bash
# List jobs
rediver-admin get jobs
rediver-admin get jobs --status=pending
rediver-admin get jobs --status=running
rediver-admin get jobs -o wide

# Job details
rediver-admin describe job job-xyz123

# View job logs
rediver-admin logs job job-xyz123

# Follow job logs in real-time (updates every 2s)
rediver-admin logs job job-xyz123 -f

# Show last N lines
rediver-admin logs job job-xyz123 --tail=100
```

#### Declarative Configuration (apply)

Similar to `kubectl apply`, you can create resources from YAML files:

```bash
# Apply from file
rediver-admin apply -f agent.yaml

# Apply from stdin
cat agent.yaml | rediver-admin apply -f -
```

**Agent manifest example:**

```yaml
# agent.yaml
apiVersion: admin.rediver.io/v1
kind: Agent
metadata:
  name: agent-us-east-1
  labels:
    environment: production
spec:
  region: us-east-1
  capabilities:
    - sast
    - sca
    - secrets
  maxJobs: 10
```

**Bootstrap Token manifest example:**

```yaml
# token.yaml
apiVersion: admin.rediver.io/v1
kind: Token
metadata:
  name: bootstrap-token-prod
spec:
  maxUses: 5
  expiresIn: 24h
```

**Admin manifest example:**

```yaml
# admin.yaml
apiVersion: admin.rediver.io/v1
kind: Admin
metadata:
  name: ops-team
spec:
  email: ops@company.com
  role: ops_admin
```

#### Delete Resources

```bash
# Delete an agent (with confirmation prompt)
rediver-admin delete agent agent-us-east-1

# Force delete without confirmation
rediver-admin delete agent agent-us-east-1 --force

# Delete a token
rediver-admin delete token tok-abc123 --force
```

### Output Formats

```bash
# Default table format
rediver-admin get agents

# Wide table with more columns
rediver-admin get agents -o wide

# JSON output (for scripting)
rediver-admin get agents -o json

# YAML output
rediver-admin get agents -o yaml

# Just names (for scripting)
rediver-admin get agents -o name
```

### Watch Mode

```bash
# Real-time updates (refreshes every 2s)
rediver-admin get agents -w
```

---

## Platform Agent Setup

Platform agents are Rediver-managed agents that can be used by multiple tenants.

### Registration Methods

#### Method 1: Bootstrap Token (Recommended)

```bash
# 1. Create bootstrap token (on admin machine)
rediver-admin create token --max-uses=1 --expires=1h

# 2. Start agent with token (on agent machine)
./agent -platform \
  -bootstrap-token=abc123.xxxxxxxxxxxxxxxx \
  -api-url=https://api.rediver.io \
  -region=us-east-1 \
  -capabilities=sast,sca,secrets

# Agent will:
# 1. Register using bootstrap token
# 2. Receive permanent API key
# 3. Start polling for jobs
```

#### Method 2: Pre-created Agent

```bash
# 1. Create agent record (on admin machine)
rediver-admin create agent \
  --name=agent-us-east-1 \
  --region=us-east-1 \
  --capabilities=sast,sca

# 2. Get the agent's API key from output
# 3. Start agent with key (on agent machine)
./agent -platform \
  -api-key=ragent_xxxxx \
  -api-url=https://api.rediver.io
```

### Docker Deployment

```bash
# Using bootstrap token
docker run -d \
  --name rediver-platform-agent \
  --restart unless-stopped \
  -e API_URL=https://api.rediver.io \
  -e BOOTSTRAP_TOKEN=abc123.xxxxxxxxxxxxxxxx \
  -v agent-data:/home/rediver/.rediver \
  rediverio/agent:platform

# Using pre-assigned API key
docker run -d \
  --name rediver-platform-agent \
  --restart unless-stopped \
  -e API_URL=https://api.rediver.io \
  -e API_KEY=ragent_xxxxx \
  rediverio/agent:platform
```

### Kubernetes Deployment

#### Option A: Using Pre-assigned API Key (Recommended for Production)

```bash
# 1. Create agent and get API key
rediver-admin create agent --name=agent-k8s-pool --region=us-east-1 --capabilities=sast,sca,secrets

# 2. Create secret with the API key
kubectl create secret generic rediver-agent-credentials \
  --from-literal=api-key=ragent_xxxxx \
  -n rediver
```

```yaml
# platform-agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rediver-platform-agent
  namespace: rediver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rediver-platform-agent
  template:
    metadata:
      labels:
        app: rediver-platform-agent
    spec:
      containers:
        - name: agent
          image: rediverio/agent:platform
          env:
            - name: API_URL
              value: "https://api.rediver.io"
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: rediver-agent-credentials
                  key: api-key
            - name: REGION
              value: "us-east-1"
            - name: CAPABILITIES
              value: "sast,sca,secrets"
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          volumeMounts:
            - name: agent-data
              mountPath: /home/rediver/.rediver
      volumes:
        - name: agent-data
          emptyDir: {}
```

#### Option B: Using Bootstrap Token (Auto-registration)

For dynamic scaling where each pod registers independently:

```bash
# 1. Create bootstrap token with enough uses for your replicas
rediver-admin create token --max-uses=10 --expires=24h

# 2. Create secret with the bootstrap token
kubectl create secret generic rediver-bootstrap-token \
  --from-literal=token=abc123.xxxxxxxxxxxxxxxx \
  -n rediver
```

```yaml
# platform-agent-bootstrap.yaml
apiVersion: apps/v1
kind: StatefulSet  # StatefulSet ensures unique agent names
metadata:
  name: rediver-platform-agent
  namespace: rediver
spec:
  serviceName: rediver-platform-agent
  replicas: 3
  selector:
    matchLabels:
      app: rediver-platform-agent
  template:
    metadata:
      labels:
        app: rediver-platform-agent
    spec:
      containers:
        - name: agent
          image: rediverio/agent:platform
          env:
            - name: API_URL
              value: "https://api.rediver.io"
            - name: BOOTSTRAP_TOKEN
              valueFrom:
                secretKeyRef:
                  name: rediver-bootstrap-token
                  key: token
            - name: REGION
              value: "us-east-1"
            - name: CAPABILITIES
              value: "sast,sca,secrets"
            # Use pod name as agent name for uniqueness
            - name: AGENT_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          volumeMounts:
            - name: agent-data
              mountPath: /home/rediver/.rediver
  volumeClaimTemplates:
    - metadata:
        name: agent-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

#### Option C: Using Helm (Recommended)

Helm chart simplifies deployment with sensible defaults and easy configuration.

**Add Repository:**

```bash
helm repo add rediver https://charts.rediver.io
helm repo update
```

**Method 1: Bootstrap Token (Auto-registration)**

Best for dynamic scaling where each pod registers as a unique agent.

```bash
# 1. Create bootstrap token
rediver-admin create token --max-uses=10 --expires=24h

# 2. Install with bootstrap token
helm install platform-agent rediver/platform-agent \
  --namespace rediver \
  --create-namespace \
  --set apiUrl=https://api.rediver.io \
  --set bootstrapToken=abc123.xxxxxxxxxxxxxxxx \
  --set replicaCount=3 \
  --set agent.region=us-east-1 \
  --set agent.capabilities=sast,sca,secrets
```

**Method 2: Pre-assigned API Key (Shared Identity)**

Best for production with fixed replicas sharing the same agent identity.

```bash
# 1. Create agent
rediver-admin create agent --name=k8s-pool --region=us-east-1 --capabilities=sast,sca,secrets

# 2. Install with API key (uses Deployment instead of StatefulSet)
helm install platform-agent rediver/platform-agent \
  --namespace rediver \
  --create-namespace \
  --set apiUrl=https://api.rediver.io \
  --set apiKey=ragent_xxxxx \
  --set useStatefulSet=false \
  --set replicaCount=3
```

**Method 3: Existing Secret**

Use credentials stored in an existing Kubernetes secret.

```bash
# 1. Create secret with credentials
kubectl create secret generic rediver-agent-creds \
  --from-literal=api-key=ragent_xxxxx \
  -n rediver

# 2. Install using existing secret
helm install platform-agent rediver/platform-agent \
  --namespace rediver \
  --set apiUrl=https://api.rediver.io \
  --set existingSecret.enabled=true \
  --set existingSecret.name=rediver-agent-creds \
  --set useStatefulSet=false
```

**Key Configuration Options:**

| Parameter | Description | Default |
|-----------|-------------|---------|
| `apiUrl` | Rediver API URL (required) | `""` |
| `bootstrapToken` | Bootstrap token for auto-registration | `""` |
| `apiKey` | Pre-assigned API key | `""` |
| `replicaCount` | Number of agent replicas | `3` |
| `useStatefulSet` | Use StatefulSet for unique agent names | `true` |
| `agent.region` | Agent region identifier | `""` |
| `agent.capabilities` | Comma-separated capabilities | `"sast,sca,secrets"` |
| `agent.maxJobs` | Max concurrent jobs per agent | `5` |
| `persistence.enabled` | Enable persistent storage (StatefulSet) | `true` |
| `persistence.size` | Storage size | `5Gi` |

**Upgrade & Maintenance:**

```bash
# Scale up
helm upgrade platform-agent rediver/platform-agent \
  --namespace rediver \
  --reuse-values \
  --set replicaCount=5

# Upgrade chart version
helm upgrade platform-agent rediver/platform-agent \
  --namespace rediver \
  --reuse-values

# Uninstall
helm uninstall platform-agent -n rediver

# If using StatefulSet, also delete PVCs
kubectl delete pvc -l app.kubernetes.io/instance=platform-agent -n rediver
```

For full documentation, see the [Helm chart README](https://github.com/rediverio/charts/tree/main/charts/platform-agent).

---

## Deployment Architecture

### Single-Server Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                     Single Server                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   API        │  │   Redis      │  │  PostgreSQL  │      │
│  │   :8080      │  │   :6379      │  │   :5432      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │   Agent 1    │  │   Agent 2    │                         │
│  │   (platform) │  │   (platform) │                         │
│  └──────────────┘  └──────────────┘                         │
│                                                              │
│  Admin CLI connects via: http://localhost:8080              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Multi-Server Production

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Production Architecture                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Operator Workstation              Load Balancer                          │
│  ┌─────────────────┐              ┌─────────────────┐                    │
│  │ rediver-admin   │──HTTPS:443──▶│   nginx/ALB     │                    │
│  │ ~/.rediver/     │              │   :443          │                    │
│  │   config.yaml   │              └────────┬────────┘                    │
│  └─────────────────┘                       │                              │
│                                            ▼                              │
│                              ┌─────────────────────────┐                 │
│                              │     API Cluster         │                 │
│                              │  ┌─────┐ ┌─────┐ ┌─────┐│                 │
│                              │  │ API │ │ API │ │ API ││                 │
│                              │  │  1  │ │  2  │ │  3  ││                 │
│                              │  └─────┘ └─────┘ └─────┘│                 │
│                              └────────────┬────────────┘                 │
│                                           │                              │
│                    ┌──────────────────────┴──────────────────────┐      │
│                    ▼                                              ▼      │
│          ┌─────────────────┐                          ┌─────────────────┐│
│          │   PostgreSQL    │                          │     Redis       ││
│          │   (Primary)     │                          │   (Cluster)     ││
│          └─────────────────┘                          └─────────────────┘│
│                                                                           │
│  Region: us-east-1                        Region: eu-west-1              │
│  ┌──────────────────────┐                ┌──────────────────────┐        │
│  │  Platform Agents     │                │  Platform Agents     │        │
│  │  ┌─────┐ ┌─────┐    │                │  ┌─────┐ ┌─────┐    │        │
│  │  │ Ag1 │ │ Ag2 │    │──Long Poll───▶ │  │ Ag1 │ │ Ag2 │    │        │
│  │  └─────┘ └─────┘    │                │  └─────┘ └─────┘    │        │
│  └──────────────────────┘                └──────────────────────┘        │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Security Best Practices

### API Key Management

1. **Never commit API keys to version control**
2. **Use key files instead of inline keys in config**
   ```bash
   # Good: key in separate file with restricted permissions
   echo "radm_xxx" > ~/.rediver/prod-key
   chmod 600 ~/.rediver/prod-key

   # Bad: key directly in config.yaml
   ```

3. **Rotate keys periodically**
   ```bash
   rediver-admin rotate-key admin <admin-id>
   ```

4. **Use least-privilege roles**
   - `viewer` for monitoring dashboards
   - `ops_admin` for day-to-day operations
   - `super_admin` only for initial setup and emergency access

### Bootstrap Token Security

1. **Use short expiration times** (1h - 24h)
2. **Limit max uses** (1-5 for production)
3. **Revoke unused tokens immediately**
4. **Monitor token usage in audit logs**

### Network Security

1. **Use TLS for all connections**
2. **Restrict admin API to internal network**
3. **Use VPN for remote admin access**
4. **Enable IP allowlisting if possible**

---

## Troubleshooting

### CLI Cannot Connect

```bash
# Check configuration
rediver-admin config view

# Test with verbose output
rediver-admin --verbose get agents

# Verify network connectivity
curl -v https://api.rediver.io/health
```

### Agent Not Registering

```bash
# Check token validity
rediver-admin get tokens

# Check agent logs
docker logs rediver-platform-agent

# Verify bootstrap token format: xxxxxx.yyyyyyyyyyyyyyyy
```

### Permission Denied

```bash
# Verify your role
rediver-admin get admins | grep your-email

# Check if operation requires super_admin
# Operations like "create admin" require super_admin role
```

---

## Quick Reference

### Command Cheat Sheet

```bash
# === CLUSTER INFO ===
rediver-admin cluster-info              # Platform overview
rediver-admin version                   # CLI version

# === AGENTS ===
rediver-admin get agents                # List all agents
rediver-admin get agents -o wide        # Detailed list
rediver-admin get agents -w             # Watch mode (auto-refresh)
rediver-admin describe agent <name>     # Agent details
rediver-admin create agent --name=<n> --region=<r> --capabilities=sast,sca
rediver-admin drain agent <name>        # Stop accepting new jobs
rediver-admin uncordon agent <name>     # Resume operations
rediver-admin delete agent <name>       # Remove agent

# === TOKENS ===
rediver-admin get tokens                # List bootstrap tokens
rediver-admin create token --max-uses=5 --expires=24h
rediver-admin describe token <id>       # Token details
rediver-admin revoke token <id> --reason="..."
rediver-admin delete token <id>         # Remove token

# === JOBS ===
rediver-admin get jobs                  # List platform jobs
rediver-admin get jobs --status=pending # Filter by status
rediver-admin describe job <id>         # Job details
rediver-admin logs job <id>             # View job logs
rediver-admin logs job <id> -f          # Follow logs in real-time

# === ADMINS ===
rediver-admin get admins                # List admin users
rediver-admin create admin --email=<e> --role=ops_admin

# === CONFIG ===
rediver-admin config set-context <name> --api-url=<url> --api-key=<key>
rediver-admin config use-context <name> # Switch context
rediver-admin config current-context    # Show current
rediver-admin config get-contexts       # List all contexts
rediver-admin config view               # Show full config

# === DECLARATIVE ===
rediver-admin apply -f agent.yaml       # Apply from file
cat manifest.yaml | rediver-admin apply -f -  # Apply from stdin
```

### Resource Aliases

| Resource | Aliases |
|----------|---------|
| `agents` | `agent`, `ag` |
| `tokens` | `token`, `tok` |
| `jobs` | `job` |
| `admins` | `admin` |

### Status Values

**Agent Status:**
- `online` - Agent is connected and accepting jobs
- `offline` - Agent is disconnected
- `draining` - Agent finishing current jobs, not accepting new ones

**Job Status:**
- `pending` - Waiting in queue
- `running` - Being processed by an agent
- `completed` - Successfully finished
- `failed` - Finished with errors
- `cancelled` - Cancelled by user/admin
- `expired` - Timed out

---

## Related Documentation

- [Authentication Guide](./authentication.md) - Tenant API key management
- [Running Agents](./running-agents.md) - Tenant agent setup
- [Docker Deployment](./docker-deployment.md) - Container deployment
- [API Reference](../backend/api-reference.md) - Full API documentation
