---
layout: default
title: Agent Usage Guide
parent: Platform Guides
nav_order: 12
---
{% raw %}

# Rediver Agent Usage Guide

The Rediver Agent is a modular security scanning tool that integrates with the Rediver platform. It supports multiple executor types for different security domains.

---

## Architecture Overview

The agent uses a **modular executor architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent Binary                           │
├─────────────────────────────────────────────────────────────┤
│                   Executor Router                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  Recon   │ │ VulnScan │ │ Secrets  │ │  Assets  │  ...  │
│  │ Executor │ │ Executor │ │ Executor │ │ Executor │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │            │            │            │              │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐        │
│  │subfinder│  │ nuclei  │  │gitleaks │  │  cloud  │        │
│  │  dnsx   │  │  trivy  │  │trufflehog│  │  apis  │        │
│  │ naabu   │  │ semgrep │  └─────────┘  └─────────┘        │
│  │ httpx   │  └─────────┘                                  │
│  │ katana  │                                               │
│  └─────────┘                                               │
└─────────────────────────────────────────────────────────────┘
```

**Executors:**
- **Recon**: External attack surface discovery (subdomains, DNS, ports, HTTP, crawling)
- **VulnScan**: Vulnerability scanning (SAST, SCA, DAST, IaC)
- **Secrets**: Secret and credential detection
- **Assets**: Cloud asset collection
- **Pipeline**: Workflow orchestration (coming soon)

---

## Installation

### Option 1: Go Install

```bash
go install github.com/rediverio/agent@latest
```

### Option 2: Build from Source

```bash
git clone https://github.com/rediverio/agent.git
cd agent

# Standard build (CLI-only tools)
go build -o agent ./agent/

# Platform mode build (for managed agents)
go build -tags platform -o agent ./agent/

# Hybrid mode build (library + CLI, reduces process overhead)
go build -tags hybrid -o agent ./agent/

# Full build (platform + hybrid)
go build -tags "platform,hybrid" -o agent ./agent/
```

### Option 3: Docker

```bash
# Pull from Docker Hub
docker pull rediverio/agent:latest

# Available variants
docker pull rediverio/agent:full    # All tools included (~1GB)
docker pull rediverio/agent:slim    # Agent only (~20MB)
docker pull rediverio/agent:ci      # CI/CD optimized (~1.2GB)
```

---

## Quick Start

### 1. Create an Agent in Rediver UI

1. Navigate to **Scoping > Agents**
2. Click **+ Add Agent**
3. Fill in details (name, type, capabilities)
4. **Copy the API key** (shown only once!)

### 2. Run Your First Scan

```bash
# Set credentials
export API_URL=https://api.rediver.io
export API_KEY=rdw_your_api_key_here

# Run a scan and push results
./agent -tool semgrep -target /path/to/project -push -verbose
```

---

## Command Line Options

### Basic Options

| Flag | Description | Example |
|------|-------------|---------|
| `-tool` | Single tool to run | `-tool semgrep` |
| `-tools` | Multiple tools (comma-separated) | `-tools semgrep,gitleaks,trivy` |
| `-target` | Path or URL to scan | `-target ./src` |
| `-push` | Push results to Rediver | `-push` |
| `-verbose` | Enable verbose logging | `-verbose` |
| `-config` | Path to config file | `-config agent.yaml` |

### Executor Options

| Flag | Description | Default |
|------|-------------|---------|
| `-enable-recon` | Enable reconnaissance executor | `false` |
| `-enable-vulnscan` | Enable vulnerability scan executor | `true` |
| `-enable-secrets` | Enable secrets detection executor | `false` |
| `-enable-assets` | Enable asset collection executor | `false` |
| `-enable-pipeline` | Enable pipeline orchestration | `false` |

### Execution Modes

| Flag | Description |
|------|-------------|
| `-daemon` | Run as continuous daemon |
| `-platform` | Run as platform-managed agent (requires `-tags platform` build) |
| `-list-tools` | List available tools |
| `-check-tools` | Check tool installation status |

### Platform Agent Options (requires `-tags platform` build)

| Flag | Description |
|------|-------------|
| `-api-url` | Platform API URL |
| `-bootstrap-token` | One-time bootstrap token for registration |
| `-region` | Agent region for job routing |
| `-max-concurrent` | Maximum concurrent jobs |
| `-credentials-file` | Path to store agent credentials |

### CI/CD Options

| Flag | Description |
|------|-------------|
| `-auto-ci` | Auto-detect CI environment |
| `-comments` | Post findings as PR/MR comments |
| `-sarif` | Generate SARIF output |
| `-sarif-output` | SARIF output file path |

---

## Available Scanners

### VulnScan Executor (SAST, SCA, DAST, IaC)

| Tool | Type | Description |
|------|------|-------------|
| `semgrep` | SAST | Code analysis with dataflow/taint tracking |
| `trivy` | SCA | Vulnerability scanning (filesystem) |
| `trivy-config` | IaC | Infrastructure misconfiguration |
| `trivy-image` | Container | Container image scanning |
| `trivy-full` | All | vuln + misconfig + secret |
| `nuclei` | DAST | Dynamic application security testing |

### Secrets Executor

| Tool | Type | Description |
|------|------|-------------|
| `gitleaks` | Secret | Secret and credential detection |
| `trufflehog` | Secret | Git history secret scanning |

### Recon Executor (External Attack Surface)

| Tool | Type | Description |
|------|------|-------------|
| `subfinder` | Subdomain | Passive subdomain enumeration |
| `dnsx` | DNS | DNS resolution and record lookup |
| `naabu` | Port | Fast port scanning |
| `httpx` | HTTP | HTTP probing and tech detection |
| `katana` | Crawler | Web crawling and URL discovery |

### Software Bill of Materials (SBOM)

When you run a vulnerability scan with Trivy, the agent automatically collects a Software Bill of Materials (SBOM). This inventory is stored in the `components` table (global library) and linked to your asset in `asset_components`.

- **Coverage**: Application dependencies (npm, pip, go.mod, etc.) and OS packages (apk, dpkg, rpm).
- **Global View**: Components are normalized globally, allowing you to track usage of a specific library version across all assets.
- **Snapshot Refresh**: Each scan refreshes the asset's dependency list, removing stale links and adding new ones, ensuring the "Vulnerable" status is always up-to-date with the latest scan.

To view the SBOM for an asset, use the API endpoint:
`GET /api/v1/assets/{id}/components`

### Check Tool Status

```bash
# List all available tools
./agent -list-tools

# Check which tools are installed
./agent -check-tools
```

---

## Configuration

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `API_URL` | Yes* | Platform API URL |
| `API_KEY` | Yes* | Agent API key |
| `AGENT_ID` | No | Agent UUID for tracking |
| `GITHUB_TOKEN` | Auto | GitHub token (for PR comments) |
| `GITLAB_TOKEN` | Auto | GitLab token (for MR comments) |

*Required when using `-push` flag

### Configuration File

Create `agent.yaml`:

```yaml
# Agent settings
agent:
  name: "prod-scanner-01"
  enable_commands: true
  command_poll_interval: 30s
  heartbeat_interval: 1m
  log_level: info

# Platform connection
server:
  base_url: "https://api.rediver.io"
  api_key: "rdw_your_api_key"
  agent_id: ""  # Auto-generated if empty
  timeout: 30s
  max_retries: 3
  retry_delay: 2s

# Scanners
scanners:
  - name: semgrep
    enabled: true

  - name: gitleaks
    enabled: true

  - name: trivy
    enabled: true

# Targets for daemon mode
targets:
  - /opt/code/project1

# Output settings
output:
  format: json
  sarif: false

# CI/CD settings
ci:
  comments: true
  fail_on: critical
  push: true

# Advanced
advanced:
  max_concurrent: 2
  scan_timeout: 30m
  cache_dir: /tmp/app-cache
  exclude:
    - "**/node_modules/**"
    - "**/vendor/**"
    - "**/.git/**"
```

---

## Execution Modes

### One-Shot Mode (Single Scan)

Run a scan and exit:

```bash
# Single tool
./agent -tool semgrep -target ./src -push

# Multiple tools
./agent -tools semgrep,gitleaks,trivy -target . -push

# With config file
./agent -config agent.yaml -push

# Dry run (no push)
./agent -tool semgrep -target . -output ./results.json
```

> **Important**: In one-shot mode, use `-push` to send results to Rediver.

### Daemon Mode (Continuous)

Run as a long-running service:

```bash
# Start daemon
./agent -daemon -config agent.yaml

# With verbose logging
./agent -daemon -config agent.yaml -verbose

# Standalone (no server commands)
./agent -daemon -config agent.yaml -standalone
```

> **Note**: In daemon mode, findings are automatically pushed after each scan.

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Security Scan
on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Security Scan
        uses: docker://rediverio/agent:ci
        with:
          args: >-
            -tools semgrep,gitleaks,trivy
            -target .
            -auto-ci
            -comments
            -push
            -verbose
            -sarif
            -sarif-output results.sarif
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          API_URL: ${{ secrets.API_URL }}
          API_KEY: ${{ secrets.API_KEY }}

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

### GitLab CI

```yaml
stages:
  - security

security-scan:
  stage: security
  image: rediverio/agent:ci
  variables:
    GITLAB_TOKEN: $CI_JOB_TOKEN
    API_URL: $API_URL
    API_KEY: $API_KEY
  script:
    - |
      agent \
        -tools semgrep,gitleaks,trivy \
        -target . \
        -auto-ci \
        -comments \
        -push \
        -verbose \
        -sarif \
        -sarif-output gl-sast-report.json
  artifacts:
    reports:
      sast: gl-sast-report.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### CI Features

| Feature | Flag | Description |
|---------|------|-------------|
| Auto CI detection | `-auto-ci` | Detects GitHub/GitLab automatically |
| Inline comments | `-comments` | Posts findings as PR/MR comments |
| Push to platform | `-push` | Sends results to Rediver |
| SARIF output | `-sarif` | Generates SARIF for security dashboards |
| Diff-based scan | Automatic | Only scans changed files in PR/MR |

---

## Docker Usage

### Basic Scan

```bash
docker run --rm -v $(pwd):/scan rediverio/agent:latest \
    -tools semgrep,gitleaks,trivy \
    -target /scan \
    -verbose
```

### With API Push

```bash
docker run --rm \
    -v $(pwd):/scan \
    -e API_URL=https://api.rediver.io \
    -e API_KEY=rdw_your_api_key \
    rediverio/agent:latest \
    -tools semgrep,gitleaks,trivy \
    -target /scan \
    -push
```

### Docker Compose

```yaml
version: '3.8'
services:
  agent:
    image: rediverio/agent:latest
    volumes:
      - ./:/scan:ro
      - ./agent.yaml:/app/agent.yaml
    environment:
      - API_KEY=${API_KEY}
    command: -daemon -config /app/agent.yaml
```

---

## Running as a Service

### Systemd

Create `/etc/systemd/system/rediver-agent.service`:

```ini
[Unit]
Description=Rediver Security Scanner Agent
After=network.target

[Service]
Type=simple
User=rediver
Group=rediver
WorkingDirectory=/opt/rediver
ExecStart=/opt/rediver/agent -daemon -config /opt/rediver/agent.yaml
Restart=always
RestartSec=10
Environment=API_KEY=rdw_your_api_key

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable rediver-agent
sudo systemctl start rediver-agent
```

### Check Service Status

```bash
sudo systemctl status rediver-agent
journalctl -u rediver-agent -f
```

---

## Troubleshooting

### Agent Shows "Inactive"

Agents become inactive if no heartbeat is received for 5 minutes.

**Causes:**
- Agent not running
- API key invalid
- Network issues

**Solutions:**
1. Check agent is running: `ps aux | grep agent`
2. Check logs for errors
3. Verify API key is correct
4. Restart the agent

### 401 Unauthorized

- API key is invalid or revoked
- Regenerate key in UI: Agents > ... > Regenerate API Key

### Findings Not Appearing

**Most common cause**: Missing `-push` flag!

```bash
# WRONG
./agent -config agent.yaml

# CORRECT
./agent -config agent.yaml -push
```

Other causes:
- Check for "Pushed: X created" in output
- Verify API connection with `-verbose`
- Ensure asset exists in Rediver

### Tool Not Found

```bash
# Check tool installation
./agent -check-tools

# Install missing tools
./agent -install-tools
```

---

---

## Build Modes

The agent supports different build modes for different use cases.

### Standard Build (Default)

```bash
go build -o agent ./agent/
```

- CLI-only mode, spawns external processes for each tool
- No external library dependencies
- Best for: CI/CD, lightweight deployments

### Hybrid Build

```bash
go build -tags hybrid -o agent ./agent/
```

- Uses Go libraries directly when available (subfinder, dnsx, naabu, httpx, katana)
- Falls back to CLI for tools without library support
- Lower process overhead, faster execution
- Best for: High-volume scanning, memory-constrained environments

**Hybrid Configuration:**

```yaml
recon:
  use_hybrid_mode: true
  prefer_library: true    # Use library when available
  subfinder_use_lib: true
  dnsx_use_lib: true
  naabu_use_lib: true
  httpx_use_lib: true
  katana_use_lib: true
```

### Platform Build

```bash
go build -tags platform -o agent ./agent/
```

- Enables `-platform` flag for managed agent mode
- Supports bootstrap token registration
- Automatic job polling and lease renewal
- Best for: Platform-managed deployments

### Full Build (Platform + Hybrid)

```bash
go build -tags "platform,hybrid" -o agent ./agent/
```

- All features enabled
- Best for: Enterprise deployments

---

## Platform Agent Mode

Platform mode enables the agent to be managed by the Rediver platform, receiving jobs remotely and reporting results automatically.

### Registration with Bootstrap Token

```bash
# First-time registration
./agent -platform \
  -api-url https://api.rediver.io \
  -bootstrap-token abc123.xxxxxxxx \
  -region us-east-1 \
  -enable-recon \
  -enable-vulnscan

# Subsequent runs (uses stored credentials)
./agent -platform \
  -api-url https://api.rediver.io \
  -region us-east-1
```

### Platform Agent Configuration

```yaml
# platform-agent.yaml
platform:
  api_url: https://api.rediver.io
  credentials_file: ~/.rediver/agent-credentials.json
  region: us-east-1
  max_concurrent: 5

executors:
  recon:
    enabled: true
    use_hybrid_mode: true
  vulnscan:
    enabled: true
  secrets:
    enabled: false
  assets:
    enabled: false
```

### Job Types

The platform can dispatch these job types:

| Job Type | Routed To | Description |
|----------|-----------|-------------|
| `recon`, `subdomain`, `dns`, `portscan`, `http`, `crawler` | Recon Executor | External attack surface discovery |
| `scan`, `vulnscan`, `sast`, `sca`, `dast`, `container`, `iac` | VulnScan Executor | Vulnerability scanning |
| `secret`, `secrets` | Secrets Executor | Secret detection |
| `collect`, `assets`, `cloud` | Assets Executor | Asset collection |
| `pipeline` | Pipeline Executor | Workflow orchestration |

---

## Executor Reference

### Recon Executor

Discovers external attack surface assets.

**Capabilities:** `subdomain`, `dns`, `portscan`, `http`, `tech-detect`, `crawler`, `url-discovery`

**Tools:**
- **subfinder**: Passive subdomain enumeration using multiple sources
- **dnsx**: DNS resolution with A, AAAA, CNAME, MX, NS, TXT records
- **naabu**: Fast SYN/CONNECT port scanning
- **httpx**: HTTP probing with status codes, titles, tech detection
- **katana**: Web crawling and JavaScript parsing

**Example Job:**

```json
{
  "type": "recon",
  "payload": {
    "target": "example.com",
    "tools": ["subfinder", "dnsx", "httpx"],
    "options": {
      "resolve": true,
      "tech_detect": true
    }
  }
}
```

### VulnScan Executor

Scans for vulnerabilities across different security domains.

**Capabilities:** `sast`, `sca`, `dast`, `container`, `iac`, `misconfiguration`

**Tools:**
- **nuclei**: Template-based DAST scanning
- **trivy**: SCA vulnerability scanning (filesystem, image, config)
- **semgrep**: SAST with dataflow/taint tracking

**Example Job:**

```json
{
  "type": "vulnscan",
  "payload": {
    "scanner": "semgrep",
    "target": "/path/to/project",
    "options": {
      "config": ["p/security-audit", "p/owasp-top-ten"],
      "dataflow_trace": true
    }
  }
}
```

---

## RIS Output Format

All executors output findings in **RIS (Rediver Interchange Schema)** format:

```json
{
  "schema_version": "1.0",
  "tool": {
    "name": "semgrep",
    "version": "1.0.0"
  },
  "assets": [
    {
      "type": "repository",
      "value": "github.com/org/repo"
    }
  ],
  "findings": [
    {
      "id": "finding-1",
      "type": "vulnerability",
      "title": "SQL Injection",
      "severity": "critical",
      "rule_id": "CWE-89",
      "location": {
        "path": "src/db/query.go",
        "start_line": 45
      }
    }
  ]
}
```

---

## Using SDK Scanners Directly

For custom integrations, you can use the SDK scanners directly instead of the agent binary:

```go
import (
    "context"
    "github.com/rediverio/sdk/pkg/scanners/recon/subfinder"
    "github.com/rediverio/sdk/pkg/scanners/semgrep"
    "github.com/rediverio/sdk/pkg/enrichers/epss"
)

func main() {
    ctx := context.Background()

    // Recon scanning
    subScanner := subfinder.NewScanner()
    result, _ := subScanner.Scan(ctx, "example.com", nil)
    fmt.Printf("Found %d subdomains\n", len(result.Subdomains))

    // SAST scanning
    sastScanner := semgrep.NewScanner()
    findings, _ := sastScanner.ScanToFindings(ctx, "./src", nil)

    // Enrich with threat intelligence
    enricher := epss.NewEnricher()
    enriched, _ := enricher.EnrichBatch(ctx, findings)
}
```

See the [SDK Development Guide](sdk-development.md) for detailed SDK usage.

---

## Next Steps

- **[Running Agents](running-agents.md)** - Create agents in Rediver UI
- **[SDK Quick Start](sdk-quick-start.md)** - Use SDK directly
- **[SDK Development](sdk-development.md)** - Build custom scanners and collectors
- **[Custom Tools Development](custom-tools-development.md)** - Build your own tools
- **[Platform Administration](platform-admin.md)** - Manage platform agents
{% endraw %}
