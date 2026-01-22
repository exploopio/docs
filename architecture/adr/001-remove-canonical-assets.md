# ADR-001: Remove Canonical Assets in Favor of Asset Groups

**Date:** 2026-01-19
**Status:** Accepted
**Decision Makers:** Engineering Team

---

## Context

During Phase 1 implementation of the CTEM (Continuous Threat Exposure Management) platform, we designed a "Canonical Assets" feature intended to provide unified asset identity resolution - grouping multiple native assets (domains, IPs, repos, etc.) into logical business entities.

However, upon evaluation, we discovered significant overlap with the existing "Asset Groups" feature.

## Problem Statement

Two overlapping features for asset organization created:
1. **User confusion** - When to use "Canonical Assets" vs "Asset Groups"?
2. **Duplicate functionality** - Both provide asset grouping with business context
3. **Maintenance overhead** - 3 additional tables, more code to maintain
4. **Terminology confusion** - "Canonical Assets" is not an official CTEM term

## Research Findings

### CTEM Framework Terminology (Gartner)

The official CTEM framework uses these terms:
- **Asset Inventory** - Technical assets discovered
- **Authoritative Asset Inventory** - Single source of truth
- **Scope / Attack Surface** - What to protect
- **Business Context** - Criticality, ownership mapping
- **CAASM** (Cyber Asset Attack Surface Management) - Tool category

**"Canonical Assets" is NOT an official CTEM term.** It's an engineering/database term (canonical form) that was incorrectly adopted.

### Asset Groups Already Has Business Context

Our existing Asset Groups implementation includes:

| Feature | Status |
|---------|--------|
| `criticality` (critical/high/medium/low) | Yes |
| `business_unit` | Yes |
| `owner` + `owner_email` | Yes |
| `environment` (production/staging/dev/test) | Yes |
| `tags` | Yes |
| `risk_score` (0-100, auto-calculated) | Yes |
| `finding_count` (real-time aggregation) | Yes |
| Asset type counts | Yes |
| Many-to-many asset linking | Yes |

### What Canonical Assets Would Add

The only unique features Canonical Assets would provide:

| Feature | Practical Need |
|---------|----------------|
| Link confidence scores | Low - Only for ML-based auto-correlation |
| Link verification workflow | Low - Manual linking = 100% confidence |
| Resolution rules | Medium - Nice-to-have for large scale |
| Business entity type | Medium - Can be added to Asset Groups |

## Decision

**Remove Canonical Assets and keep Asset Groups as the primary asset organization mechanism.**

### Rationale

1. **CTEM Alignment** - Asset Groups aligns with CTEM terminology (Scope, Business Context)
2. **Feature Parity** - Asset Groups already provides 90%+ of Canonical Assets functionality
3. **Simplicity** - One concept is easier to understand than two overlapping concepts
4. **Maintenance** - Less code, fewer tables, reduced complexity
5. **User Experience** - No confusion about which feature to use

## Consequences

### Positive
- Simpler mental model for users
- Reduced codebase complexity
- Faster development (no duplicate features)
- Aligned with industry standard CTEM terminology

### Negative
- No automatic resolution rules (can be added later if needed)
- No link confidence scores (rarely needed in practice)

### Neutral
- If "Business Entity Type" (service/application/infrastructure) is needed, add `group_type` field to Asset Groups

## Alternatives Considered

### Option A: Keep Both (Rejected)
- Canonical Assets for "business identity"
- Asset Groups for "operational grouping"
- **Rejected because:** Too much overlap, user confusion, CTEM doesn't use "canonical" terminology

### Option B: Merge into Asset Groups (Selected)
- Enhance Asset Groups with any missing features
- Remove Canonical Assets entirely
- **Selected because:** Simpler, aligns with CTEM, Asset Groups already comprehensive

### Option C: Replace Asset Groups with Canonical Assets (Rejected)
- Remove Asset Groups, use only Canonical Assets
- **Rejected because:** "Canonical Assets" is not industry-standard terminology, Asset Groups already mature

## Implementation

1. Remove Canonical Assets from UI (sidebar, pages, components)
2. Remove Canonical Assets API endpoints from frontend
3. Keep backend migrations/code for reference (can be removed later)
4. Update documentation to reflect decision
5. Consider adding `group_type` field to Asset Groups in future if needed

## References

- [Gartner CTEM Framework](https://www.gartner.com/en/documents/4922031)
- [CISA Asset Inventory Guidance](https://www.cisa.gov/resources-tools/resources/foundations-ot-cybersecurity-asset-inventory-guidance-owners-and-operators)
- [CAASM Definition](https://www.zscaler.com/products-and-solutions/caasm)

---

**Last Updated:** 2026-01-19
