---
layout: default
title: Home
nav_order: 1
permalink: /
---

# RediverIO Platform Documentation

Welcome to the **RediverIO Platform** - a Continuous Threat Exposure Management (CTEM) platform for discovering assets, scanning vulnerabilities, and prioritizing remediation.

---

## 🚀 Getting Started

**New to RediverIO?** Start here:

**→ [Getting Started Guide](./GETTING_STARTED.md)** - Get your platform running in 5 minutes

---

## 📚 Documentation Sections

### [🏗️ Architecture](./architecture/index.md)
High-level system design, data flow patterns, and technology choices.
- [System Overview](./architecture/overview.md)
- [Scan Flow Architecture](./architecture/scan-flow.md) - Complete scan lifecycle
- [Deployment Modes](./architecture/deployment-modes.md)
- [Scan Pipeline Design](./architecture/scan-pipeline-design.md)
- [Workflow Executor](./architecture/workflow-executor.md) - Automation with 14-layer security

### [💻 Backend Services](./backend/index.md)
Documentation for the Go-based API service.
- [API Reference](./backend/api-reference.md)

### [🗄️ Database](./database/index.md)
Data models, schema definitions, and migration strategies.
- [Schema Overview](./database/schema.md)
- [Migrations](./database/migrations.md)

### [🎨 User Interface](./ui/index.md)
Frontend documentation for the Next.js tenant dashboard.
- [UI Development Guides](./ui/guides/index.md)
- [UI Features](./ui/features/index.md)
- [UI Operations](./ui/ops/index.md)

### [🛡️ Admin UI](./admin-ui/index.md)
Platform administration console for managing agents, jobs, and tokens.
- [Getting Started](./admin-ui/index.md#getting-started)
- [Docker Deployment](./admin-ui/index.md#docker-deployment)
- [API Integration](./admin-ui/index.md#api-integration)

### [⚙️ Operations](./operations/index.md)
Development, deployment, and operational guides.
- [Development Guide](./operations/DEVELOPMENT.md)
- [Production Deployment](./operations/PRODUCTION_DEPLOYMENT.md) ⭐ **NEW**
- [Integration Guide](./operations/INTEGRATION.md)
- [Staging Deployment](./operations/STAGING_DEPLOYMENT.md)

### [📖 Platform Guides](./guides/index.md)
Comprehensive guides for using the platform.
- [End-to-End Workflow](./guides/END_TO_END_WORKFLOW.md)
- [Authentication](./guides/authentication.md)
- [Multi-Tenancy](./guides/multi-tenancy.md)
- [Scan Management](./guides/scan-management.md)
- [Security Best Practices](./guides/SECURITY.md) - Includes workflow automation security
- [Notification Integrations](./guides/notification-integrations.md)

### [🗺️ Roadmap](./ROADMAP.md)
Product roadmap and planned features by CTEM phase.

---

## 🔧 Component Documentation

### Agent
- **[Agent Quick Start](../agent/docs/QUICK_START.md)** ⭐ **NEW** - Run your first scan in 5 minutes
- [Agent README](../agent/README.md) - Complete documentation

### SDK
- **[SDK API Reference](../sdk/docs/API_REFERENCE.md)** ⭐ **NEW** - Package documentation
- [SDK README](../sdk/README.md) - Building custom tools

### Schemas
- [RIS Format](../schemas/README.md) - Rediver Ingest Schema

---

## 🎯 Quick Links by Role

### For New Users
| Guide | Description |
|-------|-------------|
| **[Getting Started](./GETTING_STARTED.md)** | 5-minute platform setup |
| **[End-to-End Workflow](./guides/END_TO_END_WORKFLOW.md)** | Complete scan walkthrough |
| **[Agent Quick Start](../agent/docs/QUICK_START.md)** | Run your first scan |

### For Developers
| Guide | Description |
|-------|-------------|
| **[Development Guide](./operations/DEVELOPMENT.md)** | Local development setup |
| **[API Reference](./backend/api-reference.md)** | Complete API documentation |
| **[SDK API Reference](../sdk/docs/API_REFERENCE.md)** | Build custom tools |

### For Operations
| Guide | Description |
|-------|-------------|
| **[Production Deployment](./operations/PRODUCTION_DEPLOYMENT.md)** | Kubernetes/Docker/Cloud |
| **[Staging Deployment](./operations/STAGING_DEPLOYMENT.md)** | Staging environment setup |
| **[Environment Variables](./ui/ops/ENVIRONMENT_VARIABLES.md)** | Configuration reference |

### For Platform Admins
| Guide | Description |
|-------|-------------|
| **[Admin UI](./admin-ui/index.md)** | Platform administration console |
| **[Platform Admin Guide](./guides/platform-admin.md)** | CLI and API management |
| **[Platform Agents](./features/platform-agents.md)** | Shared agent architecture |
| **[Capabilities Registry](./features/capabilities-registry.md)** | Tool capability management |

---

## 🤝 Contributing

Please refer to the [Root README](../README.md) for contribution guidelines and workspace setup.

---

**Questions? Check the guides above or visit [docs.rediver.io](https://docs.rediver.io)**
