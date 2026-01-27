---
layout: default
title: Features
nav_order: 5
has_children: true
---

# Features

Documentation for major features in the Rediver CTEM Platform.

---

## Available Features

| Feature | Description | Status |
|---------|-------------|--------|
| [Component Interactions](component-interactions.md) | Complete overview of how components interact | ✅ Reference |
| [Platform Agents](platform-agents.md) | Shared scanning infrastructure managed by Rediver | ✅ Implemented |
| [Asset Sub-Modules](asset-sub-modules.md) | Hierarchical module system for asset types | ✅ Implemented |
| [Capabilities Registry](capabilities-registry.md) | Normalized tool capability management | ✅ Implemented |
| [Scan Profiles](scan-profiles.md) | Reusable scan configs with Quality Gates | ✅ Implemented |
| [Scanner Templates](scanner-templates.md) | Custom detection rules for Nuclei, Semgrep, Gitleaks | ✅ Implemented |

---

## Feature Categories

### Infrastructure
- **Platform Agents** - Shared, multi-tenant scanning agents
- **Storage Service** - Multi-tenant storage with BYOB support

### Asset Management
- **Asset Sub-Modules** - Dynamic control over asset type visibility
- **Asset Groups** - Organize assets into logical groups

### Access Control
- **Module Access Control** - Subscription-based feature gating
- **Group-Based Access** - Fine-grained resource permissions

### Scanning
- **Scan Pipelines** - Multi-step automated scanning workflows
- **Scan Profiles** - Reusable configurations with Quality Gates and Template Modes
- **Scan Orchestration** - Intelligent scheduling and progression
- **Scanner Templates** - Custom templates for Nuclei, Semgrep, Gitleaks
- **Capabilities Registry** - Normalized tool capability management with M:N relationships
- **Quality Gates** - CI/CD pass/fail decisions based on finding thresholds
