---
layout: default
title: Schema Overview
parent: Database
nav_order: 1
---

# Schema Overview

## Core Domains

### 1. Identity & Access (IAM)
- **`users`**: Global users table.
- **`tenants`**: Organization/Tenant boundaries.
- **`tenant_members`**: Link between users and tenants (role assignment).
- **`roles`**: RBAC role definitions.
- **`permissions`**: Granular system permissions.
- **`role_permissions`**: Mapping of permissions to roles.

### 2. Assets Management
- **`assets`**: Core entity (servers, clouds, apps).
- **`asset_types`**: Definitions of asset categories.
- **`asset_groups`**: Logical grouping of assets (tag-based or static).
- **`components`**: Software components/libraries installed on assets.

### 3. Vulnerability Management
- **`findings`**: Security issues found on assets.
- **`vulnerabilities`**: Vulnerability definitions (CVEs).
- **`scans`**: Records of scan executions.
- **`scan_profiles`**: Configuration for scan jobs.

### 4. Operations
- **`agents`** (formerly workers): Distributed scan agents.
- **`audit_logs`**: System audit trail.
- **`jobs`**: Background job entries.

### 5. Platform Administration (Added v3.2)
- **`admin_users`**: Platform admin accounts with bcrypt API key auth.
- **`admin_audit_logs`**: Admin action audit trail (immutable).
- **`agent_leases`**: K8s-style lease management for platform agents.
- **`platform_agent_bootstrap_tokens`**: One-time registration tokens.
- **`platform_agent_registrations`**: Agent registration audit trail.

### 6. Pipelines & Scans
- **`pipelines`**: Scan pipeline templates.
- **`pipeline_steps`**: Steps within a pipeline.
- **`pipeline_runs`**: Pipeline execution instances.
- **`step_runs`**: Individual step execution records.
- **`commands`**: Job queue for agents (includes platform jobs).

## Relationships Diagram

```mermaid
erDiagram
    TENANT ||--o{ ASSET : owns
    TENANT ||--o{ USER : has_members
    ASSET ||--o{ FINDING : has
    ASSET ||--o{ COMPONENT : contains
    FINDING }|--|| VULNERABILITY : references
    SCAN ||--o{ FINDING : detects
```
