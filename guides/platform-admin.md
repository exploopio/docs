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

#### Option 1: Using Setup Makefile (Recommended)

If you're using the `setup/` deployment:

```bash
cd setup

# Staging environment
make bootstrap-admin-staging email=admin@yourcompany.com

# Production environment
make bootstrap-admin-prod email=admin@yourcompany.com

# With specific role
make bootstrap-admin-staging email=ops@yourcompany.com role=ops_admin
```

#### Option 2: Using docker-compose exec

```bash
# Run bootstrap-admin from within the API container
# The container already has DB_HOST, DB_USER, etc. configured
docker-compose exec api ./bootstrap-admin \
  -email "admin@yourcompany.com" \
  -role "super_admin"

# Note: The service name may be 'api' or 'app' depending on your compose file
```

#### Option 3: Using docker exec

```bash
# If using plain docker (not compose)
docker exec -it exploop-api ./bootstrap-admin \
  -email "admin@yourcompany.com" \
  -role "super_admin"
```

#### Option 4: Using standalone admin-cli image

```bash
# For environments where API container isn't accessible
# Must have network access to the database
docker run --rm \
  --network your-network \
  --entrypoint /usr/local/bin/bootstrap-admin \
  -e DATABASE_URL="postgres://user:pass@db:5432/exploop?sslmode=disable" \
  -e ADMIN_EMAIL="admin@yourcompany.com" \
  exploopio/admin-cli:latest
```

#### Option 5: Kubernetes Job (Recommended for K8s)

```yaml
# bootstrap-admin-job.yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-admin-config
  namespace: exploop
type: Opaque
stringData:
  ADMIN_EMAIL: "admin@yourcompany.com"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: bootstrap-admin
  namespace: exploop
spec:
  ttlSecondsAfterFinished: 300  # Clean up after 5 minutes
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: bootstrap-admin
          image: exploopio/api:latest
          command: ["./bootstrap-admin"]
          args: ["-role", "super_admin"]
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: exploop-db-credentials
                  key: host
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: exploop-db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: exploop-db-credentials
                  key: password
            - name: DB_NAME
              value: exploop
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
kubectl logs -f job/bootstrap-admin -n exploop

# Clean up (or wait for ttlSecondsAfterFinished)
kubectl delete job bootstrap-admin -n exploop
```

#### Option 6: kubectl exec (Quick method for K8s)

```bash
# If API pod is already running
kubectl exec -it deploy/exploop-api -n exploop -- \
  ./bootstrap-admin -email "admin@yourcompany.com" -role "super_admin"
```

#### Option 7: Using Binary (development)

```bash
# Only for local development with direct DB access
./bootstrap-admin \
  -db "postgres://user:pass@localhost:5432/exploop?sslmode=disable" \
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
  export EXPLOOP_API_KEY=radm_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
  export EXPLOOP_API_URL=https://your-api-url

  # Or save to config file:
  exploop-admin config set-context prod --api-url=https://your-api-url --api-key=radm_...
  exploop-admin config use-context prod

Test the connection:
  exploop-admin cluster-info
```

> **Important**: Save the API key immediately! It cannot be retrieved later.

### Admin Roles

| Role | Permissions |
|------|-------------|
| `super_admin` | Full access: manage admins, agents, tokens, all operations |
| `ops_admin` | Manage agents and tokens, view queue stats |
| `viewer` | Read-only access to platform status |

---

## Admin CLI (exploop-admin)

The `exploop-admin` CLI provides kubectl-style commands for platform management.

### Installation

#### Binary Installation

```bash
# Download from releases
curl -LO https://github.com/exploopio/api/releases/latest/download/exploop-admin-linux-amd64
chmod +x exploop-admin-linux-amd64
sudo mv exploop-admin-linux-amd64 /usr/local/bin/exploop-admin

# Verify installation
exploop-admin version
```

#### Using Docker

```bash
# Create alias for convenience
alias exploop-admin='docker run --rm -it \
  -e EXPLOOP_API_URL=$EXPLOOP_API_URL \
  -e EXPLOOP_API_KEY=$EXPLOOP_API_KEY \
  -v ~/.exploop:/root/.exploop \
  exploopio/admin-cli:latest'

# Use normally
exploop-admin get agents
```

### Configuration

The CLI supports three configuration methods (in priority order):

#### 1. Command Line Flags (Highest Priority)

```bash
exploop-admin --api-url=https://api.exploop.io --api-key=radm_xxx get agents
```

#### 2. Environment Variables

```bash
export EXPLOOP_API_URL=https://api.exploop.io
export EXPLOOP_API_KEY=radm_a1b2c3d4e5f6...
exploop-admin get agents
```

#### 3. Config File (~/.exploop/config.yaml)

```bash
# Create context
exploop-admin config set-context prod \
  --api-url=https://api.exploop.io \
  --api-key=radm_a1b2c3d4e5f6...

# Or use key file for security
echo "radm_a1b2c3d4e5f6..." > ~/.exploop/prod-key
chmod 600 ~/.exploop/prod-key
exploop-admin config set-context prod \
  --api-url=https://api.exploop.io \
  --api-key-file=~/.exploop/prod-key

# Switch contexts
exploop-admin config use-context prod

# List contexts
exploop-admin config get-contexts
```

**Config file format:**

```yaml
# ~/.exploop/config.yaml
apiVersion: admin.exploop.io/v1
kind: Config
current-context: prod

contexts:
  - name: prod
    context:
      api-url: https://api.exploop.io
      api-key-file: ~/.exploop/prod-key

  - name: staging
    context:
      api-url: https://api.staging.exploop.io
      api-key-file: ~/.exploop/staging-key

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
exploop-admin cluster-info

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
exploop-admin get agents
exploop-admin get agents -o wide
exploop-admin get agents -o json

# Get specific agent
exploop-admin describe agent agent-us-east-1

# Create agent (for manual registration)
exploop-admin create agent \
  --name=agent-us-east-1 \
  --region=us-east-1 \
  --capabilities=sast,sca,secrets \
  --max-jobs=10

# Maintenance operations
exploop-admin drain agent agent-us-east-1      # Stop accepting new jobs
exploop-admin uncordon agent agent-us-east-1   # Resume operations
exploop-admin delete agent agent-us-east-1     # Remove agent
```

#### Bootstrap Token Management

```bash
# List tokens
exploop-admin get tokens
exploop-admin get tokens -o wide

# Create token for agent registration
exploop-admin create token --max-uses=5 --expires=24h

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
#   ./agent -platform -bootstrap-token=abc123.xxxxxxxxxxxxxxxx -api-url=https://api.exploop.io

# Revoke token
exploop-admin revoke token tok-abc123 --reason="No longer needed"
```

#### Admin User Management (super_admin only)

```bash
# List admins
exploop-admin get admins

# Create new admin
exploop-admin create admin --email=ops@company.com --role=ops_admin

# Output includes the new admin's API key
```

#### Job Queue Monitoring

```bash
# List jobs
exploop-admin get jobs
exploop-admin get jobs --status=pending
exploop-admin get jobs --status=running
exploop-admin get jobs -o wide

# Job details
exploop-admin describe job job-xyz123

# View job logs
exploop-admin logs job job-xyz123

# Follow job logs in real-time (updates every 2s)
exploop-admin logs job job-xyz123 -f

# Show last N lines
exploop-admin logs job job-xyz123 --tail=100
```

#### Declarative Configuration (apply)

Similar to `kubectl apply`, you can create resources from YAML files:

```bash
# Apply from file
exploop-admin apply -f agent.yaml

# Apply from stdin
cat agent.yaml | exploop-admin apply -f -
```

**Agent manifest example:**

```yaml
# agent.yaml
apiVersion: admin.exploop.io/v1
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
apiVersion: admin.exploop.io/v1
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
apiVersion: admin.exploop.io/v1
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
exploop-admin delete agent agent-us-east-1

# Force delete without confirmation
exploop-admin delete agent agent-us-east-1 --force

# Delete a token
exploop-admin delete token tok-abc123 --force
```

### Output Formats

```bash
# Default table format
exploop-admin get agents

# Wide table with more columns
exploop-admin get agents -o wide

# JSON output (for scripting)
exploop-admin get agents -o json

# YAML output
exploop-admin get agents -o yaml

# Just names (for scripting)
exploop-admin get agents -o name
```

### Watch Mode

```bash
# Real-time updates (refreshes every 2s)
exploop-admin get agents -w
```

---

## Platform Agent Setup

Platform agents are Rediver-managed agents that can be used by multiple tenants.

### Registration Methods

#### Method 1: Bootstrap Token (Recommended)

```bash
# 1. Create bootstrap token (on admin machine)
exploop-admin create token --max-uses=1 --expires=1h

# 2. Start agent with token (on agent machine)
./agent -platform \
  -bootstrap-token=abc123.xxxxxxxxxxxxxxxx \
  -api-url=https://api.exploop.io \
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
exploop-admin create agent \
  --name=agent-us-east-1 \
  --region=us-east-1 \
  --capabilities=sast,sca

# 2. Get the agent's API key from output
# 3. Start agent with key (on agent machine)
./agent -platform \
  -api-key=ragent_xxxxx \
  -api-url=https://api.exploop.io
```

### Docker Deployment

```bash
# Using bootstrap token
docker run -d \
  --name exploop-platform-agent \
  --restart unless-stopped \
  -e API_URL=https://api.exploop.io \
  -e BOOTSTRAP_TOKEN=abc123.xxxxxxxxxxxxxxxx \
  -v agent-data:/home/exploop/.exploop \
  exploopio/agent:platform

# Using pre-assigned API key
docker run -d \
  --name exploop-platform-agent \
  --restart unless-stopped \
  -e API_URL=https://api.exploop.io \
  -e API_KEY=ragent_xxxxx \
  exploopio/agent:platform
```

### Kubernetes Deployment

#### Option A: Using Pre-assigned API Key (Recommended for Production)

```bash
# 1. Create agent and get API key
exploop-admin create agent --name=agent-k8s-pool --region=us-east-1 --capabilities=sast,sca,secrets

# 2. Create secret with the API key
kubectl create secret generic exploop-agent-credentials \
  --from-literal=api-key=ragent_xxxxx \
  -n exploop
```

```yaml
# platform-agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exploop-platform-agent
  namespace: exploop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: exploop-platform-agent
  template:
    metadata:
      labels:
        app: exploop-platform-agent
    spec:
      containers:
        - name: agent
          image: exploopio/agent:platform
          env:
            - name: API_URL
              value: "https://api.exploop.io"
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: exploop-agent-credentials
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
              mountPath: /home/exploop/.exploop
      volumes:
        - name: agent-data
          emptyDir: {}
```

#### Option B: Using Bootstrap Token (Auto-registration)

For dynamic scaling where each pod registers independently:

```bash
# 1. Create bootstrap token with enough uses for your replicas
exploop-admin create token --max-uses=10 --expires=24h

# 2. Create secret with the bootstrap token
kubectl create secret generic.exploop-bootstrap-token \
  --from-literal=token=abc123.xxxxxxxxxxxxxxxx \
  -n exploop
```

```yaml
# platform-agent-bootstrap.yaml
apiVersion: apps/v1
kind: StatefulSet  # StatefulSet ensures unique agent names
metadata:
  name: exploop-platform-agent
  namespace: exploop
spec:
  serviceName: exploop-platform-agent
  replicas: 3
  selector:
    matchLabels:
      app: exploop-platform-agent
  template:
    metadata:
      labels:
        app: exploop-platform-agent
    spec:
      containers:
        - name: agent
          image: exploopio/agent:platform
          env:
            - name: API_URL
              value: "https://api.exploop.io"
            - name: BOOTSTRAP_TOKEN
              valueFrom:
                secretKeyRef:
                  name:.exploop-bootstrap-token
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
              mountPath: /home/exploop/.exploop
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
helm repo add.exploop https://charts.exploop.io
helm repo update
```

**Method 1: Bootstrap Token (Auto-registration)**

Best for dynamic scaling where each pod registers as a unique agent.

```bash
# 1. Create bootstrap token
exploop-admin create token --max-uses=10 --expires=24h

# 2. Install with bootstrap token
helm install platform-agent exploop/platform-agent \
  --namespace.exploop \
  --create-namespace \
  --set apiUrl=https://api.exploop.io \
  --set bootstrapToken=abc123.xxxxxxxxxxxxxxxx \
  --set replicaCount=3 \
  --set agent.region=us-east-1 \
  --set agent.capabilities=sast,sca,secrets
```

**Method 2: Pre-assigned API Key (Shared Identity)**

Best for production with fixed replicas sharing the same agent identity.

```bash
# 1. Create agent
exploop-admin create agent --name=k8s-pool --region=us-east-1 --capabilities=sast,sca,secrets

# 2. Install with API key (uses Deployment instead of StatefulSet)
helm install platform-agent exploop/platform-agent \
  --namespace.exploop \
  --create-namespace \
  --set apiUrl=https://api.exploop.io \
  --set apiKey=ragent_xxxxx \
  --set useStatefulSet=false \
  --set replicaCount=3
```

**Method 3: Existing Secret**

Use credentials stored in an existing Kubernetes secret.

```bash
# 1. Create secret with credentials
kubectl create secret generic exploop-agent-creds \
  --from-literal=api-key=ragent_xxxxx \
  -n exploop

# 2. Install using existing secret
helm install platform-agent exploop/platform-agent \
  --namespace.exploop \
  --set apiUrl=https://api.exploop.io \
  --set existingSecret.enabled=true \
  --set existingSecret.name=exploop-agent-creds \
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
helm upgrade platform-agent exploop/platform-agent \
  --namespace.exploop \
  --reuse-values \
  --set replicaCount=5

# Upgrade chart version
helm upgrade platform-agent exploop/platform-agent \
  --namespace.exploop \
  --reuse-values

# Uninstall
helm uninstall platform-agent -n exploop

# If using StatefulSet, also delete PVCs
kubectl delete pvc -l app.kubernetes.io/instance=platform-agent -n exploop
```

For full documentation, see the [Helm chart README](https://github.com/exploopio/charts/tree/main/charts/platform-agent).

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
│  │ exploop-admin   │──HTTPS:443──▶│   nginx/ALB     │                    │
│  │ ~/.exploop/     │              │   :443          │                    │
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
   echo "radm_xxx" > ~/.exploop/prod-key
   chmod 600 ~/.exploop/prod-key

   # Bad: key directly in config.yaml
   ```

3. **Rotate keys periodically**
   ```bash
   exploop-admin rotate-key admin <admin-id>
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
exploop-admin config view

# Test with verbose output
exploop-admin --verbose get agents

# Verify network connectivity
curl -v https://api.exploop.io/health
```

### Agent Not Registering

```bash
# Check token validity
exploop-admin get tokens

# Check agent logs
docker logs exploop-platform-agent

# Verify bootstrap token format: xxxxxx.yyyyyyyyyyyyyyyy
```

### Permission Denied

```bash
# Verify your role
exploop-admin get admins | grep your-email

# Check if operation requires super_admin
# Operations like "create admin" require super_admin role
```

---

## Quick Reference

### Command Cheat Sheet

```bash
# === CLUSTER INFO ===
exploop-admin cluster-info              # Platform overview
exploop-admin version                   # CLI version

# === AGENTS ===
exploop-admin get agents                # List all agents
exploop-admin get agents -o wide        # Detailed list
exploop-admin get agents -w             # Watch mode (auto-refresh)
exploop-admin describe agent <name>     # Agent details
exploop-admin create agent --name=<n> --region=<r> --capabilities=sast,sca
exploop-admin drain agent <name>        # Stop accepting new jobs
exploop-admin uncordon agent <name>     # Resume operations
exploop-admin delete agent <name>       # Remove agent

# === TOKENS ===
exploop-admin get tokens                # List bootstrap tokens
exploop-admin create token --max-uses=5 --expires=24h
exploop-admin describe token <id>       # Token details
exploop-admin revoke token <id> --reason="..."
exploop-admin delete token <id>         # Remove token

# === JOBS ===
exploop-admin get jobs                  # List platform jobs
exploop-admin get jobs --status=pending # Filter by status
exploop-admin describe job <id>         # Job details
exploop-admin logs job <id>             # View job logs
exploop-admin logs job <id> -f          # Follow logs in real-time

# === ADMINS ===
exploop-admin get admins                # List admin users
exploop-admin create admin --email=<e> --role=ops_admin

# === CONFIG ===
exploop-admin config set-context <name> --api-url=<url> --api-key=<key>
exploop-admin config use-context <name> # Switch context
exploop-admin config current-context    # Show current
exploop-admin config get-contexts       # List all contexts
exploop-admin config view               # Show full config

# === DECLARATIVE ===
exploop-admin apply -f agent.yaml       # Apply from file
cat manifest.yaml | exploop-admin apply -f -  # Apply from stdin
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

## Development Environment Setup

This section covers how to set up admin access in a local development environment.

### Prerequisites

1. PostgreSQL database running with migrations applied
2. API server running (either via Docker or directly)

### Method 1: Using bootstrap-admin Binary (Recommended for Dev)

Build and run the bootstrap-admin tool directly:

```bash
cd api

# Build the bootstrap-admin binary
go build -o ./bin/bootstrap-admin ./cmd/bootstrap-admin

# Run with database connection
./bin/bootstrap-admin \
  -db "postgres:/.exploop.exploop@localhost:5432/exploop?sslmode=disable" \
  -email "admin@localhost" \
  -name "Dev Admin" \
  -role "super_admin"
```

**Output:**
```
=== Bootstrap Admin Created ===
  ID:    550e8400-e29b-41d4-a716-446655440000
  Email: admin@localhost
  Name:  Dev Admin
  Role:  super_admin

API Key (save this, it won't be shown again):
  rdv-admin-a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6

Use this key to authenticate with the Admin UI or API.
```

### Method 2: Using Docker Compose (setup folder)

If you're using the `setup/` deployment with docker-compose:

```bash
cd setup

# For staging environment
make bootstrap-admin-staging email=admin@localhost

# For production environment
make bootstrap-admin-prod email=admin@localhost

# With custom role and name
make bootstrap-admin-staging email=ops@localhost role=ops_admin name="Ops User"
```

### Method 3: Direct Docker Exec

If API container is already running:

```bash
# Find your API container name
docker ps | grep api

# Execute bootstrap-admin inside the container
docker exec -it <container_name> /app/bootstrap-admin \
  -email "admin@localhost" \
  -role "super_admin"
```

### Method 4: Using psql (Emergency/Manual)

If the bootstrap-admin tool is unavailable, you can insert directly via SQL:

```bash
# Generate a bcrypt hash for your API key
# Use this Go snippet or an online bcrypt generator:
# echo -n "rdv-admin-your-secret-key-here" | htpasswd -bnBC 12 "" - | tr -d ':\n'

# Or generate in Go:
go run -e 'import "golang.org/x/crypto/bcrypt"; hash, _ := bcrypt.GenerateFromPassword([]byte("rdv-admin-testsecretkey123456789012345678901234567890123456"), 12); print(string(hash))'
```

```sql
-- Connect to database
psql -h localhost -U.exploop -d.exploop

-- Insert admin user
INSERT INTO admin_users (
  id, email, name, role, api_key_hash, api_key_prefix, is_active, created_at, updated_at
) VALUES (
  gen_random_uuid(),
  'admin@localhost',
  'Dev Admin',
  'super_admin',
  '$2a$12$your-bcrypt-hash-here',  -- bcrypt hash of your API key
  'rdv-admin-testsecr...',          -- first 20 chars of your key
  true,
  NOW(),
  NOW()
);
```

**Warning**: This method is not recommended. Use bootstrap-admin when possible.

### Using the Admin API Key

Once you have the API key, you can use it in several ways:

#### 1. Admin UI Login

1. Open Admin UI at `http://localhost:3001` (or your configured port)
2. Enter the API key in the login form
3. Click "Sign In"

#### 2. API Requests with curl

```bash
# Set your API key
export ADMIN_API_KEY="rdv-admin-your-key-here"

# Validate the key
curl -X GET http://localhost:8080/api/v1/admin/auth/validate \
  -H "X-Admin-API-Key: $ADMIN_API_KEY"

# List platform agents
curl -X GET http://localhost:8080/api/v1/admin/platform-agents \
  -H "X-Admin-API-Key: $ADMIN_API_KEY"

# Get agent stats
curl -X GET http://localhost:8080/api/v1/admin/platform-agents/stats \
  -H "X-Admin-API-Key: $ADMIN_API_KEY"

# List bootstrap tokens
curl -X GET http://localhost:8080/api/v1/admin/bootstrap-tokens \
  -H "X-Admin-API-Key: $ADMIN_API_KEY"
```

#### 3. Admin CLI (exploop-admin)

```bash
# Set environment variables
export EXPLOOP_API_URL=http://localhost:8080
export EXPLOOP_API_KEY=rdv-admin-your-key-here

# Or create a config context
exploop-admin config set-context local \
  --api-url=http://localhost:8080 \
  --api-key=rdv-admin-your-key-here

exploop-admin config use-context local

# Now use commands
exploop-admin get agents
exploop-admin get tokens
```

### Environment Variables for Development

Create a `.env.local` file for easy development:

```bash
# .env.local
ADMIN_API_KEY=rdv-admin-your-key-here
ADMIN_API_URL=http://localhost:8080
```

Then source it:

```bash
source .env.local
curl -X GET $ADMIN_API_URL/api/v1/admin/auth/validate \
  -H "X-Admin-API-Key: $ADMIN_API_KEY"
```

### Admin UI Development Configuration

For the Admin UI, set the API URL in `.env.local`:

```bash
# admin-ui/.env.local
NEXT_PUBLIC_API_URL=http://localhost:8080
```

### Troubleshooting Development Setup

#### "Invalid admin API key" error

1. Check if the key was copied correctly (no extra spaces)
2. Verify the admin user exists in database:
   ```sql
   SELECT id, email, role, is_active, api_key_prefix FROM admin_users;
   ```
3. Check if the key prefix matches what's stored

#### "CORS error" in Admin UI

Ensure your API has CORS configured for localhost:

```bash
# In .env.api.local or environment
CORS_ALLOWED_ORIGINS=*
# Or specific origins:
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001
CORS_ALLOWED_HEADERS=Accept,Authorization,Content-Type,X-Request-ID,X-Admin-API-Key
```

Then restart the API server.

#### "404 Not Found" on /api/v1/admin/auth/validate

1. Ensure API server was rebuilt after adding admin routes
2. Check that admin routes are registered:
   ```bash
   # If API has route listing
   ./server -routes | grep admin
   ```

#### Database connection failed in bootstrap-admin

Check your connection string format:
```bash
# Correct format
-db "postgres://user:pass@host:5432/dbname?sslmode=disable"

# Common mistakes:
# - Missing sslmode parameter
# - Wrong port (default is 5432)
# - Password with special characters needs URL encoding
```

### Quick Start Script

Create a `scripts/dev-setup-admin.sh` for convenience:

```bash
#!/bin/bash
# scripts/dev-setup-admin.sh

set -e

DB_URL="${DB_URL:-postgres:/.exploop.exploop@localhost:5432/exploop?sslmode=disable}"
ADMIN_EMAIL="${ADMIN_EMAIL:-admin@localhost}"
ADMIN_ROLE="${ADMIN_ROLE:-super_admin}"

echo "Building bootstrap-admin..."
cd api
go build -o ./bin/bootstrap-admin ./cmd/bootstrap-admin

echo "Creating admin user..."
./bin/bootstrap-admin \
  -db "$DB_URL" \
  -email "$ADMIN_EMAIL" \
  -role "$ADMIN_ROLE"

echo ""
echo "Done! Save the API key above."
echo ""
echo "To use with Admin UI:"
echo "  1. Open http://localhost:3001"
echo "  2. Enter the API key"
echo ""
echo "To use with curl:"
echo "  export ADMIN_API_KEY='rdv-admin-...'"
echo "  curl -H 'X-Admin-API-Key: \$ADMIN_API_KEY' http://localhost:8080/api/v1/admin/auth/validate"
```

Make it executable and run:

```bash
chmod +x scripts/dev-setup-admin.sh
./scripts/dev-setup-admin.sh
```

---

## Related Documentation

- [Authentication Guide](./authentication.md) - Tenant API key management
- [Running Agents](./running-agents.md) - Tenant agent setup
- [Docker Deployment](./docker-deployment.md) - Container deployment
- [API Reference](../backend/api-reference.md) - Full API documentation
