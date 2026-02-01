---
layout: default
title: AI Triage
parent: Features
nav_order: 15
---

# AI Triage

AI-powered vulnerability triage and analysis for intelligent prioritization and remediation guidance.

---

## Overview

AI Triage leverages Large Language Models (LLMs) to analyze security findings and provide:

- **Severity Assessment**: Validate or adjust severity based on context
- **Risk Scoring**: Calculate risk score (0-100) based on exploitability and impact
- **Priority Ranking**: Determine fix priority (1-100, 1 = most urgent)
- **False Positive Detection**: Identify likely false positives with confidence scores
- **Remediation Guidance**: Generate step-by-step fix recommendations
- **Exploitability Analysis**: Assess real-world exploitation difficulty

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              AI TRIAGE FLOW                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐     ┌──────────────────┐     ┌─────────────────────────┐   │
│  │   User/     │────▶│   API Handler    │────▶│   Rate Limiter          │   │
│  │   System    │     │   (HTTP)         │     │   (10 req/min/tenant)   │   │
│  └─────────────┘     └──────────────────┘     └───────────┬─────────────┘   │
│                                                           │                  │
│                                                           ▼                  │
│                                              ┌─────────────────────────┐     │
│                                              │   Deduplication Check   │     │
│                                              │   (HasPendingOrProcessing)    │
│                                              └───────────┬─────────────┘     │
│                                                          │                   │
│                           ┌──────────────────────────────┼───────────────┐   │
│                           │                              ▼               │   │
│  ┌─────────────────┐      │     ┌──────────────────────────────────┐    │   │
│  │   AI Triage     │◀─────┼─────│   Asynq Job Queue (Redis)        │    │   │
│  │   Service       │      │     │   - ai:triage:{result_id}        │    │   │
│  └────────┬────────┘      │     └──────────────────────────────────┘    │   │
│           │               │                                              │   │
│           ▼               │          WORKER PROCESS                      │   │
│  ┌─────────────────┐      │                                              │   │
│  │  AcquireSlot    │      │     ┌──────────────────────────────────┐    │   │
│  │  (SELECT FOR    │◀─────┼─────│   AI Triage Worker               │    │   │
│  │   UPDATE)       │      │     │   (ProcessTriage)                │    │   │
│  └────────┬────────┘      │     └──────────────────────────────────┘    │   │
│           │               │                                              │   │
│           ▼               └──────────────────────────────────────────────┘   │
│  ┌─────────────────┐                                                         │
│  │  Token Limit    │     ┌─────────────────────────────────────────────────┐ │
│  │  Check          │────▶│   LLM Provider Factory                          │ │
│  └────────┬────────┘     │   ├── Platform AI (Claude/OpenAI/Gemini)        │ │
│           │              │   ├── Tenant BYOK (encrypted API keys)          │ │
│           ▼              │   └── Self-Hosted Agent                         │ │
│  ┌─────────────────┐     └──────────────────────────────────────────────────┘ │
│  │  Prompt         │                                                         │
│  │  Sanitization   │────▶ Unicode normalization, injection pattern removal   │
│  └────────┬────────┘                                                         │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────┐     ┌──────────────────────────────────────────────┐   │
│  │  LLM Call       │────▶│   Output Validation                          │   │
│  │                 │     │   - JSON schema validation                   │   │
│  └─────────────────┘     │   - Field sanitization                       │   │
│                          │   - XSS prevention                           │   │
│                          └──────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## API Endpoints

### Request Triage (Single Finding)

```http
POST /api/v1/findings/{id}/ai-triage
Authorization: Bearer <token>
Content-Type: application/json

{
    "mode": "quick"  // or "detailed"
}
```

**Response:**
```json
{
    "job_id": "01HQ5K8M...",
    "status": "pending"
}
```

**Rate Limiting:**
- 10 requests per minute per tenant
- Returns `429 Too Many Requests` when exceeded
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

### Get Triage Result

```http
GET /api/v1/findings/{id}/ai-triage
Authorization: Bearer <token>
```

**Response:**
```json
{
    "id": "01HQ5K8M...",
    "finding_id": "01HQ5K7N...",
    "status": "completed",
    "severity_assessment": "high",
    "severity_justification": "SQL injection vulnerability allows authentication bypass...",
    "risk_score": 85,
    "exploitability": "high",
    "exploitability_details": "Publicly accessible endpoint with no input validation...",
    "business_impact": "Potential data breach affecting user credentials...",
    "priority_rank": 5,
    "false_positive_likelihood": 0.1,
    "false_positive_reason": "Input is directly concatenated into SQL query...",
    "remediation_steps": [
        {
            "step": 1,
            "description": "Use parameterized queries instead of string concatenation",
            "effort": "low"
        },
        {
            "step": 2,
            "description": "Add input validation for user-controlled parameters",
            "effort": "low"
        }
    ],
    "related_cves": ["CVE-2023-1234"],
    "related_cwes": ["CWE-89", "CWE-20"],
    "analysis_summary": "High-risk SQL injection vulnerability...",
    "llm_provider": "claude",
    "llm_model": "claude-3-5-sonnet-20241022",
    "prompt_tokens": 1250,
    "completion_tokens": 450,
    "completed_at": "2025-02-01T10:30:45Z"
}
```

### Get Triage History

```http
GET /api/v1/findings/{id}/ai-triage/history?limit=10&offset=0
Authorization: Bearer <token>
```

### Bulk Triage

```http
POST /api/v1/findings/ai-triage/bulk
Authorization: Bearer <token>
Content-Type: application/json

{
    "finding_ids": ["uuid1", "uuid2", ...],
    "mode": "quick"
}
```

**Limits:**
- Maximum 100 findings per request
- Maximum 64KB payload size
- Rate limited (same as single triage)

**Response:**
```json
{
    "jobs": [
        {"finding_id": "uuid1", "job_id": "job1", "status": "pending"},
        {"finding_id": "uuid2", "job_id": "job2", "status": "pending"}
    ],
    "total_count": 2,
    "queued": 2,
    "failed": 0
}
```

---

## Error Responses

### Duplicate Request (409 Conflict)

```json
{
    "code": "CONFLICT",
    "message": "A triage request is already in progress for this finding",
    "details": {
        "action": "Wait for the current triage to complete, or check the finding's triage history",
        "endpoint": "GET /api/v1/findings/{id}/ai-triage"
    }
}
```

### Token Limit Exceeded (429 Too Many Requests)

```json
{
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Monthly AI token limit exceeded",
    "details": {
        "action": "Wait until next month or upgrade your plan for more tokens",
        "suggestion": "Review your token usage in tenant settings"
    }
}
```

### AI Disabled (403 Forbidden)

```json
{
    "code": "FORBIDDEN",
    "message": "AI triage is disabled for this tenant. Contact your administrator to enable it.",
    "details": {
        "action": "Enable AI triage in tenant settings"
    }
}
```

---

## Security Features

### 1. Rate Limiting

Per-tenant rate limiting prevents abuse of expensive LLM API calls:

| Limit | Value |
|-------|-------|
| Requests per minute | 10 |
| Scope | Per tenant |
| Headers | `X-RateLimit-*` |

### 2. Token Race Condition Prevention

Uses `SELECT FOR UPDATE` to prevent concurrent workers from exceeding token limits:

```sql
-- Atomic slot acquisition
SELECT ... FROM ai_triage_results
WHERE id = $1 AND tenant_id = $2
FOR UPDATE OF tr
```

**Flow:**
1. Worker acquires row lock
2. Checks current token usage against monthly limit
3. Updates status to 'processing' atomically
4. Commits transaction before processing

### 3. Request Deduplication

Prevents duplicate triage requests for the same finding:

```sql
SELECT EXISTS(
    SELECT 1 FROM ai_triage_results
    WHERE tenant_id = $1 AND finding_id = $2
    AND status IN ('pending', 'processing')
)
```

### 4. Prompt Injection Protection

Multi-layer protection against prompt injection attacks:

#### Unicode Normalization
```
Attacker: "ｉｇｎｏｒｅ previous instructions"  (fullwidth)
Normalized: "ignore previous instructions"
→ Detected and filtered
```

#### Pattern Detection (50+ patterns)
- Instruction override: `ignore previous instructions`
- System prompt access: `output your system prompt`
- Role manipulation: `you are now`, `pretend to be`
- Special tokens: `[SYSTEM]`, `<|im_start|>`
- Jailbreak attempts: `DAN mode`, `developer mode`

#### Cyrillic Homoglyph Replacement
```
Attacker: "іgnore" (Cyrillic 'і')
Normalized: "ignore" (ASCII 'i')
→ Detected and filtered
```

### 5. API Key Encryption

Tenant API keys (BYOK) are encrypted with AES-256-GCM:

```
Storage: enc:v1:{base64_encrypted_data}
Decryption: Only at runtime, never logged
```

**Enforcement:**
- `requireEncryptedAPIKeys: true` by default
- Rejects plaintext API keys in production

### 6. Output Validation

LLM responses are validated and sanitized:

- JSON schema validation
- Field length limits
- XSS prevention (HTML entity encoding)
- Severity value validation
- Score range validation (0-100)

### 7. Audit Logging

All triage operations are logged:

| Event | Data Logged |
|-------|-------------|
| Request | tenant_id, finding_id, user_id, mode |
| Completion | tokens_used, provider, duration |
| Token Limit | current_usage, limit, finding_id |
| Failure | error_type, sanitized_message |

---

## Configuration

### Tenant Settings (UI / API)

```json
{
    "ai": {
        "mode": "platform",              // platform, byok, agent
        "auto_triage_enabled": true,
        "auto_triage_severities": ["critical", "high"],
        "auto_triage_delay_seconds": 60,
        "monthly_token_limit": 1000000
    }
}
```

### AI Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `platform` | Platform-managed LLM | Default, included in subscription |
| `byok` | Tenant's own API keys | Enterprise, data control |
| `agent` | Self-hosted AI agent | On-prem, strict compliance |

### Environment Variables

```bash
# Feature Flag
AI_TRIAGE_ENABLED=true

# Platform AI Provider (claude, openai, or gemini)
AI_PLATFORM_PROVIDER=claude
AI_PLATFORM_MODEL=claude-3-5-sonnet-20241022

# Provider API Keys (configure based on AI_PLATFORM_PROVIDER)
ANTHROPIC_API_KEY=sk-ant-...     # For Claude
OPENAI_API_KEY=sk-...            # For OpenAI
GEMINI_API_KEY=AIza...           # For Google Gemini

# Rate Limiting
AI_MAX_CONCURRENT_JOBS=10
AI_RATE_LIMIT_RPM=60
AI_TIMEOUT_SECONDS=30
AI_MAX_TOKENS=2000

# Encryption
AI_KEY_ENCRYPTION_KEY=base64-encoded-32-byte-key
```

### Supported LLM Providers

| Provider | Models | Use Case |
|----------|--------|----------|
| `claude` | claude-3-5-sonnet, claude-sonnet-4 | Best for security analysis (default) |
| `openai` | gpt-4-turbo, gpt-4o | Alternative, good general performance |
| `gemini` | gemini-1.5-pro, gemini-1.5-flash | Cost-effective, good for bulk triage |

### BYOK Provider Configuration

For tenants using Bring Your Own Key (BYOK) mode:

```json
{
    "ai": {
        "mode": "byok",
        "provider": "gemini",           // claude, openai, gemini
        "api_key": "enc:v1:...",        // Encrypted via PUT /api/v1/settings/ai-triage/key
        "model_override": "gemini-1.5-pro"
    }
}
```

---

## Data Model

### ai_triage_results Table

```sql
CREATE TABLE ai_triage_results (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    finding_id UUID NOT NULL,

    -- Request
    triage_type VARCHAR(50),    -- 'quick', 'detailed', 'auto', 'bulk'
    requested_by UUID,
    requested_at TIMESTAMP,

    -- Processing
    status VARCHAR(20),         -- pending, processing, completed, failed
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,

    -- AI Provider
    llm_provider VARCHAR(50),
    llm_model VARCHAR(100),
    prompt_tokens INT,
    completion_tokens INT,

    -- Analysis Results
    severity_assessment VARCHAR(20),
    severity_justification TEXT,
    risk_score FLOAT,
    exploitability VARCHAR(20),
    exploitability_details TEXT,
    business_impact TEXT,
    priority_rank INT,
    remediation_steps JSONB,
    false_positive_likelihood FLOAT,
    false_positive_reason TEXT,
    related_cves TEXT[],
    related_cwes TEXT[],
    raw_response JSONB,
    analysis_summary TEXT,

    -- Metadata
    metadata JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_ai_triage_tenant_finding
    ON ai_triage_results(tenant_id, finding_id);
CREATE INDEX idx_ai_triage_status
    ON ai_triage_results(status)
    WHERE status IN ('pending', 'processing');
```

---

## UI Integration

### AI Triage Button

Located in the finding detail panel:

```tsx
<AITriageButton
    findingId={finding.id}
    onComplete={(result) => refetchFinding()}
/>
```

**States:**
- `idle`: Shows "AI Analyze" button
- `pending`: Shows "Queued..." with spinner
- `processing`: Shows "Analyzing..." with progress
- `completed`: Shows result summary
- `failed`: Shows error with retry option

### Polling Behavior

Automatic polling with exponential backoff:

```
Initial: 2s → 3s → 4.5s → 6.7s → 10s (max)
```

### Result Display

```
┌─────────────────────────────────────────────────────┐
│  AI Analysis                                         │
├─────────────────────────────────────────────────────┤
│  Severity: HIGH (↑ from MEDIUM)                     │
│  Risk Score: 85/100                                 │
│  Priority: #5                                       │
│  False Positive: 10% likelihood                     │
├─────────────────────────────────────────────────────┤
│  Summary:                                           │
│  SQL injection vulnerability allows authentication   │
│  bypass in the login endpoint...                    │
├─────────────────────────────────────────────────────┤
│  Remediation Steps:                                 │
│  1. Use parameterized queries (Low effort)          │
│  2. Add input validation (Low effort)               │
│  3. Implement WAF rules (Medium effort)             │
└─────────────────────────────────────────────────────┘
```

---

## Flow Diagrams

### Single Triage Flow

```
User clicks "AI Analyze"
        │
        ▼
┌───────────────────┐
│ POST /ai-triage   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐     ┌───────────────────┐
│ Rate Limit Check  │────▶│ 429 Too Many      │
│ (10/min/tenant)   │ NO  │ Requests          │
└─────────┬─────────┘     └───────────────────┘
          │ YES
          ▼
┌───────────────────┐     ┌───────────────────┐
│ Deduplication     │────▶│ 409 Conflict      │
│ Check             │ DUP │ (already pending) │
└─────────┬─────────┘     └───────────────────┘
          │ OK
          ▼
┌───────────────────┐
│ Create pending    │
│ triage result     │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Enqueue to Asynq  │
│ (Redis)           │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Return job_id     │
│ status: pending   │
└───────────────────┘

        WORKER PICKS UP JOB
               │
               ▼
┌───────────────────┐     ┌───────────────────┐
│ AcquireTriageSlot │────▶│ Already processing│
│ (SELECT FOR       │ NO  │ Skip job          │
│  UPDATE)          │     └───────────────────┘
└─────────┬─────────┘
          │ OK
          ▼
┌───────────────────┐     ┌───────────────────┐
│ Token Limit Check │────▶│ Fail: limit       │
│                   │OVER │ exceeded          │
└─────────┬─────────┘     └───────────────────┘
          │ OK
          ▼
┌───────────────────┐
│ Create LLM        │
│ Provider          │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Build & Sanitize  │
│ Prompt            │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Call LLM API      │
│ (Claude/OpenAI)   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Validate & Store  │
│ Response          │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Update Finding    │
│ with AI fields    │
└───────────────────┘
```

### Auto-Triage Flow

```
Scanner creates finding
        │
        ▼
┌───────────────────┐
│ Check tenant      │
│ auto_triage_      │
│ enabled           │
└─────────┬─────────┘
          │ YES
          ▼
┌───────────────────┐     ┌───────────────────┐
│ Check severity    │────▶│ Skip: severity    │
│ in allowed list   │ NO  │ not configured    │
└─────────┬─────────┘     └───────────────────┘
          │ YES
          ▼
┌───────────────────┐
│ Enqueue with      │
│ delay (60s)       │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Worker processes  │
│ after delay       │
└───────────────────┘
```

---

## Best Practices

### For Administrators

1. **Set Token Limits**: Configure monthly limits to control costs
2. **Review Auto-Triage Settings**: Only enable for critical/high severity
3. **Use BYOK for Sensitive Data**: Configure tenant API keys for data privacy
4. **Monitor Usage**: Track token consumption in audit logs

### For Users

1. **Use Quick Mode First**: Start with quick triage, use detailed for complex findings
2. **Don't Repeat Requests**: Check if triage is already in progress
3. **Review AI Suggestions**: AI analysis should inform, not replace, human judgment
4. **Report Inaccuracies**: Help improve the system by reporting wrong assessments

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 409 Conflict | Duplicate request | Wait for current triage or check history |
| 429 Rate Limited | Too many requests | Wait and retry, or check tenant limits |
| Token limit exceeded | Monthly quota used | Wait for reset or increase limit |
| AI disabled | Tenant not configured | Enable in settings |
| Timeout | LLM API slow | Retry later |

### Debug Commands

```bash
# Check pending jobs
SELECT * FROM ai_triage_results
WHERE status = 'pending'
ORDER BY created_at DESC;

# Check token usage this month
SELECT SUM(prompt_tokens + completion_tokens) as total
FROM ai_triage_results
WHERE tenant_id = 'xxx'
  AND created_at >= date_trunc('month', NOW() AT TIME ZONE 'UTC')
  AND status = 'completed';

# Check for stuck jobs
SELECT * FROM ai_triage_results
WHERE status = 'processing'
  AND started_at < NOW() - INTERVAL '10 minutes';
```

---

## Workflow Integration

AI Triage integrates seamlessly with the [Workflow Automation](workflows.md) system.

### AI Triage Triggers

Workflows can be triggered when AI triage completes or fails:

| Trigger Type | Description | Available Filters |
|--------------|-------------|-------------------|
| `ai_triage_completed` | AI analysis finished successfully | `severity_filter`, `risk_score_min` |
| `ai_triage_failed` | AI analysis failed | None |

**Example: High Risk Alert Workflow**

```json
{
  "name": "High Risk AI Triage Alert",
  "nodes": [
    {
      "node_key": "trigger",
      "node_type": "trigger",
      "config": {
        "trigger_type": "ai_triage_completed",
        "trigger_config": {
          "severity_filter": ["critical", "high"],
          "risk_score_min": 70
        }
      }
    },
    {
      "node_key": "notify",
      "node_type": "notification",
      "config": {
        "notification_type": "slack",
        "notification_config": {
          "channel": "#security-critical",
          "template": {
            "text": "🚨 High Risk: {{trigger.summary}}\nRisk Score: {{trigger.risk_score}}"
          }
        }
      }
    }
  ]
}
```

### AI Triage Actions

Trigger AI analysis from workflows:

```json
{
  "node_key": "run_ai",
  "node_type": "action",
  "config": {
    "action_type": "trigger_ai_triage",
    "action_config": {
      "mode": "quick"
    }
  }
}
```

### Context Variables

When triggered by AI triage events, these variables are available in conditions:

| Variable | Type | Description |
|----------|------|-------------|
| `trigger.severity_assessment` | string | AI-assessed severity |
| `trigger.risk_score` | number | Risk score 0-100 |
| `trigger.priority_rank` | number | Priority rank 1-100 |
| `trigger.false_positive_likelihood` | number | FP probability 0-1 |
| `trigger.summary` | string | Analysis summary |
| `trigger.finding_id` | string | Associated finding ID |

### Common Patterns

**Auto-Escalate High Risk:**
```
Trigger(ai_triage_completed)
  → Condition(risk_score > 80)
  → Action(create_ticket)
  → Notify(pagerduty)
```

**Mark Likely False Positives:**
```
Trigger(ai_triage_completed)
  → Condition(false_positive_likelihood > 0.7)
  → Action(update_status: needs_review)
  → Action(add_tags: likely-fp)
```

**Handle Failures:**
```
Trigger(ai_triage_failed)
  → Action(add_tags: manual-review)
  → Notify(email: security-team)
```

---

## Related Documentation

- [ADR-007: AI Integration](../decisions/007-ai-integration.md) - Architecture Decision Record
- [Finding Lifecycle](finding-lifecycle.md) - How findings are managed
- [Workflow Automation](workflows.md) - Automate actions based on AI results
