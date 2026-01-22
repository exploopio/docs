# Plans & Licensing System

> Implementation guide for multi-tenant subscription plans with Stripe integration

## Overview

This document describes the architecture for implementing plan-based module access control in a multi-tenant SaaS system. It enables different tenants to purchase different subscription plans that unlock specific modules/features.

## Architecture: 3-Layer Access Control

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1: LICENSING (Tenant)                  │
├─────────────────────────────────────────────────────────────────┤
│  Tenant → Plan → Modules                                        │
│                                                                 │
│  "What modules can this tenant access?"                         │
│                                                                 │
│  Determined by: Subscription plan                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 2: RBAC (User)                         │
├─────────────────────────────────────────────────────────────────┤
│  User → Roles → Permissions                                     │
│                                                                 │
│  "What can this user do within allowed modules?"                │
│                                                                 │
│  Note: Can only assign permissions from modules tenant has      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 3: DATA SCOPE (Groups)                 │
├─────────────────────────────────────────────────────────────────┤
│  User → Groups → Assets/Data                                    │
│                                                                 │
│  "What data can this user see?"                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Database Schema

### 1. Modules Table

Modules are system-defined feature areas. Each module contains related permissions.

```sql
-- =============================================================================
-- Modules (System-defined feature areas)
-- =============================================================================
CREATE TABLE modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(50) UNIQUE NOT NULL,      -- 'assets', 'findings', 'scans'
    name VARCHAR(100) NOT NULL,            -- 'Asset Management'
    description TEXT,
    icon VARCHAR(50),                       -- Lucide icon name: 'shield', 'scan'
    category VARCHAR(50),                   -- 'core', 'security', 'compliance'
    display_order INT DEFAULT 0,
    is_core BOOLEAN DEFAULT FALSE,         -- Core modules available to all plans
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_modules_slug ON modules(slug);
CREATE INDEX idx_modules_category ON modules(category);

-- Seed data
INSERT INTO modules (slug, name, description, icon, category, is_core, display_order) VALUES
('dashboard', 'Dashboard', 'Overview and analytics dashboard', 'layout-dashboard', 'core', true, 1),
('assets', 'Asset Management', 'Manage and inventory assets', 'server', 'core', true, 2),
('findings', 'Vulnerability Findings', 'View and manage security findings', 'shield-alert', 'security', false, 3),
('scans', 'Security Scanning', 'Configure and run security scans', 'scan', 'security', false, 4),
('reports', 'Reports & Analytics', 'Generate security reports', 'file-text', 'security', false, 5),
('compliance', 'Compliance Management', 'Track compliance frameworks', 'clipboard-check', 'compliance', false, 6),
('integrations', 'Integrations', 'Third-party integrations', 'plug', 'platform', false, 7),
('api', 'API Access', 'REST API access', 'code', 'platform', false, 8),
('audit', 'Audit Logs', 'Security audit trail', 'scroll-text', 'enterprise', false, 9),
('sso', 'SSO & SAML', 'Single sign-on authentication', 'key', 'enterprise', false, 10),
('teams', 'Team Management', 'Manage team members and roles', 'users', 'core', true, 11);
```

### 2. Module Permissions Mapping

Maps which permissions belong to which module.

```sql
-- =============================================================================
-- Module Permissions (Which permissions belong to which module)
-- =============================================================================
CREATE TABLE module_permissions (
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,
    permission_id VARCHAR(100) NOT NULL,   -- 'assets:read', 'assets:write'
    PRIMARY KEY (module_id, permission_id)
);

-- Index for reverse lookup
CREATE INDEX idx_module_permissions_permission ON module_permissions(permission_id);

-- Seed data (example - map all existing permissions to modules)
INSERT INTO module_permissions (module_id, permission_id)
SELECT m.id, p.id
FROM modules m
CROSS JOIN permissions p
WHERE
    (m.slug = 'assets' AND p.id LIKE 'assets:%') OR
    (m.slug = 'findings' AND p.id LIKE 'findings:%') OR
    (m.slug = 'scans' AND p.id LIKE 'scans:%') OR
    (m.slug = 'reports' AND p.id LIKE 'reports:%') OR
    (m.slug = 'compliance' AND p.id LIKE 'compliance:%') OR
    (m.slug = 'integrations' AND p.id LIKE 'integrations:%') OR
    (m.slug = 'api' AND p.id LIKE 'api:%') OR
    (m.slug = 'audit' AND p.id LIKE 'audit:%') OR
    (m.slug = 'sso' AND p.id LIKE 'sso:%') OR
    (m.slug = 'teams' AND p.id LIKE 'team:%') OR
    (m.slug = 'dashboard' AND p.id LIKE 'dashboard:%');
```

### 3. Plans Table

Subscription plans/tiers that tenants can purchase.

```sql
-- =============================================================================
-- Plans (Subscription tiers)
-- =============================================================================
CREATE TABLE plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(50) UNIQUE NOT NULL,      -- 'free', 'pro', 'business', 'enterprise'
    name VARCHAR(100) NOT NULL,            -- 'Pro Plan'
    description TEXT,

    -- Pricing
    price_monthly DECIMAL(10, 2),          -- NULL for custom/enterprise
    price_yearly DECIMAL(10, 2),           -- Usually ~20% discount
    currency VARCHAR(3) DEFAULT 'USD',

    -- Stripe integration
    stripe_price_id_monthly VARCHAR(100),  -- Stripe Price ID for monthly
    stripe_price_id_yearly VARCHAR(100),   -- Stripe Price ID for yearly
    stripe_product_id VARCHAR(100),        -- Stripe Product ID

    -- Display
    is_public BOOLEAN DEFAULT TRUE,        -- Show on pricing page?
    is_popular BOOLEAN DEFAULT FALSE,      -- Highlight as "Most Popular"
    is_active BOOLEAN DEFAULT TRUE,
    display_order INT DEFAULT 0,

    -- Marketing
    features JSONB DEFAULT '[]',           -- ["Unlimited assets", "Priority support"]
    badge VARCHAR(50),                     -- 'POPULAR', 'BEST VALUE'

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_plans_slug ON plans(slug);
CREATE INDEX idx_plans_is_public ON plans(is_public) WHERE is_public = TRUE;

-- Seed data
INSERT INTO plans (slug, name, description, price_monthly, price_yearly, is_public, is_popular, display_order, features) VALUES
('free', 'Free', 'Get started with basic features', 0, 0, true, false, 1,
    '["Up to 50 assets", "2 team members", "Community support", "Basic dashboard"]'::jsonb),
('pro', 'Pro', 'For growing security teams', 49, 470, true, true, 2,
    '["Up to 500 assets", "10 team members", "Email support", "Vulnerability scanning", "Custom reports"]'::jsonb),
('business', 'Business', 'For organizations with compliance needs', 149, 1430, true, false, 3,
    '["Up to 2,000 assets", "25 team members", "Priority support", "Compliance frameworks", "API access", "Integrations"]'::jsonb),
('enterprise', 'Enterprise', 'Custom solutions for large organizations', NULL, NULL, true, false, 4,
    '["Unlimited assets", "Unlimited team members", "Dedicated support", "SSO/SAML", "Audit logs", "Custom integrations", "SLA guarantee"]'::jsonb);
```

### 4. Plan Modules Mapping

Which modules are included in each plan, with optional limits.

```sql
-- =============================================================================
-- Plan Modules (Which modules are included in each plan)
-- =============================================================================
CREATE TABLE plan_modules (
    plan_id UUID REFERENCES plans(id) ON DELETE CASCADE,
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,

    -- Optional limits for this module in this plan
    limits JSONB DEFAULT '{}',  -- {"max_items": 100, "max_scans_per_month": 10}

    PRIMARY KEY (plan_id, module_id)
);

-- Index
CREATE INDEX idx_plan_modules_plan ON plan_modules(plan_id);
CREATE INDEX idx_plan_modules_module ON plan_modules(module_id);

-- Seed data
-- Free plan: Core modules only with limits
INSERT INTO plan_modules (plan_id, module_id, limits)
SELECT p.id, m.id,
    CASE
        WHEN m.slug = 'assets' THEN '{"max_items": 50}'::jsonb
        WHEN m.slug = 'teams' THEN '{"max_members": 2}'::jsonb
        ELSE '{}'::jsonb
    END
FROM plans p, modules m
WHERE p.slug = 'free' AND m.is_core = TRUE;

-- Pro plan: Core + Security modules
INSERT INTO plan_modules (plan_id, module_id, limits)
SELECT p.id, m.id,
    CASE
        WHEN m.slug = 'assets' THEN '{"max_items": 500}'::jsonb
        WHEN m.slug = 'teams' THEN '{"max_members": 10}'::jsonb
        WHEN m.slug = 'scans' THEN '{"max_per_month": 100}'::jsonb
        ELSE '{}'::jsonb
    END
FROM plans p, modules m
WHERE p.slug = 'pro' AND (m.is_core = TRUE OR m.category = 'security');

-- Business plan: Core + Security + Compliance + Platform
INSERT INTO plan_modules (plan_id, module_id, limits)
SELECT p.id, m.id,
    CASE
        WHEN m.slug = 'assets' THEN '{"max_items": 2000}'::jsonb
        WHEN m.slug = 'teams' THEN '{"max_members": 25}'::jsonb
        ELSE '{}'::jsonb
    END
FROM plans p, modules m
WHERE p.slug = 'business' AND m.category IN ('core', 'security', 'compliance', 'platform');

-- Enterprise plan: All modules, no limits
INSERT INTO plan_modules (plan_id, module_id, limits)
SELECT p.id, m.id, '{}'::jsonb
FROM plans p, modules m
WHERE p.slug = 'enterprise';
```

### 5. Tenant Subscription

Add subscription fields to tenants table.

```sql
-- =============================================================================
-- Tenant Subscription (Extend tenants table)
-- =============================================================================
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS plan_id UUID REFERENCES plans(id);
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS subscription_status VARCHAR(20) DEFAULT 'active';
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS subscription_started_at TIMESTAMPTZ;
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS subscription_expires_at TIMESTAMPTZ;
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS subscription_cancelled_at TIMESTAMPTZ;
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS billing_cycle VARCHAR(10) DEFAULT 'monthly'; -- monthly, yearly
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS limits_override JSONB DEFAULT '{}';
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS billing_email VARCHAR(255);
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS stripe_customer_id VARCHAR(100);
ALTER TABLE tenants ADD COLUMN IF NOT EXISTS stripe_subscription_id VARCHAR(100);

-- Subscription status values:
-- 'active'    - Subscription is active
-- 'trial'     - In trial period
-- 'past_due'  - Payment failed, grace period
-- 'cancelled' - Cancelled but access until period end
-- 'expired'   - Access revoked

-- Index
CREATE INDEX idx_tenants_plan ON tenants(plan_id);
CREATE INDEX idx_tenants_subscription_status ON tenants(subscription_status);
CREATE INDEX idx_tenants_stripe_customer ON tenants(stripe_customer_id);

-- Default new tenants to free plan
UPDATE tenants SET plan_id = (SELECT id FROM plans WHERE slug = 'free')
WHERE plan_id IS NULL;
```

### 6. Usage Tracking (Optional)

Track usage for billing and limits enforcement.

```sql
-- =============================================================================
-- Usage Tracking (For metered billing and limits)
-- =============================================================================
CREATE TABLE usage_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    module_id UUID NOT NULL REFERENCES modules(id) ON DELETE CASCADE,
    metric VARCHAR(50) NOT NULL,           -- 'assets_count', 'scans_count', 'api_calls'
    value BIGINT NOT NULL DEFAULT 0,
    period_start DATE NOT NULL,            -- Start of billing period
    period_end DATE NOT NULL,              -- End of billing period
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE (tenant_id, module_id, metric, period_start)
);

-- Index
CREATE INDEX idx_usage_records_tenant ON usage_records(tenant_id);
CREATE INDEX idx_usage_records_period ON usage_records(period_start, period_end);

-- Function to get current usage
CREATE OR REPLACE FUNCTION get_tenant_usage(p_tenant_id UUID, p_metric VARCHAR)
RETURNS BIGINT AS $$
DECLARE
    v_usage BIGINT;
BEGIN
    SELECT COALESCE(SUM(value), 0) INTO v_usage
    FROM usage_records
    WHERE tenant_id = p_tenant_id
      AND metric = p_metric
      AND period_start <= CURRENT_DATE
      AND period_end >= CURRENT_DATE;

    RETURN v_usage;
END;
$$ LANGUAGE plpgsql;
```

## Backend Implementation

### 1. Domain Models

```go
// internal/domain/licensing/module.go
package licensing

type Module struct {
    ID          string
    Slug        string
    Name        string
    Description string
    Icon        string
    Category    string
    IsCore      bool
    IsActive    bool
}

type Plan struct {
    ID                    string
    Slug                  string
    Name                  string
    Description           string
    PriceMonthly          *float64
    PriceYearly           *float64
    Currency              string
    StripeProductID       string
    StripePriceIDMonthly  string
    StripePriceIDYearly   string
    IsPublic              bool
    IsPopular             bool
    Features              []string
    Modules               []PlanModule
}

type PlanModule struct {
    ModuleID string
    Module   *Module
    Limits   map[string]interface{}
}

type TenantSubscription struct {
    TenantID             string
    PlanID               string
    Plan                 *Plan
    Status               SubscriptionStatus
    BillingCycle         string
    StartedAt            time.Time
    ExpiresAt            *time.Time
    CancelledAt          *time.Time
    LimitsOverride       map[string]interface{}
    StripeCustomerID     string
    StripeSubscriptionID string
}

type SubscriptionStatus string

const (
    StatusActive    SubscriptionStatus = "active"
    StatusTrial     SubscriptionStatus = "trial"
    StatusPastDue   SubscriptionStatus = "past_due"
    StatusCancelled SubscriptionStatus = "cancelled"
    StatusExpired   SubscriptionStatus = "expired"
)
```

### 2. Licensing Service

```go
// internal/app/licensing_service.go
package app

type LicensingService struct {
    repo          LicensingRepository
    stripeClient  *stripe.Client
    logger        *logger.Logger
}

// CheckModuleAccess verifies if tenant has access to a module
func (s *LicensingService) CheckModuleAccess(ctx context.Context, tenantID, moduleSlug string) (bool, error) {
    subscription, err := s.repo.GetTenantSubscription(ctx, tenantID)
    if err != nil {
        return false, err
    }

    // Check subscription status
    if !subscription.IsActive() {
        return false, ErrSubscriptionInactive
    }

    // Check if module is in plan
    for _, pm := range subscription.Plan.Modules {
        if pm.Module.Slug == moduleSlug {
            return true, nil
        }
    }

    return false, nil
}

// CheckModuleLimit verifies if tenant is within limits for a module metric
func (s *LicensingService) CheckModuleLimit(ctx context.Context, tenantID, moduleSlug, metric string, currentValue int64) (bool, int64, error) {
    subscription, err := s.repo.GetTenantSubscription(ctx, tenantID)
    if err != nil {
        return false, 0, err
    }

    // Find module in plan
    var planModule *PlanModule
    for _, pm := range subscription.Plan.Modules {
        if pm.Module.Slug == moduleSlug {
            planModule = &pm
            break
        }
    }

    if planModule == nil {
        return false, 0, ErrModuleNotInPlan
    }

    // Check for override limits first
    if override, ok := subscription.LimitsOverride[metric]; ok {
        limit := int64(override.(float64))
        return currentValue < limit, limit, nil
    }

    // Check plan limits
    if limit, ok := planModule.Limits[metric]; ok {
        limitVal := int64(limit.(float64))
        return currentValue < limitVal, limitVal, nil
    }

    // No limit defined = unlimited
    return true, -1, nil
}

// GetTenantModules returns all modules available to a tenant
func (s *LicensingService) GetTenantModules(ctx context.Context, tenantID string) ([]Module, error) {
    subscription, err := s.repo.GetTenantSubscription(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    modules := make([]Module, 0, len(subscription.Plan.Modules))
    for _, pm := range subscription.Plan.Modules {
        modules = append(modules, *pm.Module)
    }

    return modules, nil
}
```

### 3. Permission Check Integration

```go
// internal/app/auth_service.go
package app

// CheckPermission performs 3-layer access check
func (s *AuthService) CheckPermission(ctx context.Context, userID, tenantID, permission string) error {
    // Layer 1: Check tenant's plan includes the module for this permission
    moduleSlug, err := s.getPermissionModule(permission)
    if err != nil {
        return err
    }

    hasModule, err := s.licensingService.CheckModuleAccess(ctx, tenantID, moduleSlug)
    if err != nil {
        return err
    }
    if !hasModule {
        return ErrModuleNotInPlan
    }

    // Layer 2: Check user has this permission via RBAC
    hasPermission, err := s.rbacService.UserHasPermission(ctx, userID, tenantID, permission)
    if err != nil {
        return err
    }
    if !hasPermission {
        return ErrPermissionDenied
    }

    // Layer 3: Data scope is checked at query level
    return nil
}

// getPermissionModule looks up which module a permission belongs to
func (s *AuthService) getPermissionModule(permission string) (string, error) {
    // Cache this mapping for performance
    // permission format: "module:action" e.g., "assets:read"
    parts := strings.SplitN(permission, ":", 2)
    if len(parts) < 2 {
        return "", ErrInvalidPermission
    }
    return parts[0], nil
}
```

### 4. Stripe Integration

```go
// internal/app/billing_service.go
package app

import (
    "github.com/stripe/stripe-go/v76"
    "github.com/stripe/stripe-go/v76/checkout/session"
    "github.com/stripe/stripe-go/v76/customer"
    "github.com/stripe/stripe-go/v76/subscription"
    "github.com/stripe/stripe-go/v76/webhook"
)

type BillingService struct {
    repo           LicensingRepository
    stripeKey      string
    webhookSecret  string
    logger         *logger.Logger
}

// CreateCheckoutSession creates a Stripe Checkout session for subscription
func (s *BillingService) CreateCheckoutSession(ctx context.Context, tenantID, planSlug, billingCycle string) (string, error) {
    tenant, err := s.repo.GetTenant(ctx, tenantID)
    if err != nil {
        return "", err
    }

    plan, err := s.repo.GetPlanBySlug(ctx, planSlug)
    if err != nil {
        return "", err
    }

    // Get or create Stripe customer
    var customerID string
    if tenant.StripeCustomerID != "" {
        customerID = tenant.StripeCustomerID
    } else {
        params := &stripe.CustomerParams{
            Email: stripe.String(tenant.BillingEmail),
            Metadata: map[string]string{
                "tenant_id": tenantID,
            },
        }
        c, err := customer.New(params)
        if err != nil {
            return "", err
        }
        customerID = c.ID

        // Save customer ID
        s.repo.UpdateTenantStripeCustomer(ctx, tenantID, customerID)
    }

    // Get price ID based on billing cycle
    priceID := plan.StripePriceIDMonthly
    if billingCycle == "yearly" {
        priceID = plan.StripePriceIDYearly
    }

    // Create checkout session
    params := &stripe.CheckoutSessionParams{
        Customer: stripe.String(customerID),
        Mode:     stripe.String(string(stripe.CheckoutSessionModeSubscription)),
        LineItems: []*stripe.CheckoutSessionLineItemParams{
            {
                Price:    stripe.String(priceID),
                Quantity: stripe.Int64(1),
            },
        },
        SuccessURL: stripe.String("https://app.example.com/settings/billing?success=true"),
        CancelURL:  stripe.String("https://app.example.com/settings/billing?cancelled=true"),
        Metadata: map[string]string{
            "tenant_id": tenantID,
            "plan_slug": planSlug,
        },
    }

    sess, err := session.New(params)
    if err != nil {
        return "", err
    }

    return sess.URL, nil
}

// HandleWebhook processes Stripe webhook events
func (s *BillingService) HandleWebhook(payload []byte, signature string) error {
    event, err := webhook.ConstructEvent(payload, signature, s.webhookSecret)
    if err != nil {
        return err
    }

    switch event.Type {
    case "checkout.session.completed":
        return s.handleCheckoutCompleted(event)
    case "customer.subscription.updated":
        return s.handleSubscriptionUpdated(event)
    case "customer.subscription.deleted":
        return s.handleSubscriptionDeleted(event)
    case "invoice.payment_failed":
        return s.handlePaymentFailed(event)
    case "invoice.paid":
        return s.handleInvoicePaid(event)
    }

    return nil
}

func (s *BillingService) handleCheckoutCompleted(event stripe.Event) error {
    var sess stripe.CheckoutSession
    if err := json.Unmarshal(event.Data.Raw, &sess); err != nil {
        return err
    }

    tenantID := sess.Metadata["tenant_id"]
    planSlug := sess.Metadata["plan_slug"]

    // Get plan
    plan, err := s.repo.GetPlanBySlug(context.Background(), planSlug)
    if err != nil {
        return err
    }

    // Update tenant subscription
    return s.repo.UpdateTenantSubscription(context.Background(), tenantID, &TenantSubscription{
        PlanID:               plan.ID,
        Status:               StatusActive,
        StripeSubscriptionID: sess.Subscription.ID,
        StartedAt:            time.Now(),
    })
}
```

## Frontend Implementation

### 1. Types

```typescript
// src/features/licensing/types/index.ts

export interface Module {
  id: string;
  slug: string;
  name: string;
  description: string;
  icon: string;
  category: string;
  is_core: boolean;
}

export interface Plan {
  id: string;
  slug: string;
  name: string;
  description: string;
  price_monthly: number | null;
  price_yearly: number | null;
  currency: string;
  is_public: boolean;
  is_popular: boolean;
  features: string[];
  modules: PlanModule[];
}

export interface PlanModule {
  module_id: string;
  module: Module;
  limits: Record<string, number>;
}

export interface TenantSubscription {
  plan_id: string;
  plan: Plan;
  status: 'active' | 'trial' | 'past_due' | 'cancelled' | 'expired';
  billing_cycle: 'monthly' | 'yearly';
  started_at: string;
  expires_at: string | null;
  limits_override: Record<string, number>;
}
```

### 2. Hooks

```typescript
// src/features/licensing/api/use-licensing.ts

import useSWR from 'swr';
import { useTenant } from '@/context/tenant-context';

// Get tenant's current subscription
export function useTenantSubscription() {
  const { currentTenant } = useTenant();
  const { data, error, isLoading, mutate } = useSWR<TenantSubscription>(
    currentTenant ? `/api/v1/tenants/${currentTenant.slug}/subscription` : null
  );

  return {
    subscription: data,
    error,
    isLoading,
    mutate,
  };
}

// Get all available plans
export function usePlans() {
  const { data, error, isLoading } = useSWR<Plan[]>('/api/v1/plans');

  return {
    plans: data ?? [],
    error,
    isLoading,
  };
}

// Get tenant's available modules
export function useTenantModules() {
  const { currentTenant } = useTenant();
  const { data, error, isLoading } = useSWR<Module[]>(
    currentTenant ? `/api/v1/tenants/${currentTenant.slug}/modules` : null
  );

  return {
    modules: data ?? [],
    error,
    isLoading,
  };
}

// Check if tenant has access to a module
export function useModuleAccess(moduleSlug: string) {
  const { modules, isLoading } = useTenantModules();

  return {
    hasAccess: modules.some(m => m.slug === moduleSlug),
    isLoading,
  };
}

// Get module limits
export function useModuleLimits(moduleSlug: string) {
  const { subscription, isLoading } = useTenantSubscription();

  const planModule = subscription?.plan.modules.find(
    pm => pm.module.slug === moduleSlug
  );

  const limits = {
    ...planModule?.limits,
    ...subscription?.limits_override,
  };

  return {
    limits,
    isLoading,
  };
}
```

### 3. Components

```tsx
// src/features/licensing/components/module-gate.tsx

interface ModuleGateProps {
  module: string;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function ModuleGate({ module, children, fallback }: ModuleGateProps) {
  const { hasAccess, isLoading } = useModuleAccess(module);

  if (isLoading) {
    return <Skeleton className="h-32 w-full" />;
  }

  if (!hasAccess) {
    return fallback ?? <UpgradePrompt module={module} />;
  }

  return <>{children}</>;
}

// src/features/licensing/components/upgrade-prompt.tsx

interface UpgradePromptProps {
  module: string;
  variant?: 'banner' | 'card' | 'inline';
}

export function UpgradePrompt({ module, variant = 'card' }: UpgradePromptProps) {
  const { modules } = useTenantModules();
  const moduleInfo = modules.find(m => m.slug === module);

  if (variant === 'banner') {
    return (
      <div className="bg-gradient-to-r from-primary/10 to-primary/5 border border-primary/20 rounded-lg p-4">
        <div className="flex items-center gap-3">
          <Lock className="h-5 w-5 text-primary" />
          <div className="flex-1">
            <p className="font-medium">Upgrade to unlock {moduleInfo?.name}</p>
            <p className="text-sm text-muted-foreground">
              This feature is not available on your current plan.
            </p>
          </div>
          <Button asChild>
            <Link href="/settings/billing">Upgrade Plan</Link>
          </Button>
        </div>
      </div>
    );
  }

  return (
    <Card className="text-center py-12">
      <Lock className="h-12 w-12 mx-auto text-muted-foreground mb-4" />
      <h3 className="text-lg font-semibold mb-2">
        {moduleInfo?.name ?? 'Feature'} Not Available
      </h3>
      <p className="text-muted-foreground mb-6 max-w-md mx-auto">
        Upgrade your plan to access this feature and unlock more capabilities.
      </p>
      <Button asChild>
        <Link href="/settings/billing">View Plans</Link>
      </Button>
    </Card>
  );
}

// src/features/licensing/components/limit-indicator.tsx

interface LimitIndicatorProps {
  module: string;
  metric: string;
  currentValue: number;
  label?: string;
}

export function LimitIndicator({ module, metric, currentValue, label }: LimitIndicatorProps) {
  const { limits } = useModuleLimits(module);
  const limit = limits[metric];

  if (!limit || limit === -1) {
    return null; // No limit
  }

  const percentage = (currentValue / limit) * 100;
  const isNearLimit = percentage >= 80;
  const isAtLimit = percentage >= 100;

  return (
    <div className="space-y-1">
      <div className="flex justify-between text-sm">
        <span className="text-muted-foreground">{label ?? metric}</span>
        <span className={cn(
          isAtLimit && "text-destructive",
          isNearLimit && !isAtLimit && "text-warning"
        )}>
          {currentValue.toLocaleString()} / {limit.toLocaleString()}
        </span>
      </div>
      <Progress
        value={Math.min(percentage, 100)}
        className={cn(
          isAtLimit && "bg-destructive/20 [&>div]:bg-destructive",
          isNearLimit && !isAtLimit && "bg-warning/20 [&>div]:bg-warning"
        )}
      />
    </div>
  );
}
```

### 4. Usage in Pages

```tsx
// src/app/(dashboard)/compliance/page.tsx

export default function CompliancePage() {
  return (
    <ModuleGate module="compliance">
      <ComplianceDashboard />
    </ModuleGate>
  );
}

// src/components/layout/sidebar.tsx

export function Sidebar() {
  const { modules } = useTenantModules();
  const hasModule = (slug: string) => modules.some(m => m.slug === slug);

  return (
    <nav>
      {/* Core modules - always shown */}
      <SidebarItem icon={LayoutDashboard} label="Dashboard" href="/" />
      <SidebarItem icon={Server} label="Assets" href="/assets" />

      {/* Conditional modules based on plan */}
      {hasModule('findings') && (
        <SidebarItem icon={ShieldAlert} label="Findings" href="/findings" />
      )}
      {hasModule('scans') && (
        <SidebarItem icon={Scan} label="Scans" href="/scans" />
      )}
      {hasModule('compliance') && (
        <SidebarItem icon={ClipboardCheck} label="Compliance" href="/compliance" />
      )}

      {/* Or show locked items */}
      {!hasModule('compliance') && (
        <SidebarItem
          icon={Lock}
          label="Compliance"
          href="/settings/billing"
          className="opacity-50"
        />
      )}
    </nav>
  );
}
```

### 5. Billing Page

```tsx
// src/app/(dashboard)/settings/billing/page.tsx

export default function BillingPage() {
  const { subscription, isLoading } = useTenantSubscription();
  const { plans } = usePlans();

  if (isLoading) return <BillingPageSkeleton />;

  return (
    <div className="space-y-8">
      {/* Current Plan */}
      <Card>
        <CardHeader>
          <CardTitle>Current Plan</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="flex items-center justify-between">
            <div>
              <h3 className="text-2xl font-bold">{subscription?.plan.name}</h3>
              <p className="text-muted-foreground">{subscription?.plan.description}</p>
            </div>
            <Badge variant={subscription?.status === 'active' ? 'default' : 'destructive'}>
              {subscription?.status}
            </Badge>
          </div>

          {/* Usage limits */}
          <div className="mt-6 space-y-4">
            <LimitIndicator module="assets" metric="max_items" currentValue={assets.length} label="Assets" />
            <LimitIndicator module="teams" metric="max_members" currentValue={members.length} label="Team Members" />
          </div>
        </CardContent>
      </Card>

      {/* Available Plans */}
      <div>
        <h2 className="text-xl font-semibold mb-4">Available Plans</h2>
        <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-4">
          {plans.map(plan => (
            <PlanCard
              key={plan.id}
              plan={plan}
              isCurrentPlan={plan.id === subscription?.plan_id}
            />
          ))}
        </div>
      </div>
    </div>
  );
}
```

## API Endpoints

```
# Plans (Public)
GET    /api/v1/plans                           # List all public plans
GET    /api/v1/plans/{slug}                    # Get plan details

# Tenant Subscription
GET    /api/v1/tenants/{tenant}/subscription   # Get subscription
GET    /api/v1/tenants/{tenant}/modules        # Get available modules
GET    /api/v1/tenants/{tenant}/usage          # Get usage metrics

# Billing (Stripe)
POST   /api/v1/billing/checkout                # Create checkout session
POST   /api/v1/billing/portal                  # Create customer portal session
POST   /api/v1/billing/webhook                 # Stripe webhook handler

# Admin (Super admin only)
GET    /api/v1/admin/plans                     # List all plans
POST   /api/v1/admin/plans                     # Create plan
PUT    /api/v1/admin/plans/{id}                # Update plan
DELETE /api/v1/admin/plans/{id}                # Delete plan
POST   /api/v1/admin/tenants/{id}/subscription # Manually set subscription
```

## Implementation Checklist

### Phase 1: Database & Backend Core
- [ ] Create migration for modules table
- [ ] Create migration for plans table
- [ ] Create migration for plan_modules table
- [ ] Add subscription fields to tenants table
- [ ] Create licensing repository
- [ ] Create licensing service
- [ ] Implement module access check
- [ ] Integrate with permission check

### Phase 2: Stripe Integration
- [ ] Set up Stripe account and products
- [ ] Create billing service
- [ ] Implement checkout session creation
- [ ] Implement webhook handler
- [ ] Handle subscription lifecycle events
- [ ] Create customer portal integration

### Phase 3: Frontend
- [ ] Create licensing types
- [ ] Create hooks (useModuleAccess, useTenantModules, etc.)
- [ ] Create ModuleGate component
- [ ] Create UpgradePrompt component
- [ ] Update sidebar with module gates
- [ ] Create billing settings page
- [ ] Create plan selection UI

### Phase 4: Polish & Testing
- [ ] Add usage tracking
- [ ] Implement limit checks
- [ ] Add upgrade/downgrade flows
- [ ] Add trial period handling
- [ ] Test all subscription states
- [ ] Add billing emails

## Environment Variables

```env
# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Pricing Page URL
PRICING_URL=https://example.com/pricing
```

## References

- [Stripe Subscription Documentation](https://stripe.com/docs/billing/subscriptions/overview)
- [Stripe Checkout](https://stripe.com/docs/payments/checkout)
- [Stripe Customer Portal](https://stripe.com/docs/billing/subscriptions/integrating-customer-portal)
- [Stripe Webhooks](https://stripe.com/docs/webhooks)

---

**Last Updated**: 2026-01-22
