---
layout: default
title: Getting Started
nav_order: 1
---

# Getting Started with RediverIO Platform

Get your security platform running in **5 minutes**.

## What is RediverIO?

RediverIO is a **Continuous Threat Exposure Management (CTEM)** platform that helps you discover assets, scan for vulnerabilities, and prioritize remediation across your entire attack surface.

**Key Capabilities:**
- 🔍 **Asset Discovery** - Auto-discover repositories, cloud resources, smart contracts
- 🛡️ **Security Scanning** - SAST, SCA, secrets, IaC misconfigurations, Web3 vulnerabilities  
- 📊 **Risk Prioritization** - AI-powered scoring with EPSS and threat intelligence
- 🔗 **Multi-Source Integration** - Aggregate findings from Wiz, Tenable, CrowdStrike, and more

---

## Prerequisites

- **Docker & Docker Compose** (required)
- **Git** (required)
- **Go 1.25+** (optional, for development)

---

## 5-Minute Quickstart

### Step 1: Clone Workspace

```bash
# Clone the meta-repository
git clone https://github.com/rediverio/rediver-platform.git rediverio
cd rediverio

# Initialize all sub-repositories (api, agent, ui, sdk)
make setup
```

This clones the API, Agent, UI, and SDK repositories into your workspace.

---

### Step 2: Start the Platform

```bash
# Start all services (PostgreSQL, Redis, API, UI)
make up

# View logs (optional)
make logs
```

**Services Starting:**
- 🗄️ PostgreSQL (database)
- 🔴 Redis (cache & queues)
- 🔧 API (backend at `localhost:8080`)
- 🎨 UI (frontend at `localhost:3000`)
- 🛡️ Admin UI (admin console at `localhost:3001`) - Optional

Wait ~30 seconds for services to be healthy.

---

### Step 3: Access the UI

Open your browser:

**URL:** [http://localhost:3000](http://localhost:3000)

**Default Credentials:**
- Email: `admin@rediver.io`
- Password: `Admin123!`

---

### Step 4: Run Your First Scan

#### 4a. Create an Agent

1. Login to the UI
2. Navigate to **Settings → Agents**
3. Click **"Create Agent"**
4. Choose type: **Runner** (for CI/CD one-shot scans)
5. Copy the **API Key**

#### 4b. Run the Agent with Docker

```bash
docker run --rm \
  -v $(pwd):/scan \
  -e API_URL=http://localhost:8080 \
  -e API_KEY=your-api-key-here \
  rediverio/agent:latest \
  -tools semgrep,gitleaks,trivy -target /scan -push -verbose
```

Replace `your-api-key-here` with the key from step 4a.

This scans the current directory for:
- **semgrep** - Code vulnerabilities (SAST)
- **gitleaks** - Exposed secrets
- **trivy** - Package vulnerabilities (SCA)

#### 4c. View Results

1. Go to **Findings** in the UI
2. Filter by your repository/agent
3. Review detected vulnerabilities
4. Assign severity and remediation

---

## What's Next?

### Learn the Platform

| Topic | Guide |
|-------|-------|
| **Architecture** | [System Overview](./architecture/overview.md) |
| **End-to-End Workflow** | [Complete Scan Workflow](./guides/END_TO_END_WORKFLOW.md) |
| **API Reference** | [API Endpoints](./backend/api-reference.md) |
| **Agent Guide** | [Agent Quick Start](../agent/docs/QUICK_START.md) |

### Deploy to Production

| Topic | Guide |
|-------|-------|
| **Kubernetes** | [Production Deployment](./operations/PRODUCTION_DEPLOYMENT.md) |
| **Environment Config** | [Environment Variables](./ui/ops/ENVIRONMENT_VARIABLES.md) |
| **Security Hardening** | [Production Checklist](./ui/ops/PRODUCTION_CHECKLIST.md) |

### Platform Administration

| Topic | Guide |
|-------|-------|
| **Admin UI** | [Admin UI Documentation](./admin-ui/index.md) |
| **Platform Admin CLI** | [Platform Admin Guide](./guides/platform-admin.md) |
| **Platform Agents** | [Shared Agent Architecture](./features/platform-agents.md) |

### Integrate CI/CD

| Platform | Example |
|----------|---------|
| **GitHub Actions** | [Agent README](../agent/README.md#github-actions) |
| **GitLab CI** | [Agent README](../agent/README.md#gitlab-ci) |
| **Custom** | [SDK Documentation](../sdk/README.md) |

---

## Common Commands

```bash
# Start platform
make up

# Stop platform
make down

# View logs
make logs

# Update all repositories
make pull-all

# Check status
make status

# Restart services
make restart
```

---

## Troubleshooting

### "Port 3000 already in use"

```bash
# Find and kill the process
lsof -ti:3000 | xargs kill -9

# Or change the port in docker-compose
```

### "Database connection failed"

```bash
# Check PostgreSQL is running
docker compose ps

# Restart database
docker compose restart postgres
```

### "Cannot connect to API"

Check API is running:
```bash
curl http://localhost:8080/health
```

Expected response: `{"status":"ok"}`

---

## Need Help?

- 📚 **Documentation:** [docs.rediver.io](https://docs.rediver.io)
- 💬 **Discord:** [discord.gg/rediverio](https://discord.gg/rediverio)
- 🐛 **Issues:** [GitHub Issues](https://github.com/rediverio/rediver-platform/issues)

---

**Ready to scan? Let's go! 🚀**
