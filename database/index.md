---
layout: default
title: Database
has_children: true
nav_order: 4
---

# Database

The platform uses **PostgreSQL 17** as the primary relational database and **Redis 7** for caching and queues.

## Schema Design

The database schema is organized into logical domains, supporting the clean architecture of the backend services.

- **[Schema Overview](./schema.md)**: Detailed breakdown of tables and relationships.
- **[Migrations](./migrations.md)**: Database version control and migration history.
- **[Platform Agent Tiers](./platform-agent-tiers.md)**: Tiered platform agent system with SQL functions (v3.3).
- **[Finding Repository Queries](./finding-queries.md)**: SQL queries and performance optimization for ingestion.

## Connection

| Environment | Host | Port | Database | User |
|-------------|------|------|----------|------|
| Development | `localhost` | `5432` | .exploop` | `postgres` |
| Docker | `postgres` | `5432` | .exploop` | `postgres` |
| Production | `<managed-db-host>` | `5432` | .exploop` | `<app-user>` |

## Key Concepts

- **Tenancy**: Multi-tenancy is supported via `tenant_id` column in most major entities.
- **Soft Deletes**: Used for `assets` and `findings` to preserve history.
- **UUIDs**: Primary keys are random UUID v4.
- **Audit**: Changes to critical entities are tracked in `audit_logs`.
