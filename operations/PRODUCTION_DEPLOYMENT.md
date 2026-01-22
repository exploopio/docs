---
layout: default
title: Production Deployment
parent: Operations
nav_order: 13
---

# Production Deployment Guide

Deploy the RediverIO platform to production environments using Kubernetes, Docker Compose, or cloud-managed services.

---

## Architecture Overview

```
                    ┌─────────────────────────────────┐
                    │    Load Balancer / Ingress      │
                    │    (TLS Termination)             │
                    └─────────────────┬───────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            │                         │                         │
            ▼                         ▼                         ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │  UI (Next.js) │      │   API (Go)    │      │   Keycloak    │
    │   Replicas: 2 │      │   Replicas: 3 │      │   (Optional)  │
    └───────────────┘      └───────────────┘      └───────────────┘
            │                         │                         │
            └─────────────────────────┼─────────────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            │                         │                         │
            ▼                         ▼                         ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │  PostgreSQL   │      │     Redis     │      │   Keycloak    │
    │  Primary +    │      │   Cluster     │      │   PostgreSQL  │
    │  Replicas     │      │   Sentinel    │      │               │
    └───────────────┘      └───────────────┘      └───────────────┘
```

---

## Deployment Options

### Option 1: Kubernetes with Helm (Recommended)

**Best for:** Production, Auto-scaling, High Availability

### Option 2: Docker Compose

**Best for:** Small teams, Single-server deployments

### Option 3: Cloud Managed Services

**Best for:** Minimal operations overhead

---

## Option 1: Kubernetes Deployment

### Prerequisites

- Kubernetes cluster (v1.25+)
- Helm 3
- kubectl configured
- Domain name with DNS access
- SSL/TLS certificate (or cert-manager)

---

### Step 1: Prepare Namespace

```bash
# Create namespace
kubectl create namespace rediver

# Set as default
kubectl config set-context --current --namespace=rediver
```

---

### Step 2: Create Secrets

```bash
# Generate secure secrets
export JWT_SECRET=$(openssl rand -base64 64)
export CSRF_SECRET=$(openssl rand -base64 32)
export DB_PASSWORD=$(openssl rand -base64 32)
export REDIS_PASSWORD=$(openssl rand -base64 32)

# Create Kubernetes secrets
kubectl create secret generic rediver-secrets \
  --from-literal=jwt-secret=$JWT_SECRET \
  --from-literal=csrf-secret=$CSRF_SECRET \
  --from-literal=db-password=$DB_PASSWORD \
  --from-literal=redis-password=$REDIS_PASSWORD \
  --namespace=rediver
```

---

### Step 3: Configure Values

Create `values.yaml`:

```yaml
# values.yaml
global:
  domain: app.rediver.io
  tlsEnabled: true

api:
  replicaCount: 3
  image:
    repository: rediverio/api
    tag: latest
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  env:
    AUTH_PROVIDER: local  # or "oidc" for Keycloak
    CORS_ALLOWED_ORIGINS: "https://app.rediver.io"
    LOG_LEVEL: info

ui:
  replicaCount: 2
  image:
    repository: rediverio/ui
    tag: latest
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "1Gi"
      cpu: "500m"

postgresql:
  enabled: true
  auth:
    existingSecret: rediver-secrets
    secretKeys:
      adminPasswordKey: db-password
  primary:
    persistence:
      size: 50Gi
  readReplicas:
    replicaCount: 1

redis:
  enabled: true
  auth:
    existingSecret: rediver-secrets
    existingSecretPasswordKey: redis-password
  master:
    persistence:
      size: 10Gi

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: app.rediver.io
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: rediver-tls
      hosts:
        - app.rediver.io
```

---

### Step 4: Install Helm Chart

```bash
# Add Rediver Helm repository
helm repo add rediverio https://charts.rediver.io
helm repo update

# Install
helm install rediver rediverio/rediver \
  --namespace rediver \
  --values values.yaml \
  --wait --timeout 10m

# Check status
helm status rediver --namespace rediver
```

---

### Step 5: Verify Deployment

```bash
# Check all pods are running
kubectl get pods --namespace rediver

# Expected output:
# NAME                          READY   STATUS    RESTARTS   AGE
# rediver-api-xxx               1/1     Running   0          2m
# rediver-api-yyy               1/1     Running   0          2m
# rediver-api-zzz               1/1     Running   0          2m
# rediver-ui-xxx                1/1     Running   0          2m
# rediver-ui-yyy                1/1     Running   0          2m
# rediver-postgresql-0          1/1     Running   0          2m
# rediver-redis-master-0        1/1     Running   0          2m

# Check services
kubectl get svc --namespace rediver

# Check ingress
kubectl get ingress --namespace rediver
```

---

### Step 6: Run Database Migrations

```bash
# Run migrations job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: rediver-migrate
  namespace: rediver
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: rediverio/api:latest
        command: ["migrate", "-path", "/app/migrations", "-database", "\$(DATABASE_URL)", "up"]
        env:
        - name: DATABASE_URL
          value: "postgres://rediver:\$(DB_PASSWORD)@rediver-postgresql:5432/rediver?sslmode=disable"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: rediver-secrets
              key: db-password
      restartPolicy: OnFailure
EOF

# Wait for completion
kubectl wait --for=condition=complete job/rediver-migrate --namespace rediver --timeout=5m
```

---

### Step 7: Seed Initial Data (Optional)

```bash
# Seed test data for staging
kubectl exec -it deployment/rediver-api --namespace rediver -- \
  psql \$DATABASE_URL -f /app/seeds/seed_required.sql
```

---

### Step 8: Access the Platform

1. Update your DNS to point to the Ingress IP:
   ```bash
   kubectl get ingress rediver-ingress --namespace rediver
   ```

2. Navigate to your domain: `https://app.rediver.io`

3. Login with default credentials (change immediately):
   - Email: `admin@rediver.io`
   - Password: `Admin123!`

---

## Option 2: Docker Compose Deployment

### Prerequisites

- Docker Engine 24+
- Docker Compose v2
- Domain with DNS access
- SSL certificate (or use Certbot)

---

### Step 1: Create Production Compose File

Create `docker-compose.prod.yml`:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: rediver
      POSTGRES_USER: rediver
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rediver"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s

  api:
    image: rediverio/api:latest
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: rediver
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: rediver
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      AUTH_JWT_SECRET: ${JWT_SECRET}
      CSRF_SECRET: ${CSRF_SECRET}
      CORS_ALLOWED_ORIGINS: https://app.rediver.io
      NEXT_PUBLIC_APP_URL: https://app.rediver.io
      LOG_LEVEL: info
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  ui:
    image: rediverio/ui:latest
    restart: unless-stopped
    depends_on:
      - api
    environment:
      NEXT_PUBLIC_BACKEND_API_URL: http://api:8080
      NEXT_PUBLIC_APP_URL: https://app.rediver.io
      CSRF_SECRET: ${CSRF_SECRET}
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/health"]
      interval: 30s

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - ui
      - api

volumes:
  postgres_data:
  redis_data:
```

---

### Step 2: Configure Nginx

Create `nginx.conf`:

```nginx
upstream ui {
    server ui:3000;
}

upstream api {
    server api:8080;
}

server {
    listen 80;
    server_name app.rediver.io;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.rediver.io;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # UI
    location / {
        proxy_pass http://ui;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Health check
    location /health {
        proxy_pass http://api/health;
    }
}
```

---

### Step 3: Generate SSL Certificate

Using Let's Encrypt:

```bash
# Install certbot
sudo apt install certbot

# Generate certificate
sudo certbot certonly --standalone \
  -d app.rediver.io \
  --email admin@rediver.io \
  --agree-tos

# Copy certificates
sudo cp /etc/letsencrypt/live/app.rediver.io/fullchain.pem ./ssl/
sudo cp /etc/letsencrypt/live/app.rediver.io/privkey.pem ./ssl/
```

---

### Step 4: Deploy

```bash
# Create .env file
cat > .env.prod <<EOF
DB_PASSWORD=$(openssl rand -base64 32)
REDIS_PASSWORD=$(openssl rand -base64 32)
JWT_SECRET=$(openssl rand -base64 64)
CSRF_SECRET=$(openssl rand -base64 32)
EOF

# Start services
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d

# Check status
docker compose -f docker-compose.prod.yml ps
```

---

## Option 3: Cloud Managed Services

### AWS (ECS Fargate)

Use AWS Copilot or ECS Task Definitions with:
- **RDS PostgreSQL** - Managed database
- **ElastiCache Redis** - Managed cache
- **ALB** - Load balancer
- **ACM** - SSL certificates

### GCP (Cloud Run)

Deploy containers to Cloud Run with:
- **Cloud SQL PostgreSQL**
- **Memorystore Redis**
- **Cloud Load Balancing**
- **Google-managed certificates**

### Azure (Container Instances)

Deploy with:
- **Azure Database for PostgreSQL**
- **Azure Cache for Redis**
- **Application Gateway**
- **Azure-managed certificates**

---

## Security Checklist

### Pre-Deployment

- [ ] Change all default passwords
- [ ] Generate secure JWT/CSRF secrets (min 64 chars)
- [ ] Enable TLS/SSL for all endpoints
- [ ] Configure firewall rules (whitelist only required ports)
- [ ] Enable audit logging
- [ ] Set up backup strategy for PostgreSQL
- [ ] Configure Redis persistence
- [ ] Review CORS allowed origins

### Post-Deployment

- [ ] Force password change for default admin account
- [ ] Enable 2FA for admin users
- [ ] Configure rate limiting
- [ ] Set up monitoring and alerting
- [ ] Enable database encryption at rest
- [ ] Configure automated backups
- [ ] Set up log aggregation (ELK/Loki)
- [ ] Enable metrics collection (Prometheus)

---

## Monitoring & Observability

### Health Checks

All services expose health endpoints:

```bash
# API
curl https://app.rediver.io/api/health

# UI (via proxy)
curl https://app.rediver.io/api/health
```

### Metrics (Prometheus)

API exposes metrics at `/metrics`:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'rediver-api'
    static_configs:
      - targets: ['rediver-api:8080']
```

### Logging

Configure structured JSON logging:

```yaml
# API environment
LOG_LEVEL: info
LOG_FORMAT: json
```

Ship logs to:
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Loki + Grafana**
- **Cloud provider logs** (CloudWatch, Stackdriver, Azure Monitor)

---

## Scaling

### Horizontal Pod Autoscaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rediver-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rediver-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Database Scaling

- **Read replicas** for PostgreSQL
- **Connection pooling** with PgBouncer
- **Redis Cluster** for high availability

---

## Backup Strategy

### PostgreSQL Backups

```bash
# Daily automated backups
kubectl create cronjob rediver-backup \
  --image=postgres:17 \
  --schedule="0 2 * * *" \
  -- pg_dump -h postgres -U rediver rediver | gzip > /backups/backup-$(date +%Y%m%d).sql.gz
```

### Retention Policy

- **Daily backups:** 7 days
- **Weekly backups:** 4 weeks
- **Monthly backups:** 12 months

---

## Disaster Recovery

### RTO/RPO Targets

- **RTO (Recovery Time Objective):** < 1 hour
- **RPO (Recovery Point Objective):** < 15 minutes

### Recovery Procedure

1. Restore PostgreSQL from latest backup
2. Redeploy application containers
3. Run health checks
4. Verify data integrity
5. Resume normal operations

---

## Troubleshooting

### Pods in CrashLoopBackOff

```bash
# Check logs
kubectl logs -l app=rediver-api --namespace rediver --tail=100

# Common causes:
# - Database not ready
# - Missing secrets
# - Migration failure
```

### Database Connection Issues

```bash
# Test connection from API pod
kubectl exec -it deployment/rediver-api --namespace rediver -- sh
pg_isready -h postgres -U rediver
```

### UI Not Loading

```bash
# Check API is reachable from UI
kubectl exec -it deployment/rediver-ui --namespace rediver -- curl http://rediver-api:8080/health
```

---

## Next Steps

### Further Reading

- **[Security Best Practices](../guides/SECURITY.md)** - Authentication, secrets management, access control
- **[Monitoring & Alerting](./MONITORING.md)** - Prometheus, Grafana, Loki, SLIs/SLOs
- **[Scaling Guide](./SCALING.md)** - Performance optimization (Coming in future release)

### Related Guides

- [Architecture Overview](../architecture/overview.md)
- [End-to-End Workflow](../guides/END_TO_END_WORKFLOW.md)
- [Agent Quick Start](../../agent/docs/QUICK_START.md)

---

**Your platform is production-ready! 🚀**
