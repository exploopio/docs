---
layout: default
title: Development Guide
parent: Operations
nav_order: 10
---

# Development Guide

This guide details the workflow for developing on the RediverIO platform.

## Architecture Overview

RediverIO follows a Polyrepo structure managed via a central Workspace (Meta-Repo).

*   **API**: Core business logic and REST endpoints.
*   **Agent**: Distributed security scanner that runs on customer infrastructure/CI.
*   **UI**: Web management console.
*   **SDK**: Shared libraries used by API and Agent.

## Workspace Management

We avoid Git Submodules in favor of a simpler "Meta-Repo" approach handling by `setup-workspace.sh`.

### Directory Layout

```
rediverio/               # Root Workspace (Meta-Repo)
├── Makefile             # Global orchestration
├── go.work              # Go Workspace config
├── setup-workspace.sh   # Ops script
├── api/                 # -> git@github.com:rediverio/api.git
├── agent/               # -> git@github.com:rediverio/agent.git
├── sdk/                 # -> git@github.com:rediverio/sdk.git
└── ...
```

## Daily Workflow

### 1. Syncing Code

Start your day by syncing all repositories:

```bash
make pull-all
```

This iterates through every subdirectory and runs `git pull`.

### 2. Cross-Module Development (Go)

If you need to add a feature to `sdk` and use it in `api`:

1.  Modify `sdk/pkg/newfeature.go`.
2.  In `api/go.mod`, **DO NOT** use `replace` directive.
3.  Because `go.work` exists in root, the `api` build will automatically use your local `sdk` code.
    ```go
    // In api/main.go
    import "github.com/rediverio/sdk/pkg/newfeature" // Works immediately!
    ```
4.  **Commit Sequence:**
    *   Commit & Push `sdk` first.
    *   Get the new `sdk` version/commit hash.
    *   Update `api/go.mod` to use the new `sdk` version:
        ```bash
        cd api
        go get github.com/rediverio/sdk@latest
        ```
    *   Commit & Push `api`.

## IDE Setup

### VS Code (Recommended)

#### Required Extensions

*   **Go**: `golang.go`
*   **Frontend**: `dbaeumer.vscode-eslint`, `esbenp.prettier-vscode`

#### Workspace Settings

It is recommended to have a `.vscode/settings.json` in your workspace focusing on `go.lintTool: "golangci-lint"`.

### JetBrains (GoLand)

1.  Enable `goimports` on save.
2.  Set `golangci-lint` as external linter.

## Backend Development (API)

### Running Locally

```bash
cd api
make install-tools  # Install golangci-lint, air, migrate
make dev           # Run with hot reload
make run           # Run normally
```

### Database Migrations

```bash
make migrate-create name=add_users_table
make migrate-up
make migrate-down
```

### Testing

```bash
make test
make test-coverage
```

## Frontend Development (UI)

### Setup & Run

```bash
cd ui
npm install
npm run dev        # Dev mode with Turbopack
npm run lint -- --fix
```

## Environment Variables

### Generate Secrets

```bash
# JWT Secret (Backend)
openssl rand -base64 48

# CSRF Secret (Frontend)
openssl rand -base64 32
```

### Docker Development

Use the root Makefile to spin up the full stack:

```bash
make up        # Start Postgres, Keycloak, etc.
make logs      # View logs
make down      # Stop everything
```

## Troubleshooting

### Go Module Issues

If VSCode complains about modules:
1.  Open the **Root Folder** (`rediverio/`).
2.  Run `go work sync`.
3.  Restart VSCode/Go Language Server.

### Docker Issues

```bash
# Clean everything
docker compose down -v
docker system prune -a
```
