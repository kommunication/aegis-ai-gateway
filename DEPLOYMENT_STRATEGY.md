# AEGIS AI Gateway - Comprehensive Deployment Strategy

**Version**: 1.0  
**Last Updated**: March 2026  
**Target**: Production-Ready Enterprise Deployments  

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Deployment Options Matrix](#deployment-options-matrix)
3. [AWS Deployment Options](#aws-deployment-options)
4. [GCP Deployment Options](#gcp-deployment-options)
5. [Azure Deployment Options](#azure-deployment-options)
6. [On-Premise Deployment](#on-premise-deployment)
7. [Hybrid Cloud Scenarios](#hybrid-cloud-scenarios)
8. [Reference Architectures](#reference-architectures)
9. [High Availability Architecture](#high-availability-architecture)
10. [CI/CD Pipeline](#cicd-pipeline)
11. [Security Hardening](#security-hardening)
12. [Cost Optimization](#cost-optimization)
13. [Migration Paths](#migration-paths)
14. [Decision Framework](#decision-framework)

---

## Executive Summary

AEGIS AI Gateway is a production-grade AI request proxy built with Go, PostgreSQL, and Redis. This document provides comprehensive deployment strategies for organizations at different scales:

- **Small (Startup)**: <10M requests/month, ~$500-2,000/month
- **Medium (Growth)**: 10M-100M requests/month, ~$2,000-15,000/month
- **Large (Enterprise)**: 100M-1B+ requests/month, ~$15,000-100,000+/month

### Key Capabilities

- Multi-provider routing (OpenAI, Anthropic, Azure OpenAI, vLLM)
- Enterprise-grade authentication & authorization
- Content filtering (PII detection, secrets scanning, injection detection)
- Real-time cost tracking and analytics
- Prometheus metrics & observability
- Streaming SSE support
- Circuit breakers & failover
- Classification-based access control

### Technology Stack

- **Runtime**: Go 1.25+ (single binary, ~50MB)
- **Database**: PostgreSQL 16+ (auth, audit logs, usage records)
- **Cache**: Redis 7+ (auth cache, rate limiting)
- **Optional**: gRPC filter service (Python, spaCy NLP)

---

## Deployment Options Matrix

Quick comparison of all deployment options:

| Platform | Best For | Complexity | Time to Deploy | HA Native | Cost (10M req/mo) | Cost (100M req/mo) | Cost (1B req/mo) |
|----------|----------|------------|----------------|-----------|-------------------|--------------------|--------------------|
| **AWS ECS Fargate** | Startups, rapid iteration | Low | 2-4 hours | Yes | $600-800 | $3,000-4,500 | $25,000-35,000 |
| **AWS EKS** | Growth companies, K8s expertise | High | 1-2 days | Yes | $800-1,200 | $4,000-7,000 | $30,000-50,000 |
| **AWS Lambda** | Variable load, cost-sensitive | Medium | 4-6 hours | Yes | $400-600 | $2,500-4,000 | $20,000-35,000 |
| **GCP Cloud Run** | Startups, serverless-first | Very Low | 1-3 hours | Yes | $500-700 | $2,800-4,200 | $22,000-32,000 |
| **GCP GKE** | Multi-cloud, enterprise | High | 1-2 days | Yes | $750-1,100 | $3,800-6,500 | $28,000-48,000 |
| **Azure Container Apps** | Azure-first orgs | Low | 2-4 hours | Yes | $550-750 | $3,200-4,800 | $24,000-36,000 |
| **Azure AKS** | Enterprise, compliance | High | 1-2 days | Yes | $800-1,150 | $4,200-7,200 | $32,000-52,000 |
| **On-Premise (VMs)** | Data sovereignty, compliance | Medium | 1-3 days | Manual | $300-500* | $1,500-3,000* | $8,000-20,000* |
| **On-Premise (Bare Metal)** | Maximum performance | High | 3-5 days | Manual | $200-400* | $1,000-2,500* | $5,000-15,000* |
| **Hybrid Cloud** | Gradual migration, DR | Very High | 1-2 weeks | Complex | $1,000-1,500 | $5,000-10,000 | $40,000-70,000 |

*On-premise costs exclude initial hardware CapEx and assume amortized costs.

---

## AWS Deployment Options

### Option 1: AWS ECS Fargate (Recommended for Most)

**Best for**: Startups to mid-size companies, teams without K8s expertise, rapid iteration.

#### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Route 53 (DNS)                                            │ │
│  └───────────────┬────────────────────────────────────────────┘ │
│                  │                                               │
│  ┌───────────────▼────────────────────────────────────────────┐ │
│  │  CloudFront CDN (Optional, for static assets)             │ │
│  └───────────────┬────────────────────────────────────────────┘ │
│                  │                                               │
│  ┌───────────────▼────────────────────────────────────────────┐ │
│  │  Application Load Balancer (ALB)                           │ │
│  │  - Health checks: /aegis/v1/health                         │ │
│  │  - SSL termination (ACM certificate)                       │ │
│  │  - WAF attached (rate limiting, geo-blocking)              │ │
│  └───────┬────────────────────────────────┬───────────────────┘ │
│          │                                 │                     │
│  ┌───────▼─────────┐             ┌────────▼──────────┐          │
│  │  Target Group 1 │             │  Target Group 2   │          │
│  │  us-east-1a     │             │  us-east-1b       │          │
│  └───────┬─────────┘             └────────┬──────────┘          │
│          │                                 │                     │
│  ┌───────▼─────────────────────────────────▼───────────────┐    │
│  │           ECS Fargate Cluster                            │    │
│  │  ┌─────────────────┐     ┌─────────────────┐            │    │
│  │  │  Task (AZ-1a)   │     │  Task (AZ-1b)   │            │    │
│  │  │ ┌─────────────┐ │     │ ┌─────────────┐ │            │    │
│  │  │ │  Gateway     │ │     │ │  Gateway     │ │            │    │
│  │  │ │  Container   │ │     │ │  Container   │ │            │    │
│  │  │ │  (0.5 vCPU)  │ │     │ │  (0.5 vCPU)  │ │            │    │
│  │  │ │  (1GB RAM)   │ │     │ │  (1GB RAM)   │ │            │    │
│  │  │ └─────────────┘ │     │ └─────────────┘ │            │    │
│  │  └─────────────────┘     └─────────────────┘            │    │
│  │  Auto-scaling: 2-20 tasks (CPU/Memory based)            │    │
│  └──────────────────────────────────────────────────────────┘    │
│                          │                 │                     │
│  ┌───────────────────────▼─────────────────▼───────────────┐    │
│  │  ElastiCache for Redis (Cluster Mode)                   │    │
│  │  - Primary: us-east-1a                                  │    │
│  │  - Replica: us-east-1b                                  │    │
│  │  - Node type: cache.r7g.large (2 vCPU, 13.07 GB)       │    │
│  │  - Encryption in-transit & at-rest                      │    │
│  └──────────────────────────────────────────────────────────┘    │
│                          │                                       │
│  ┌───────────────────────▼───────────────────────────────────┐  │
│  │  RDS PostgreSQL (Multi-AZ)                                │  │
│  │  - Primary: us-east-1a                                    │  │
│  │  - Standby: us-east-1b (sync replication)                │  │
│  │  - Instance: db.t4g.medium (2 vCPU, 4GB)                 │  │
│  │  - Storage: 100GB gp3 SSD (auto-scaling to 1TB)          │  │
│  │  - Backup: 7-day retention, point-in-time recovery       │  │
│  │  - Encryption: KMS-managed                                │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Secrets Manager                                          │   │
│  │  - Database credentials                                   │   │
│  │  - Provider API keys (OpenAI, Anthropic, Azure)          │   │
│  │  - Auto-rotation: 30 days                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudWatch Logs + Metrics                                │   │
│  │  - Log groups: /ecs/aegis-gateway                         │   │
│  │  - Retention: 30 days                                     │   │
│  │  - Custom metrics from Prometheus exporter               │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

#### Infrastructure Requirements

**Compute (ECS Fargate)**:
- Service: 2-20 tasks (auto-scaling)
- Task size: 0.5 vCPU, 1GB RAM per task (small), 2 vCPU, 4GB RAM (medium), 4 vCPU, 8GB RAM (large)
- Network mode: awsvpc
- Platform version: LATEST (1.4.0+)

**Database (RDS PostgreSQL)**:
- Small: db.t4g.medium (2 vCPU, 4GB), 100GB gp3
- Medium: db.r6g.large (2 vCPU, 16GB), 500GB gp3
- Large: db.r6g.2xlarge (8 vCPU, 64GB), 2TB gp3 or io2
- Multi-AZ: Required for production
- Backup: Automated daily, 7-30 day retention

**Cache (ElastiCache Redis)**:
- Small: cache.t4g.medium (2 vCPU, 3.09GB), 1 primary + 1 replica
- Medium: cache.r7g.large (2 vCPU, 13.07GB), 1 primary + 2 replicas
- Large: cache.r7g.xlarge (4 vCPU, 26.32GB), 2 shards, 2 replicas each
- Cluster mode: Enabled for >100M req/mo
- Encryption: In-transit (TLS) + at-rest (KMS)

**Networking**:
- VPC: Dedicated VPC with public + private subnets across 2-3 AZs
- ALB: Application Load Balancer in public subnets
- ECS Tasks: Private subnets with NAT Gateway for egress
- Security Groups:
  - ALB: Inbound 443 from 0.0.0.0/0, outbound to ECS tasks
  - ECS Tasks: Inbound 8080 from ALB, outbound HTTPS to providers
  - RDS: Inbound 5432 from ECS tasks only
  - Redis: Inbound 6379 from ECS tasks only

#### Scaling Strategy

**Horizontal Scaling (Tasks)**:
```yaml
# Auto-scaling policy
Target Tracking:
  - Metric: ECSServiceAverageCPUUtilization
    Target: 70%
  - Metric: ECSServiceAverageMemoryUtilization
    Target: 80%
  - Metric: ALBRequestCountPerTarget
    Target: 1000 requests/minute/task

Step Scaling:
  - CPU > 85%: Add 2 tasks (max 20)
  - CPU < 30% for 10min: Remove 1 task (min 2)
  - Memory > 90%: Add 2 tasks immediately
```

**Vertical Scaling (Task Size)**:
- Monitor per-task metrics over 7 days
- Increase vCPU if CPU > 80% sustained
- Increase RAM if memory > 85% sustained
- Start with 0.5 vCPU, 1GB for <1M req/day
- Use 2 vCPU, 4GB for 1M-10M req/day
- Use 4 vCPU, 8GB for >10M req/day

**Database Scaling**:
- Monitor RDS CloudWatch metrics
- Scale vertically: Upgrade instance class during maintenance window
- Read replicas: Add 1-2 read replicas for analytics/reporting queries
- Storage auto-scaling: Enable with max 1-5TB based on growth rate

#### Cost Estimates

**Small Deployment (<10M req/mo)**:
```
ECS Fargate:
  - 2 tasks × 0.5 vCPU × $0.04048/vCPU/hr × 730hr = $59
  - 2 tasks × 1GB × $0.004445/GB/hr × 730hr = $6.49
  
RDS PostgreSQL (db.t4g.medium Multi-AZ):
  - Instance: $0.136/hr × 730hr × 2 (Multi-AZ) = $198
  - Storage: 100GB × $0.138/GB = $13.80
  
ElastiCache Redis (cache.t4g.medium):
  - Primary: $0.068/hr × 730hr = $49.64
  - Replica: $0.068/hr × 730hr = $49.64
  
ALB:
  - Base: $0.0225/hr × 730hr = $16.43
  - LCU: ~5 LCU × $0.008/LCU/hr × 730hr = $29.20
  
NAT Gateway:
  - 1 AZ: $0.045/hr × 730hr = $32.85
  - Data: 100GB × $0.045/GB = $4.50
  
Data Transfer:
  - Out to internet: 500GB × $0.09/GB = $45
  
CloudWatch Logs:
  - Ingestion: 50GB × $0.50/GB = $25
  - Storage: 50GB × $0.03/GB = $1.50

Secrets Manager:
  - 10 secrets × $0.40/secret/mo = $4

Total: ~$535/month
```

**Medium Deployment (50M req/mo)**:
```
ECS Fargate:
  - 8 tasks × 2 vCPU × $0.04048/vCPU/hr × 730hr = $473
  - 8 tasks × 4GB × $0.004445/GB/hr × 730hr = $104
  
RDS PostgreSQL (db.r6g.large Multi-AZ):
  - Instance: $0.384/hr × 730hr × 2 = $560
  - Storage: 500GB × $0.138/GB = $69
  
ElastiCache Redis (cache.r7g.large):
  - Primary: $0.226/hr × 730hr = $165
  - Replicas: 2 × $0.226/hr × 730hr = $330
  
ALB:
  - Base: $16.43
  - LCU: ~20 LCU × $0.008/LCU/hr × 730hr = $116.80
  
NAT Gateway (2 AZ):
  - 2 × $32.85 = $65.70
  - Data: 500GB × $0.045/GB = $22.50
  
Data Transfer:
  - Out: 2TB × $0.09/GB = $184
  
CloudWatch:
  - Ingestion: 200GB × $0.50/GB = $100
  - Storage: 200GB × $0.03/GB = $6

Reserved Instances Savings (-30%):
  - RDS RI: -$168
  - ElastiCache RI: -$148

Total: ~$1,895/month (with RIs: ~$1,579/month)
```

**Large Deployment (500M req/mo)**:
```
ECS Fargate:
  - 40 tasks × 4 vCPU × $0.04048/vCPU/hr × 730hr = $4,732
  - 40 tasks × 8GB × $0.004445/GB/hr × 730hr = $1,036
  
RDS PostgreSQL (db.r6g.4xlarge Multi-AZ):
  - Instance: $1.536/hr × 730hr × 2 = $2,243
  - Storage: 2TB × $0.138/GB = $282
  - IOPS: 20,000 provisioned IOPS × $0.20 = $400
  
ElastiCache Redis (Cluster mode, 4 shards):
  - 4 shards × 2 nodes × cache.r7g.xlarge
  - 8 nodes × $0.452/hr × 730hr = $2,639
  
ALB:
  - Base: $16.43
  - LCU: ~100 LCU × $0.008/LCU/hr × 730hr = $584
  
NAT Gateway (3 AZ):
  - 3 × $32.85 = $98.55
  - Data: 2TB × $0.045/GB = $92.16
  
Data Transfer:
  - Out: 10TB × $0.09/GB (first 10TB tier) = $922
  
CloudWatch:
  - Ingestion: 1TB × $0.50/GB = $512
  - Storage: 1TB × $0.03/GB = $30.72

Reserved Instances Savings (-40% for 3yr):
  - RDS RI: -$1,090
  - ElastiCache RI: -$1,055

Total: ~$13,496/month (with RIs: ~$11,351/month)
```

#### Pros & Cons

**Pros**:
✅ No infrastructure management (serverless containers)  
✅ Fast deployment (hours, not days)  
✅ Auto-scaling built-in  
✅ Integrates seamlessly with AWS ecosystem  
✅ Lower operational overhead than EKS  
✅ Pay-per-use pricing model  
✅ Multi-AZ by default  

**Cons**:
❌ Less control than EKS/EC2  
❌ Cold start potential (mitigated with min task count)  
❌ Limited to AWS (vendor lock-in)  
❌ More expensive than EC2 spot for sustained high load  
❌ Task size constraints (max 4 vCPU, 30GB)  

#### Security Considerations

**Network Isolation**:
- VPC with private subnets for ECS tasks
- Security groups with least-privilege rules
- No public IPs on tasks (egress via NAT Gateway)
- VPC endpoints for AWS services (S3, ECR, Secrets Manager, CloudWatch)

**Secrets Management**:
- AWS Secrets Manager for API keys and DB credentials
- IAM roles for ECS tasks (no hardcoded credentials)
- Automatic secret rotation (30-90 days)
- Encryption at rest with KMS customer-managed keys

**TLS/SSL**:
- ACM certificate on ALB (free, auto-renewal)
- TLS 1.2+ only, strong cipher suites
- HSTS headers enabled
- Internal communication: Redis TLS, RDS SSL enforced

**IAM Policies**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:aegis/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:*:key/aegis-secrets-key"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:*:log-group:/ecs/aegis-gateway:*"
    }
  ]
}
```

**WAF Rules**:
- Rate limiting: 2000 req/5min per IP (adjustable)
- Geo-blocking: Block requests from high-risk countries
- SQL injection protection
- XSS protection
- Known bad inputs (AWS Managed Rules)

#### Monitoring & Observability

**CloudWatch Metrics**:
- ECS task CPU/memory utilization
- ALB request count, latency, 4xx/5xx errors
- RDS CPU, connections, IOPS, replication lag
- Redis CPU, memory, cache hit rate
- Custom metrics from Prometheus exporter (via CloudWatch agent)

**CloudWatch Alarms**:
```yaml
Critical:
  - ECS tasks all unhealthy (notify: PagerDuty)
  - RDS replica lag > 60 seconds
  - Redis primary failover
  - ALB 5xx rate > 5%

Warning:
  - ECS CPU > 80% for 10 minutes
  - RDS storage < 20% free
  - Redis memory > 85%
  - ALB latency p99 > 2s
```

**Logging**:
- Application logs: CloudWatch Logs with JSON structured logging
- Access logs: ALB access logs to S3
- VPC Flow Logs: Network traffic analysis
- Retention: 30 days (production), 7 days (dev)

**Distributed Tracing** (Optional):
- AWS X-Ray integration
- Trace request flow: ALB → ECS → RDS/Redis → External API
- Identify bottlenecks and latency sources

#### Disaster Recovery

**RTO (Recovery Time Objective)**: < 15 minutes  
**RPO (Recovery Point Objective)**: < 5 minutes

**Backup Strategy**:
- RDS automated backups: Daily, 7-day retention
- RDS manual snapshots: Weekly, retained 30 days
- Redis backups: Daily snapshots to S3
- Infrastructure-as-Code: Terraform/CloudFormation in git

**Failover Procedures**:
1. **RDS Primary Failure**: 
   - Automatic failover to standby (1-2 minutes)
   - Update DNS if using Route 53 health checks
   
2. **Redis Primary Failure**:
   - ElastiCache promotes replica automatically (< 1 minute)
   - Application reconnects automatically
   
3. **AZ Failure**:
   - ALB routes traffic to healthy AZ
   - ECS auto-scales tasks in remaining AZ(s)
   
4. **Region Failure** (requires multi-region setup):
   - Promote secondary region RDS read replica
   - Update Route 53 to failover to secondary region
   - Restore Redis from S3 snapshot

**Recovery Procedures**:
```bash
# Restore from RDS snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier aegis-db-restored \
  --db-snapshot-identifier aegis-db-snapshot-2026-03-25

# Restore Redis from backup
aws elasticache create-cache-cluster \
  --cache-cluster-id aegis-redis-restored \
  --snapshot-name aegis-redis-backup-2026-03-25
```

---

### Option 2: AWS EKS (Kubernetes)

**Best for**: Growth-stage companies with K8s expertise, multi-cloud future, complex microservices.

#### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                     │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Route 53 + External DNS                                        │ │
│  └───────────────┬─────────────────────────────────────────────────┘ │
│                  │                                                    │
│  ┌───────────────▼─────────────────────────────────────────────────┐ │
│  │  AWS Load Balancer Controller (ALB Ingress)                     │ │
│  │  - SSL termination (ACM)                                        │ │
│  │  - WAF integration                                              │ │
│  └───────────────┬─────────────────────────────────────────────────┘ │
│                  │                                                    │
│  ┌───────────────▼─────────────────────────────────────────────────┐ │
│  │              EKS Cluster (Kubernetes 1.30+)                      │ │
│  │                                                                  │ │
│  │  ┌────────────────────────────────────────────────────────────┐ │ │
│  │  │  Ingress NGINX (or AWS LB Controller)                      │ │ │
│  │  │  - Path-based routing: /v1/* → gateway-service             │ │ │
│  │  │  - Rate limiting (optional, also in app)                   │ │ │
│  │  └────────────┬───────────────────────────────────────────────┘ │ │
│  │               │                                                  │ │
│  │  ┌────────────▼───────────────────────────────────────────────┐ │ │
│  │  │  aegis-gateway Service (ClusterIP)                         │ │ │
│  │  └────────────┬───────────────────────────────────────────────┘ │ │
│  │               │                                                  │ │
│  │  ┌────────────▼───────────────────────────────────────────────┐ │ │
│  │  │  aegis-gateway Deployment                                  │ │ │
│  │  │  - Replicas: 3-50 (HPA)                                    │ │ │
│  │  │  - Resources: 500m CPU, 1Gi RAM (requests)                 │ │ │
│  │  │               2 CPU, 4Gi RAM (limits)                      │ │ │
│  │  │  - Pod Anti-Affinity: Spread across nodes/AZs             │ │ │
│  │  │  - PodDisruptionBudget: maxUnavailable=1                   │ │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                │ │ │
│  │  │  │  Pod 1   │  │  Pod 2   │  │  Pod 3   │                │ │ │
│  │  │  │  AZ-1a   │  │  AZ-1b   │  │  AZ-1c   │   ...          │ │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘                │ │ │
│  │  └────────────────────────────────────────────────────────────┘ │ │
│  │                                                                  │ │
│  │  ┌──────────────────────────────────────────────────────────┐  │ │
│  │  │  Node Group (Managed, Auto-scaling)                       │  │ │
│  │  │  - Instance type: t3.large (2 vCPU, 8GB)                  │  │ │
│  │  │  - Min: 3 nodes (1 per AZ)                                │  │ │
│  │  │  - Max: 20 nodes                                          │  │ │
│  │  │  - Spot instances: 50% mix for cost savings               │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  │                                                                  │ │
│  │  ┌──────────────────────────────────────────────────────────┐  │ │
│  │  │  Supporting Services                                       │  │ │
│  │  │  - Prometheus Operator (monitoring)                        │  │ │
│  │  │  - Grafana (dashboards)                                    │  │ │
│  │  │  - Fluent Bit (log shipping to CloudWatch)                │  │ │
│  │  │  - Cluster Autoscaler                                      │  │ │
│  │  │  - External Secrets Operator (Secrets Manager sync)       │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                          │                 │                          │
│  ┌───────────────────────▼─────────────────▼─────────────────────┐   │
│  │  ElastiCache Redis (same as ECS option)                       │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                          │                                            │
│  ┌───────────────────────▼────────────────────────────────────────┐  │
│  │  RDS PostgreSQL (same as ECS option)                           │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

#### Infrastructure Requirements

**EKS Control Plane**:
- Version: 1.30+ (managed by AWS)
- Cost: $0.10/hour ($73/month) per cluster
- Endpoint: Public + Private (recommended)

**Node Groups**:
- Small: 3-6 × t3.large (2 vCPU, 8GB), on-demand
- Medium: 6-12 × t3.xlarge (4 vCPU, 16GB), 50% spot, 50% on-demand
- Large: 12-30 × c6i.2xlarge (8 vCPU, 16GB), 70% spot, 30% on-demand
- AMI: Amazon EKS-optimized AMI (AL2023)
- Storage: 100GB gp3 EBS per node

**Kubernetes Resources**:
```yaml
# Deployment spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aegis-gateway
  namespace: aegis
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: gateway
        image: YOUR_ECR_REPO/aegis-gateway:v1.2.3
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: aegis-db-credentials
              key: host
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: aegis-config
              key: redis_host
        livenessProbe:
          httpGet:
            path: /aegis/v1/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /aegis/v1/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - aegis-gateway
              topologyKey: topology.kubernetes.io/zone

---
# HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: aegis-gateway-hpa
  namespace: aegis
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aegis-gateway
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60

---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: aegis-gateway-pdb
  namespace: aegis
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: aegis-gateway
```

#### Scaling Strategy

**Horizontal Pod Autoscaling**:
- Metric-based: CPU (70%), Memory (80%), Custom (requests/sec)
- Scale up aggressively (50% increase per minute)
- Scale down conservatively (1 pod per minute after 5min stabilization)
- Min replicas: 3 (production), Max: 50+

**Cluster Autoscaler**:
- Automatically adds/removes nodes based on pending pods
- Scale-up when pods are unschedulable for 30 seconds
- Scale-down when node utilization < 50% for 10 minutes
- Respects PodDisruptionBudgets

**Vertical Pod Autoscaling** (Optional):
- Use VPA in "recommend" mode initially
- Adjust resource requests/limits based on actual usage
- Apply during deployment updates (not live)

#### Cost Estimates

**Small Deployment (<10M req/mo)**:
```
EKS Control Plane: $73/month

Node Group (3 × t3.large on-demand):
  - Compute: 3 × $0.0832/hr × 730hr = $182.21
  - EBS: 3 × 100GB × $0.08/GB = $24
  
RDS + Redis + ALB: ~$400 (same as ECS)

Data Transfer: ~$50

Total: ~$730/month
```

**Medium Deployment (50M req/mo)**:
```
EKS Control Plane: $73/month

Node Group (8 × t3.xlarge, 50% spot):
  - On-demand: 4 × $0.1664/hr × 730hr = $486
  - Spot: 4 × $0.05/hr × 730hr = $146
  - EBS: 8 × 100GB × $0.08/GB = $64
  
RDS + Redis: ~$1,100

ALB + NAT: ~$200

Data Transfer: ~$200

Total: ~$2,269/month
```

**Large Deployment (500M req/mo)**:
```
EKS Control Plane: $73/month

Node Group (20 × c6i.2xlarge, 70% spot):
  - On-demand: 6 × $0.34/hr × 730hr = $1,490
  - Spot: 14 × $0.10/hr × 730hr = $1,022
  - EBS: 20 × 100GB × $0.08/GB = $160
  
RDS + Redis: ~$5,000

ALB + NAT: ~$700

Data Transfer: ~$1,000

Total: ~$9,445/month
```

#### Pros & Cons

**Pros**:
✅ Maximum flexibility and control  
✅ Multi-cloud portability (K8s standard)  
✅ Advanced deployment strategies (canary, blue/green)  
✅ Rich ecosystem (Helm charts, operators)  
✅ Spot instance support for cost savings  
✅ Fine-grained resource management  
✅ Best for microservices architectures  

**Cons**:
❌ High operational complexity  
❌ Requires K8s expertise  
❌ Longer setup time (days vs hours)  
❌ More moving parts to manage  
❌ Higher base cost (control plane + nodes)  
❌ Steeper learning curve  

#### Security Considerations

**RBAC (Role-Based Access Control)**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: aegis
  name: aegis-operator
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"] # Read-only secrets
```

**Network Policies**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: aegis-gateway-netpol
  namespace: aegis
spec:
  podSelector:
    matchLabels:
      app: aegis-gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443 # HTTPS to external APIs
```

**Pod Security Standards**:
- Enforce "restricted" policy for all workloads
- Non-root containers (UID > 1000)
- Read-only root filesystem
- Drop all capabilities, add only NET_BIND_SERVICE if needed
- No privileged escalation

**Secrets Management**:
- External Secrets Operator syncs from AWS Secrets Manager
- Secrets encrypted at rest in etcd (KMS)
- Short-lived service account tokens (projected volumes)

#### Monitoring & Observability

**Prometheus Stack**:
```yaml
# Install kube-prometheus-stack via Helm
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi
```

**ServiceMonitor for AEGIS**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: aegis-gateway
  namespace: aegis
spec:
  selector:
    matchLabels:
      app: aegis-gateway
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

**Grafana Dashboards**:
- Request rate, latency, error rate (RED metrics)
- Pod CPU/memory usage
- Database connection pool metrics
- Redis cache hit rate
- Cost per request breakdown

**Logging**:
- Fluent Bit DaemonSet ships logs to CloudWatch
- Structured JSON logging from application
- Index logs by namespace, pod, container

#### Disaster Recovery

**Backup Strategy**:
- etcd snapshots: Daily (managed by AWS for EKS)
- Application state: RDS/Redis backups (same as ECS)
- Cluster configuration: GitOps (ArgoCD/FluxCD) or Helm charts in git
- Persistent volumes: EBS snapshots (if used)

**Cluster Recovery**:
```bash
# Recreate EKS cluster from IaC
terraform apply

# Restore applications from GitOps repo
kubectl apply -k overlays/production

# Restore RDS from snapshot (same as ECS)
```

---

### Option 3: AWS Lambda (Serverless)

**Best for**: Variable/spiky traffic, cost-sensitive startups, proof-of-concept.

#### Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                               │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  API Gateway (HTTP API or REST API)                      │  │
│  │  - Custom domain (Route 53)                              │  │
│  │  - Authorizer: Lambda function (API key validation)      │  │
│  │  - Throttling: 10,000 req/sec burst, 5,000 steady       │  │
│  │  - WAF attached                                          │  │
│  └────────────────────┬─────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼─────────────────────────────────────┐  │
│  │  Lambda Function: aegis-gateway-handler                  │  │
│  │  - Runtime: Custom runtime (Go binary)                   │  │
│  │  - Memory: 512MB-3GB (auto-tunes)                        │  │
│  │  - Timeout: 120 seconds (for streaming)                  │  │
│  │  - Concurrency: Reserved 100, Provisioned 10            │  │
│  │  - VPC: Enabled (for RDS/Redis access)                   │  │
│  │  - Layers: Shared dependencies (if any)                  │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  Lambda Instances (auto-scaled)                    │  │  │
│  │  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐              │  │  │
│  │  │  │ Inst1│ │ Inst2│ │ Inst3│ │ ...  │ (10-100+)    │  │  │
│  │  │  └──────┘ └──────┘ └──────┘ └──────┘              │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └────────────────────┬─────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼─────────────────────────────────────┐  │
│  │  ElastiCache Redis (Serverless or standard)              │  │
│  │  - Mode: Serverless (pay-per-request) or standard        │  │
│  │  - Used for: Auth cache, rate limiting                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼─────────────────────────────────────┐  │
│  │  RDS Proxy (connection pooling)                          │  │
│  │  - Max connections: 1000                                 │  │
│  │  - Idle timeout: 1800s                                   │  │
│  │  - Auth: IAM (no password in Lambda)                     │  │
│  └────────────────────┬─────────────────────────────────────┘  │
│                       │                                         │
│  ┌────────────────────▼─────────────────────────────────────┐  │
│  │  RDS PostgreSQL (Aurora Serverless v2 recommended)       │  │
│  │  - Min ACU: 0.5, Max ACU: 16                             │  │
│  │  - Auto-scaling based on load                            │  │
│  │  - Multi-AZ for production                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Secrets Manager (API keys, DB credentials)              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  CloudWatch Logs + X-Ray (monitoring & tracing)          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### Infrastructure Requirements

**Lambda Function**:
- Memory: 512MB (small), 1536MB (medium), 3008MB (large)
- Timeout: 120 seconds (API Gateway limit: 30s for REST, 30s for HTTP API v1, can use WebSocket for streaming)
- Runtime: Custom (Go 1.x binary compiled for Linux ARM64 or x86_64)
- VPC: Required for RDS/Redis access (adds cold start latency)
- Provisioned concurrency: 5-20 instances (reduces cold starts)
- Reserved concurrency: 100-1000 (prevents runaway costs)

**API Gateway**:
- Type: HTTP API (cheaper, lower latency) or REST API (more features)
- Throttling: 10,000 req/sec burst, 5,000 steady-state (adjustable)
- Caching: Optional (disabled to reduce cost, cache in Redis instead)
- Custom domain: Route 53 + ACM certificate

**RDS Proxy**:
- Required for Lambda (manages connection pooling)
- Max connections: 100-1000 depending on Lambda concurrency
- IAM authentication (no password management)

**Database**:
- **Aurora Serverless v2** (recommended): 0.5-16 ACU auto-scaling
- **RDS PostgreSQL**: db.t4g.medium minimum (not ideal for Lambda spikes)

**Redis**:
- **ElastiCache Serverless** (GA March 2024): Pay-per-request, auto-scaling
- **Standard**: cache.t4g.small minimum

#### Scaling Strategy

**Lambda Auto-Scaling**:
- Automatic: AWS scales to concurrent requests (up to account limit: 1000 default, 10,000+ with quota increase)
- Provisioned concurrency: Keep 10-20 warm instances for low latency
- Reserved concurrency: Cap at 500-1000 to prevent cost explosions

**Database Scaling**:
- Aurora Serverless v2: Auto-scales from 0.5 ACU to 16 ACU (or higher)
- Scaling triggered by CPU/connections
- Pause after 5 minutes of inactivity (optional)

**Cold Start Mitigation**:
- Use ARM64 architecture (Graviton2): 34% better price/performance
- Minimize binary size (strip debug symbols)
- Use provisioned concurrency for critical paths
- Keep Lambda functions warm with CloudWatch Events (ping every 5 min)

#### Cost Estimates

**Small Deployment (<10M req/mo)**:
```
Lambda:
  - Requests: 10M × $0.20/1M = $2
  - Compute: 10M × 512MB × 200ms avg × $0.0000166667/GB-sec
    = 10M × 0.5GB × 0.2s × $0.0000166667 = $16.67
  - Provisioned concurrency: 5 × 512MB × 730hr × $0.0000041667/GB-hr
    = 5 × 0.5 × 730 × $0.0000041667 = $7.60
  
API Gateway (HTTP API):
  - Requests: 10M × $1.00/1M = $10
  
RDS Proxy:
  - 1 proxy × $0.015/hr × 730hr = $10.95
  
Aurora Serverless v2:
  - 0.5 ACU average × $0.12/ACU/hr × 730hr = $43.80
  
ElastiCache Serverless:
  - Data: 1GB × $0.125/GB = $0.13
  - ECPUs: 5,000 × $0.0034/1000 = $0.017
  
Secrets Manager: $4

Data Transfer: $20

Total: ~$115/month
```

**Medium Deployment (50M req/mo)**:
```
Lambda:
  - Requests: 50M × $0.20/1M = $10
  - Compute: 50M × 1GB × 250ms × $0.0000166667 = $208.33
  - Provisioned: 10 × 1GB × 730hr × $0.0000041667 = $30.42
  
API Gateway: 50M × $1.00/1M = $50

RDS Proxy: $10.95

Aurora Serverless v2:
  - 2 ACU average × $0.12/ACU/hr × 730hr = $175.20
  
ElastiCache Serverless:
  - Data: 5GB × $0.125/GB = $0.63
  - ECPUs: 50,000 × $0.0034/1000 = $0.17

Data Transfer: $100

Total: ~$585/month
```

**Large Deployment (500M req/mo)**:
```
Lambda:
  - Requests: 500M × $0.20/1M = $100
  - Compute: 500M × 2GB × 300ms × $0.0000166667 = $5,000
  - Provisioned: 50 × 2GB × 730hr × $0.0000041667 = $304.17
  
API Gateway: 500M × $1.00/1M = $500

RDS Proxy (2 proxies): $21.90

Aurora Serverless v2:
  - 8 ACU average × $0.12/ACU/hr × 730hr = $700.80
  
ElastiCache Serverless:
  - Data: 20GB × $0.125/GB = $2.50
  - ECPUs: 500,000 × $0.0034/1000 = $1.70

Data Transfer: $800

Total: ~$7,431/month
```

#### Pros & Cons

**Pros**:
✅ Lowest cost for low/variable traffic  
✅ Zero infrastructure management  
✅ Infinite horizontal scaling (within account limits)  
✅ Pay-per-request (no idle cost)  
✅ Auto-scaling built-in  
✅ Fast deployment (minutes)  

**Cons**:
❌ Cold start latency (100-500ms, mitigated with provisioned concurrency)  
❌ Limited execution time (120s max)  
❌ VPC networking adds latency  
❌ Difficult to debug/troubleshoot  
❌ Vendor lock-in (AWS-specific)  
❌ Connection pooling complexity (requires RDS Proxy)  
❌ Streaming support limited (requires WebSocket or custom setup)  

#### Security Considerations

**Lambda Execution Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds-db:connect"
      ],
      "Resource": "arn:aws:rds-db:us-east-1:*:dbuser:*/aegis_app"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:aegis/*"
    }
  ]
}
```

**API Gateway Authorizer**:
- Lambda authorizer validates API keys
- Returns IAM policy (allow/deny)
- Caches authorization decisions (TTL: 300s)

**Network Security**:
- Lambda in private VPC subnets
- Security group allows outbound HTTPS only
- RDS Proxy in same VPC, different subnet
- No public IPs on Lambda

#### Monitoring & Observability

**CloudWatch Metrics**:
- Lambda invocations, errors, duration, throttles
- API Gateway 4xx/5xx, latency, cache hit/miss
- RDS Proxy connections, connection borrow time
- Aurora Serverless ACU utilization

**X-Ray Tracing**:
- Trace full request path: API Gateway → Lambda → RDS Proxy → Aurora
- Identify bottlenecks (cold start, DB query, external API)
- Segment timing breakdown

**CloudWatch Logs Insights**:
```sql
-- Average request duration by endpoint
fields @timestamp, @message
| filter @message like /request_duration/
| stats avg(duration_ms) as avg_duration by endpoint
| sort avg_duration desc

-- Error rate by hour
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() as errors by bin(1h)
```

#### Disaster Recovery

**Backup Strategy**:
- Aurora automated backups: Continuous (point-in-time restore)
- Function code: Stored in S3 (versioned)
- IaC: Terraform/CloudFormation in git

**Recovery**:
- Deploy new Lambda version from S3
- Restore Aurora from automated backup
- Update API Gateway to point to new Lambda

**Multi-Region Failover** (Advanced):
- Route 53 health checks on API Gateway endpoint
- Failover to secondary region (Lambda + Aurora global database)
- RPO: ~1 second (Aurora global database lag)
- RTO: ~5 minutes (DNS propagation + Aurora promotion)

---

## GCP Deployment Options

### Option 1: GCP Cloud Run (Recommended for Most)

**Best for**: Startups, rapid iteration, serverless-first teams, similar to AWS Fargate.

#### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                         Google Cloud                              │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Cloud DNS + Cloud CDN (optional)                          │  │
│  └───────────────┬────────────────────────────────────────────┘  │
│                  │                                                │
│  ┌───────────────▼────────────────────────────────────────────┐  │
│  │  Cloud Load Balancer (Global HTTPS LB)                     │  │
│  │  - SSL certificate (Google-managed or custom)              │  │
│  │  - Cloud Armor (WAF) attached                              │  │
│  │  - Backend: Cloud Run service                              │  │
│  └───────────────┬────────────────────────────────────────────┘  │
│                  │                                                │
│  ┌───────────────▼────────────────────────────────────────────┐  │
│  │  Cloud Run Service: aegis-gateway                          │  │
│  │  - Region: us-central1 (or multi-region)                   │  │
│  │  - Min instances: 1-3 (avoid cold starts)                  │  │
│  │  - Max instances: 100                                      │  │
│  │  - CPU: 1-4 vCPU per instance                              │  │
│  │  - Memory: 2-8 GiB per instance                            │  │
│  │  - Concurrency: 80 requests per instance                   │  │
│  │  - Timeout: 3600s (max)                                    │  │
│  │  - Autoscaling: CPU (70%) + request count                  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  Instances (auto-scaled, serverless)                 │  │  │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐                   │  │  │
│  │  │  │  Inst1 │ │  Inst2 │ │  Inst3 │  ...              │  │  │
│  │  │  └────────┘ └────────┘ └────────┘                   │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────┬───────────────────────────────────────┘  │
│                       │                                           │
│  ┌────────────────────▼───────────────────────────────────────┐  │
│  │  Memorystore for Redis (Managed Redis)                     │  │
│  │  - Tier: Standard (HA, auto-failover)                      │  │
│  │  - Memory: 1-5 GB (small), 10-50 GB (medium), 100+ GB (large)│
│  │  - Version: 7.0                                             │  │
│  │  - VPC peering: Connect to Cloud Run via Serverless VPC    │  │
│  │  - Replica: Read replicas for high read throughput         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                       │                                           │
│  ┌────────────────────▼───────────────────────────────────────┐  │
│  │  Cloud SQL for PostgreSQL (Managed PostgreSQL)             │  │
│  │  - Version: PostgreSQL 16                                  │  │
│  │  - Machine: db-custom-2-8192 (2 vCPU, 8GB) - small         │  │
│  │             db-custom-4-16384 (4 vCPU, 16GB) - medium      │  │
│  │             db-custom-8-32768 (8 vCPU, 32GB) - large       │  │
│  │  - Storage: 100-2000 GB SSD (auto-resize enabled)          │  │
│  │  - HA: Regional (automatic failover)                       │  │
│  │  - Backups: Automated daily, 7-day retention               │  │
│  │  - Encryption: Customer-managed keys (CMEK) optional       │  │
│  │  - Connection: Private IP (VPC) + Cloud SQL Proxy          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Secret Manager (API keys, DB credentials)                 │  │
│  │  - Secrets: openai-api-key, anthropic-api-key, db-password │  │
│  │  - Versioning: Enabled                                     │  │
│  │  - Rotation: Manual (can automate with Cloud Functions)    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Cloud Logging + Cloud Monitoring                           │  │
│  │  - Logs: Application logs, access logs                     │  │
│  │  - Metrics: Request count, latency, error rate             │  │
│  │  - Alerts: SLO-based (99.9% availability, p99 < 1s)        │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

#### Infrastructure Requirements

**Cloud Run Service**:
- CPU: 1-4 vCPU per instance (allocated only during request handling)
- Memory: 2-8 GiB per instance
- Concurrency: 80 (default), 1-1000 (tunable)
- Min instances: 1-3 (warm, reduces cold start)
- Max instances: 100-1000
- Timeout: 3600s (1 hour max)
- Execution environment: gen2 (faster cold starts)

**Cloud SQL (PostgreSQL)**:
- Small: db-custom-2-8192 (2 vCPU, 8GB), 100GB SSD
- Medium: db-custom-4-16384 (4 vCPU, 16GB), 500GB SSD
- Large: db-custom-8-32768 (8 vCPU, 32GB), 2TB SSD
- HA: Regional (primary + standby in different zones)
- Connection: Cloud SQL Proxy (encrypted, no IP whitelisting)

**Memorystore (Redis)**:
- Small: 1-5 GB, Standard tier (HA)
- Medium: 10-50 GB, Standard tier
- Large: 100-300 GB, Standard tier with read replicas
- Version: 7.0
- Tier: Standard (auto-failover), not Basic (single instance)

**Networking**:
- Serverless VPC Access connector (Cloud Run → Cloud SQL/Redis)
- Cloud Armor (WAF) on Load Balancer
- Private IP for Cloud SQL and Memorystore
- Cloud NAT for Cloud Run egress (optional)

#### Scaling Strategy

**Cloud Run Auto-Scaling**:
- Metric: CPU utilization (70%) + request concurrency (80)
- Scale up: Instant (new instance in < 1 second)
- Scale down: After 15 minutes of low traffic
- Min instances prevent cold starts for critical services

**Database Scaling**:
- Vertical: Change machine type (few minutes downtime)
- Horizontal: Add read replicas for read-heavy workloads
- Storage: Auto-resize when 90% full

**Redis Scaling**:
- Vertical: Increase memory size (zero downtime)
- Horizontal: Enable read replicas (Standard tier)

#### Cost Estimates

**Small Deployment (<10M req/mo)**:
```
Cloud Run:
  - CPU: 10M req × 200ms × 1 vCPU × $0.00002400/vCPU-sec = $48
  - Memory: 10M req × 200ms × 2GiB × $0.00000250/GiB-sec = $10
  - Requests: 10M × $0.40/1M = $4
  - Min instance (1 always-on): 1 × 1 vCPU × 730hr × 3600s × $0.00002400 = $63.07
  
Cloud SQL:
  - Instance: db-custom-2-8192 × $0.0965/hr × 730hr = $70.45
  - Storage: 100GB × $0.17/GB = $17
  - HA standby: $70.45
  
Memorystore Redis:
  - Standard 1GB: $0.049/GB/hr × 1GB × 730hr = $35.77
  
Load Balancer:
  - Forwarding rule: $0.025/hr × 730hr = $18.25
  - Data processed: 500GB × $0.008/GB = $4
  
Cloud Armor: $5/month base

Data egress: 500GB × $0.12/GB = $60

Total: ~$406/month
```

**Medium Deployment (50M req/mo)**:
```
Cloud Run:
  - CPU: 50M × 250ms × 2 vCPU × $0.00002400 = $600
  - Memory: 50M × 250ms × 4GiB × $0.00000250 = $125
  - Requests: 50M × $0.40/1M = $20
  - Min instances (3): 3 × 2 vCPU × 730hr × 3600s × $0.00002400 = $378.43
  
Cloud SQL:
  - Instance: db-custom-4-16384 × $0.193/hr × 730hr = $140.89
  - Storage: 500GB × $0.17/GB = $85
  - HA standby: $140.89
  
Memorystore Redis:
  - Standard 10GB: $0.049/GB/hr × 10GB × 730hr = $357.70
  
Load Balancer:
  - Forwarding rule: $18.25
  - Data: 2TB × $0.008/GB = $16.38
  
Data egress: 2TB × $0.12/GB = $245.76

Total: ~$2,128/month
```

**Large Deployment (500M req/mo)**:
```
Cloud Run:
  - CPU: 500M × 300ms × 4 vCPU × $0.00002400 = $14,400
  - Memory: 500M × 300ms × 8GiB × $0.00000250 = $3,000
  - Requests: 500M × $0.40/1M = $200
  - Min instances (10): 10 × 4 vCPU × 730hr × 3600s × $0.00002400 = $2,522.88
  
Cloud SQL:
  - Instance: db-custom-16-65536 × $0.772/hr × 730hr = $563.56
  - Storage: 2TB × $0.17/GB = $347.52
  - HA standby: $563.56
  
Memorystore Redis (with read replicas):
  - Standard 100GB primary: $0.049/GB/hr × 100GB × 730hr = $3,577
  - Read replica: $3,577
  
Load Balancer:
  - Forwarding rule: $18.25
  - Data: 20TB × $0.008/GB = $163.84
  
Data egress: 10TB × $0.12/GB = $1,228.80

Total: ~$30,162/month
```

#### Pros & Cons

**Pros**:
✅ Fully managed, zero infrastructure  
✅ Auto-scales to zero (cost-effective for variable load)  
✅ Fast cold starts (gen2: < 500ms)  
✅ No cluster management  
✅ Integrated with GCP ecosystem  
✅ Simple deployment (Docker image or source-based)  
✅ Built-in traffic splitting (canary, blue/green)  

**Cons**:
❌ GCP-specific (vendor lock-in)  
❌ Cold starts (mitigated with min instances)  
❌ Limited control vs GKE  
❌ Concurrency model requires tuning  
❌ VPC networking adds complexity  

#### Security Considerations

**IAM & Service Accounts**:
```bash
# Cloud Run service account with least privilege
gcloud iam service-accounts create aegis-run-sa \
  --display-name="AEGIS Cloud Run Service Account"

# Grant access to Secret Manager
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:aegis-run-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Grant access to Cloud SQL
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:aegis-run-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

**Cloud Armor (WAF)**:
```yaml
# Rate limiting rule
- priority: 1000
  description: "Rate limit per IP"
  match:
    expr: "true"
  rateLimitOptions:
    conformAction: "allow"
    exceedAction: "deny(429)"
    enforceOnKey: "IP"
    rateLimitThreshold:
      count: 2000
      intervalSec: 60

# Geo-blocking
- priority: 2000
  description: "Block high-risk countries"
  match:
    expr: "origin.region_code in ['CN', 'RU', 'KP']"
  action: "deny(403)"
```

**VPC Security**:
- Cloud Run in Serverless VPC Access (egress via connector)
- Cloud SQL: Private IP only (no public IP)
- Memorystore: VPC-native, no public access
- Cloud NAT for controlled egress

#### Monitoring & Observability

**Cloud Monitoring Dashboards**:
- Cloud Run: Request count, latency, error rate, instance count
- Cloud SQL: CPU, memory, connections, replication lag
- Memorystore: Hit rate, evictions, CPU

**SLO-Based Alerts**:
```yaml
# Example SLO: 99.9% availability
serviceLevelIndicator:
  requestBased:
    goodTotalRatio:
      goodServiceFilter: |
        resource.type="cloud_run_revision"
        metric.type="run.googleapis.com/request_count"
        metric.labels.response_code_class="2xx"
      totalServiceFilter: |
        resource.type="cloud_run_revision"
        metric.type="run.googleapis.com/request_count"
serviceLevelObjective:
  goal: 0.999
  rollingPeriod: 2592000s  # 30 days
```

**Cloud Trace**:
- Distributed tracing for request flow
- Latency breakdown by component

#### Disaster Recovery

**Backup**:
- Cloud SQL: Automated daily backups, point-in-time recovery (7 days)
- Memorystore: Manual snapshots to Cloud Storage
- Container images: Artifact Registry (versioned)

**Multi-Region Failover**:
- Deploy identical setup in `us-east1` (secondary)
- Cloud SQL cross-region replica
- Cloud DNS with health check failover
- RPO: ~1 minute (replication lag)
- RTO: ~5 minutes (DNS + Cloud SQL promotion)

---

### Option 2: GCP GKE (Google Kubernetes Engine)

**Best for**: Multi-cloud strategy, complex microservices, teams with K8s expertise.

*(Similar architecture to AWS EKS, adapted for GCP)*

#### Infrastructure Requirements

**GKE Cluster**:
- Type: Standard (Autopilot for less management)
- Version: 1.30+ (rapid channel for latest features)
- Node pools:
  - Small: 3-6 × e2-standard-2 (2 vCPU, 8GB)
  - Medium: 6-12 × e2-standard-4 (4 vCPU, 16GB)
  - Large: 12-30 × c2-standard-8 (8 vCPU, 32GB)
- Spot VMs: 50-70% for cost savings
- Network: VPC-native (alias IPs)

**Workload**:
```yaml
# Similar to EKS Deployment spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aegis-gateway
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: gateway
        image: gcr.io/PROJECT_ID/aegis-gateway:v1.2.3
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: aegis-db-creds
              key: host
```

#### Cost Estimates

**Small (<10M req/mo)**: ~$650/month  
**Medium (50M req/mo)**: ~$2,100/month  
**Large (500M req/mo)**: ~$8,900/month  

*(Similar breakdown to AWS EKS, adjusted for GCP pricing)*

#### Pros & Cons

**Pros**: Same as AWS EKS (portability, flexibility, rich ecosystem)  
**Cons**: Same as AWS EKS (complexity, operational overhead)  

*(Full details omitted for brevity; refer to AWS EKS section for methodology)*

---

## Azure Deployment Options

### Option 1: Azure Container Apps (Recommended)

**Best for**: Azure-first organizations, serverless containers, similar to AWS Fargate + Cloud Run.

#### Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                     Microsoft Azure                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Azure Front Door (CDN + WAF)                            │  │
│  │  - Custom domain + SSL                                   │  │
│  │  - DDoS protection                                       │  │
│  │  - Web Application Firewall                              │  │
│  └───────────────┬──────────────────────────────────────────┘  │
│                  │                                              │
│  ┌───────────────▼──────────────────────────────────────────┐  │
│  │  Container Apps Environment                              │  │
│  │  - Region: East US                                       │  │
│  │  - VNet integration: Enabled                             │  │
│  │  - Log Analytics workspace attached                      │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  Container App: aegis-gateway                      │  │  │
│  │  │  - Min replicas: 1-3                               │  │  │
│  │  │  - Max replicas: 30                                │  │  │
│  │  │  - CPU: 0.5-2.0 vCPU per replica                   │  │  │
│  │  │  - Memory: 1-4 Gi per replica                      │  │  │
│  │  │  - Ingress: HTTPS, external                        │  │  │
│  │  │  - Scale rules: HTTP (100 concurrent req/replica)  │  │  │
│  │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐              │  │  │
│  │  │  │ Replica │ │ Replica │ │ Replica │  ...         │  │  │
│  │  │  └─────────┘ └─────────┘ └─────────┘              │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼───────────────────────────────────┐  │
│  │  Azure Cache for Redis (Enterprise tier)                 │  │
│  │  - SKU: E10 (12 GB) or E20 (25 GB)                       │  │
│  │  - Clustering: Enabled                                   │  │
│  │  - Geo-replication: Optional (Enterprise tier)           │  │
│  │  - Persistence: RDB + AOF                                │  │
│  │  - Private endpoint: VNet integrated                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼───────────────────────────────────┐  │
│  │  Azure Database for PostgreSQL (Flexible Server)         │  │
│  │  - SKU: General Purpose D2s_v3 (2 vCPU, 8 GiB) - small   │  │
│  │         General Purpose D4s_v3 (4 vCPU, 16 GiB) - medium │  │
│  │         Memory Optimized E8s_v3 (8 vCPU, 64 GiB) - large │  │
│  │  - Storage: 128-2048 GB (auto-grow enabled)              │  │
│  │  - HA: Zone-redundant (primary + standby)                │  │
│  │  - Backup: Automated, 7-day retention                    │  │
│  │  - Private endpoint: VNet integrated                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Azure Key Vault                                          │  │
│  │  - Secrets: DB password, OpenAI key, Anthropic key       │  │
│  │  - Access policy: Container App managed identity         │  │
│  │  - Soft delete: Enabled (90 days)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Azure Monitor + Application Insights                     │  │
│  │  - Metrics: Request count, latency, availability         │  │
│  │  - Logs: Application logs, platform logs                 │  │
│  │  - Alerts: SLA-based (99.9% availability)                │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### Infrastructure Requirements

**Container App**:
- CPU: 0.5-2.0 vCPU per replica
- Memory: 1-4 Gi per replica
- Min replicas: 1-3 (always-on)
- Max replicas: 30-100
- Concurrency: 100 requests per replica
- Timeout: 240 seconds

**PostgreSQL Flexible Server**:
- Small: General Purpose D2s_v3 (2 vCPU, 8 GiB), 128 GB
- Medium: General Purpose D4s_v3 (4 vCPU, 16 GiB), 512 GB
- Large: Memory Optimized E8s_v3 (8 vCPU, 64 GiB), 2 TB
- HA: Zone-redundant (automatic failover)

**Azure Cache for Redis**:
- Small: Standard C1 (1 GB) or Premium P1 (6 GB)
- Medium: Premium P2 (13 GB) or Enterprise E10 (12 GB)
- Large: Enterprise E20 (25 GB) with clustering

#### Cost Estimates

**Small (<10M req/mo)**: ~$520/month  
**Medium (50M req/mo)**: ~$2,450/month  
**Large (500M req/mo)**: ~$18,500/month  

*(Detailed breakdown omitted for brevity; similar methodology to AWS/GCP)*

#### Pros & Cons

**Pros**:
✅ Azure-native integration (Entra ID, Key Vault, Monitor)  
✅ Serverless scaling  
✅ Dapr integration (distributed app runtime)  
✅ Simple deployment  

**Cons**:
❌ Azure-specific (vendor lock-in)  
❌ Less mature than AWS/GCP serverless offerings  
❌ Limited regions  

---

## On-Premise Deployment

### Option 1: Virtual Machines (VMware, Proxmox, KVM)

**Best for**: Data sovereignty, regulatory compliance (GDPR, HIPAA), air-gapped environments.

#### Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    On-Premise Data Center                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Hardware Load Balancer (F5, HAProxy, NGINX)            │  │
│  │  - VIP: 10.0.1.10 (internal), 203.0.113.10 (external)   │  │
│  │  - SSL termination                                       │  │
│  │  - Health checks: GET /aegis/v1/health every 5s         │  │
│  │  - Algorithms: Least connections                        │  │
│  └───────────────┬──────────────────────────────────────────┘  │
│                  │                                              │
│  ┌───────────────▼──────────────────────────────────────────┐  │
│  │  Gateway VM Cluster (3-10 VMs)                          │  │
│  │  ┌─────────────────┐  ┌─────────────────┐              │  │
│  │  │  VM1            │  │  VM2            │  ...          │  │
│  │  │  10.0.1.11      │  │  10.0.1.12      │              │  │
│  │  │  Ubuntu 24.04   │  │  Ubuntu 24.04   │              │  │
│  │  │  4 vCPU, 8GB    │  │  4 vCPU, 8GB    │              │  │
│  │  │  systemd service│  │  systemd service│              │  │
│  │  └─────────────────┘  └─────────────────┘              │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼───────────────────────────────────┐  │
│  │  Redis Cluster (3 masters + 3 replicas)                  │  │
│  │  - VM3-VM8: 10.0.1.13-10.0.1.18                           │  │
│  │  - 2 vCPU, 4GB per VM                                     │  │
│  │  - Redis 7.0, cluster mode enabled                       │  │
│  │  - Persistence: AOF + RDB                                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                         │                                       │
│  ┌──────────────────────▼───────────────────────────────────┐  │
│  │  PostgreSQL Cluster (Patroni + etcd)                     │  │
│  │  - VM9-VM11: 10.0.1.19-10.0.1.21                          │  │
│  │  - Primary: VM9, Replicas: VM10-VM11                     │  │
│  │  - 8 vCPU, 32GB per VM                                   │  │
│  │  - Storage: 500GB SSD RAID 10                            │  │
│  │  - Auto-failover via Patroni + etcd consensus            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Secrets Management (HashiCorp Vault)                     │  │
│  │  - VM12-VM14: 10.0.1.22-10.0.1.24 (HA cluster)            │  │
│  │  - 2 vCPU, 4GB per VM                                     │  │
│  │  - Storage backend: Raft (integrated)                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Monitoring Stack                                          │  │
│  │  - Prometheus (VM15): Metrics collection                  │  │
│  │  - Grafana (VM16): Dashboards                             │  │
│  │  - Loki (VM17): Log aggregation                           │  │
│  │  - Alertmanager (VM18): Alert routing (PagerDuty, Slack)  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Backup Storage (NAS/SAN)                                  │  │
│  │  - Daily PostgreSQL dumps (pgBackRest)                    │  │
│  │  - Redis RDB snapshots                                    │  │
│  │  - Retention: 30 days on-site, 90 days off-site          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### Infrastructure Requirements

**Gateway VMs**:
- Count: 3-10 (depends on load)
- Specs: 4 vCPU, 8GB RAM, 50GB SSD per VM
- OS: Ubuntu 24.04 LTS or RHEL 9
- Deployment: systemd service with auto-restart
- Config management: Ansible/Chef/Puppet

**PostgreSQL Cluster**:
- Primary + 2 replicas (Patroni for auto-failover)
- Specs: 8 vCPU, 32GB RAM, 500GB-2TB SSD RAID 10
- OS: Ubuntu 24.04 LTS
- Version: PostgreSQL 16

**Redis Cluster**:
- 3 masters + 3 replicas (cluster mode)
- Specs: 2 vCPU, 4GB RAM, 50GB SSD per node
- Persistence: AOF + RDB snapshots

**Load Balancer**:
- Hardware: F5 BIG-IP, Cisco ACE
- Software: HAProxy, NGINX Plus, Traefik
- HA: Active-passive pair (VRRP/keepalived)

#### Scaling Strategy

**Horizontal Scaling**:
- Add more gateway VMs (no state, easy to scale)
- Update load balancer backend pool
- Provision via automation (Ansible playbooks)

**Vertical Scaling**:
- Increase VM resources (vCPU, RAM)
- Requires VM restart (brief downtime)

**Database Scaling**:
- Vertical: Increase primary VM resources
- Horizontal: Add read replicas (Patroni manages replication)

#### Cost Estimates (Amortized Hardware)

**Small (<10M req/mo)**:
```
VMs (3 gateway + 3 Redis + 3 Postgres + 4 monitoring):
  - Hardware: 13 VMs × $100/mo (amortized) = $1,300
  - But shared across workloads, allocate 30% to AEGIS = $390
  
Power & Cooling: $50/month

Network bandwidth: $20/month (internal only)

Operations (1 engineer, 10% time): $1,200/month

Total: ~$1,660/month (or $300/mo excluding ops labor)
```

**Medium (50M req/mo)**:
```
VMs (8 gateway + 6 Redis + 3 Postgres + 4 monitoring):
  - Allocation: $650/month
  
Power: $80/month

Bandwidth: $50/month

Operations (1 engineer, 20% time): $2,400/month

Total: ~$3,180/month (or $780/mo excluding ops)
```

**Large (500M req/mo)**:
```
VMs (30 gateway + 6 Redis + 5 Postgres + 6 monitoring):
  - Allocation: $2,100/month
  
Power: $200/month

Bandwidth: $150/month

Operations (2 engineers, 30% time): $7,200/month

Total: ~$9,650/month (or $2,450/mo excluding ops)
```

**Note**: Costs exclude initial hardware CapEx (~$50k-200k for servers, storage, networking). Amortized over 5 years.

#### Pros & Cons

**Pros**:
✅ Full control over infrastructure  
✅ Data sovereignty (no data leaves premises)  
✅ Compliance-friendly (GDPR, HIPAA, FedRAMP)  
✅ No cloud egress fees  
✅ Predictable costs (after CapEx amortization)  
✅ Customizable hardware (GPUs, FPGAs, etc.)  

**Cons**:
❌ High upfront CapEx  
❌ Operational burden (patching, monitoring, on-call)  
❌ Scaling takes time (procurement, provisioning)  
❌ Disaster recovery complex (physical backups)  
❌ Requires in-house expertise  
❌ Limited redundancy (single data center risk)  

#### Security Hardening

**Network Segmentation**:
- VLANs: Management, application, database, monitoring
- Firewall rules: Least-privilege (only required ports)
- DMZ for public-facing load balancer

**OS Hardening**:
- CIS Benchmarks for Ubuntu/RHEL
- Disable unnecessary services
- Automatic security updates (unattended-upgrades)
- SELinux/AppArmor enabled

**Secrets Management**:
- HashiCorp Vault for centralized secrets
- Auto-unseal with KMS or cloud provider
- Audit logging enabled

**Monitoring**:
- Prometheus + Grafana for metrics
- Loki for logs
- Alertmanager for critical alerts (PagerDuty, email)

---

## Hybrid Cloud Scenarios

### Scenario 1: Multi-Cloud for Redundancy (AWS Primary + GCP Secondary)

**Use Case**: Maximum uptime (99.99%), avoid single-cloud vendor lock-in.

#### Architecture

- **Primary**: AWS (ECS Fargate in us-east-1)
- **Secondary**: GCP (Cloud Run in us-central1)
- **Database**: Aurora Global Database (AWS primary, GCP cross-region read replica)
- **Redis**: ElastiCache (AWS) + Memorystore (GCP), replicated via custom sync
- **Traffic Management**: Route 53 with health check failover

#### Cost

~1.8× single-cloud cost (redundant infrastructure + replication overhead)

#### Pros

✅ Near-zero downtime (cloud provider outages)  
✅ Vendor negotiation leverage  
✅ Flexibility to migrate workloads  

#### Cons

❌ Complexity (manage two platforms)  
❌ Data egress costs for replication  
❌ Harder to debug cross-cloud issues  

---

### Scenario 2: On-Prem + Cloud Bursting (AWS)

**Use Case**: Keep most data on-prem (compliance), burst to cloud during peak load.

#### Architecture

- **On-Premise**: Primary gateway + PostgreSQL + Redis (handles 80% of baseline traffic)
- **AWS (Burst)**: ECS Fargate + RDS Read Replica + ElastiCache (handles spikes)
- **Connection**: Site-to-Site VPN or Direct Connect
- **Load Balancer**: On-prem HAProxy routes overflow to AWS

#### Cost

Base: On-prem costs (~$1,500/mo) + AWS minimal standby (~$300/mo)  
Peak: On-prem + AWS full burst (~$3,000 total during spike months)

#### Pros

✅ Keep sensitive data on-prem  
✅ Cost-effective for variable workloads  
✅ Gradual cloud migration path  

#### Cons

❌ Network latency (VPN/Direct Connect)  
❌ Complex failover logic  
❌ Data sync challenges  

---

## Reference Architectures

### Reference Architecture 1: Small Deployment (Startup, <10M req/mo)

**Profile**:
- Early-stage startup, 1-2 engineers
- Budget: $500-1,000/month
- Traffic: 300k requests/day, mostly business hours
- Compliance: Standard (no special requirements)

**Recommended Stack**: **GCP Cloud Run** or **AWS ECS Fargate**

**Architecture**:
```
- Cloud Run: 1-3 instances (auto-scale)
- Cloud SQL: db-custom-2-8192 (2 vCPU, 8GB), HA enabled
- Memorystore: 1GB Standard tier
- Load Balancer: Global HTTPS LB
- Monitoring: Cloud Monitoring (free tier)
```

**Deployment Steps**:
1. Create GCP project, enable APIs
2. Build Docker image: `docker build -t gcr.io/PROJECT/aegis-gateway .`
3. Push to Artifact Registry: `docker push gcr.io/PROJECT/aegis-gateway`
4. Deploy Cloud Run: `gcloud run deploy aegis-gateway --image=... --min-instances=1`
5. Provision Cloud SQL: `gcloud sql instances create aegis-db --tier=db-custom-2-8192`
6. Run migrations: `./bin/migrate -database=postgres://... up`
7. Generate API keys: `./bin/keygen`
8. Configure load balancer + custom domain

**Cost**: ~$500/month  
**Time to Deploy**: 2-4 hours (first time), 30 minutes (subsequent)

---

### Reference Architecture 2: Medium Deployment (Growth Company, 10M-100M req/mo)

**Profile**:
- Growth-stage company, 5-10 engineers
- Budget: $2,000-10,000/month
- Traffic: 1-3M requests/day, global users
- Compliance: SOC2, GDPR

**Recommended Stack**: **AWS ECS Fargate** (multi-region)

**Architecture**:
```
Primary Region (us-east-1):
  - ECS Fargate: 5-15 tasks (auto-scale)
  - RDS PostgreSQL: db.r6g.large Multi-AZ
  - ElastiCache: cache.r7g.large (primary + 2 replicas)
  - ALB: HTTPS, WAF enabled

Secondary Region (us-west-2):
  - ECS Fargate: 3-10 tasks
  - RDS Read Replica (cross-region)
  - ElastiCache: cache.r7g.large

Traffic Management:
  - Route 53: Latency-based routing
  - Health checks: Failover to us-west-2 if us-east-1 down

Monitoring:
  - CloudWatch: Metrics, logs, alarms
  - Datadog or New Relic: APM, distributed tracing
  - PagerDuty: On-call rotation

CI/CD:
  - GitHub Actions: Build, test, push to ECR
  - Blue/green deployments via ECS
  - Automated rollback on high error rate
```

**Cost**: ~$4,000-7,000/month (depending on traffic)  
**Time to Deploy**: 1-2 days (infrastructure + automation)

**Compliance**:
- SOC2: Enable CloudTrail, Config, GuardDuty
- GDPR: Encrypt at rest (KMS), in transit (TLS), data retention policies
- Audit logs: All requests logged to S3 (immutable, 7-year retention)

---

### Reference Architecture 3: Large Deployment (Enterprise, 100M-1B+ req/mo)

**Profile**:
- Enterprise company, 20+ engineers, dedicated platform team
- Budget: $15,000-100,000/month
- Traffic: 10M-30M requests/day, global, mission-critical
- Compliance: SOC2, GDPR, HIPAA, FedRAMP

**Recommended Stack**: **AWS EKS** (multi-region) or **Multi-Cloud (AWS + GCP)**

**Architecture**:
```
Primary Region (AWS us-east-1):
  - EKS Cluster: 15-40 nodes (c6i.2xlarge), 50% spot
  - Pods: 30-100 replicas (HPA)
  - RDS: db.r6g.4xlarge Multi-AZ, 10,000 IOPS
  - ElastiCache: Cluster mode, 4 shards, 2 replicas each
  - ALB: HTTPS, WAF with custom rules, DDoS protection

Secondary Region (AWS us-west-2):
  - Identical setup, handles 30% of traffic (active-active)

Tertiary Region (GCP us-central1):
  - GKE cluster (standby for disaster recovery)
  - Cloud SQL cross-cloud replica (experimental)

Database:
  - Aurora Global Database (primary in us-east-1, replicas in us-west-2)
  - Read replicas in each region for analytics

Traffic Management:
  - Global Accelerator (AWS): Anycast IPs, automatic failover
  - Route 53: Geo-routing, health checks
  - CloudFlare (optional): Global CDN, additional WAF layer

Monitoring & Observability:
  - Prometheus Operator: Metrics collection
  - Grafana: Real-time dashboards
  - ELK Stack: Centralized logging (Elasticsearch, Logstash, Kibana)
  - Jaeger: Distributed tracing
  - Datadog: Unified observability (APM, infrastructure, logs)

Security:
  - Secrets: HashiCorp Vault (multi-region)
  - Network: VPC peering, Transit Gateway
  - Compliance: AWS Audit Manager, automated evidence collection

CI/CD:
  - GitOps: ArgoCD for K8s deployments
  - GitHub Actions: Build, test, security scans
  - Canary deployments: 5% → 25% → 50% → 100% over 2 hours
  - Automated rollback: Error rate > 1%, latency p99 > 2s
```

**Cost**: ~$30,000-80,000/month  
**Time to Deploy**: 1-2 weeks (full setup + testing)

**Disaster Recovery**:
- RPO: < 1 minute (continuous replication)
- RTO: < 5 minutes (automated failover)
- Runbooks: Tested quarterly
- Chaos engineering: Quarterly game days (simulate region failures)

---

## High Availability Architecture

### Multi-Region Active-Active

**Goal**: 99.99% uptime, global low latency, survive full region outage.

#### Components

**Traffic Management**:
```
CloudFlare or CloudFront (CDN + DDoS)
       ↓
Route 53 Geo-Routing
       ↓
┌──────────────┬──────────────┐
│ us-east-1    │ eu-west-1    │
│ 50% traffic  │ 50% traffic  │
└──────────────┴──────────────┘
```

**Database**:
- **Primary**: Aurora Global Database
  - Writer in us-east-1
  - Read replicas in eu-west-1, ap-southeast-1
  - Failover time: < 1 minute
- **Eventual consistency**: Gateway tolerates 1-2 second replication lag

**Redis**:
- Separate Redis clusters per region (no cross-region replication)
- TTL-based cache (invalidation not critical)
- On failover: Cache miss rate increases temporarily (acceptable)

**Compute**:
- ECS/EKS in each region
- Auto-scaling independent per region
- Health checks every 10 seconds

#### Failover Procedures

**Automatic Failover** (Region Failure):
1. Route 53 health check fails (3 consecutive failures, 30s interval)
2. Route 53 updates DNS to healthy region (TTL: 60s)
3. Clients gradually failover (within 2-5 minutes)
4. Aurora promotes replica in healthy region to writer (if primary region writer failed)

**Manual Failover** (Planned Maintenance):
1. Scale up secondary region (2× capacity)
2. Update Route 53 weights (gradual shift: 50/50 → 30/70 → 0/100)
3. Monitor error rates, latency
4. Perform maintenance in primary region
5. Reverse weight shift

---

## CI/CD Pipeline

### GitHub Actions Workflow

**Stages**: Build → Test → Security Scan → Push → Deploy → Verify → Notify

#### Workflow File

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, staging, production]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.25'
      
      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
      
      - name: Run tests
        run: |
          go test -v -race -coverprofile=coverage.txt -covermode=atomic ./...
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.txt

  build-docker:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/production'
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/aegis-gateway:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/aegis-gateway:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  security-scan:
    needs: build-docker
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/aegis-gateway:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  deploy-staging:
    needs: security-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/GitHubActions-DeployRole
          aws-region: us-east-1
      
      - name: Update ECS service (blue/green)
        run: |
          # Create new task definition with updated image
          NEW_TASK_DEF=$(aws ecs describe-task-definition \
            --task-definition aegis-gateway-staging \
            --query 'taskDefinition' | \
            jq --arg IMAGE "${{ steps.login-ecr.outputs.registry }}/aegis-gateway:${{ github.sha }}" \
            '.containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)')
          
          aws ecs register-task-definition --cli-input-json "$NEW_TASK_DEF"
          
          # Update service
          aws ecs update-service \
            --cluster aegis-staging \
            --service aegis-gateway \
            --task-definition aegis-gateway-staging \
            --force-new-deployment
      
      - name: Wait for deployment to stabilize
        run: |
          aws ecs wait services-stable \
            --cluster aegis-staging \
            --services aegis-gateway
      
      - name: Run smoke tests
        run: |
          ENDPOINT="https://staging.aegis.example.com"
          
          # Health check
          curl -f "$ENDPOINT/aegis/v1/health" || exit 1
          
          # Test request
          curl -f -X POST "$ENDPOINT/v1/chat/completions" \
            -H "Authorization: Bearer ${{ secrets.STAGING_API_KEY }}" \
            -H "Content-Type: application/json" \
            -d '{"model":"aegis-fast","messages":[{"role":"user","content":"test"}]}' \
            || exit 1

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/production'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy with canary strategy
        run: |
          # Deploy to 5% of traffic (canary)
          # Wait 10 minutes, monitor metrics
          # If error rate < 0.5%, promote to 25%
          # Wait 10 minutes
          # If still healthy, promote to 100%
          # Else, rollback
          
          # (Detailed script omitted for brevity)

      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ AEGIS Gateway deployed to production",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Successful*\nCommit: ${{ github.sha }}\nAuthor: ${{ github.actor }}"
                  }
                }
              ]
            }
```

### Deployment Strategies

**1. Rolling Update** (Zero Downtime):
- Deploy new version gradually (1 instance at a time)
- Health check passes before proceeding
- Total time: 5-15 minutes (depending on instance count)
- Risk: Low (gradual rollout)

**2. Blue/Green**:
- Deploy full new environment (green) alongside old (blue)
- Test green thoroughly
- Switch load balancer from blue to green (instant cutover)
- Keep blue running for 1 hour (rollback window)
- Risk: Medium (full cutover, but easy rollback)

**3. Canary** (Recommended for Production):
- Deploy to small % of traffic (5%)
- Monitor metrics: error rate, latency, cost per request
- Gradually increase: 5% → 25% → 50% → 100% (over 1-2 hours)
- Automated rollback if metrics degrade
- Risk: Very Low (early detection of issues)

### Rollback Procedures

**Automated Rollback Triggers**:
- Error rate > 1% (5xx responses)
- Latency p99 > 2 seconds (sustained for 5 minutes)
- Custom metrics: Cost per request > 2× baseline, API key validation failures > 5%

**Manual Rollback** (Emergency):
```bash
# AWS ECS
aws ecs update-service \
  --cluster aegis-production \
  --service aegis-gateway \
  --task-definition aegis-gateway:123 \
  --force-new-deployment

# Kubernetes
kubectl rollout undo deployment/aegis-gateway -n aegis

# Verify
kubectl rollout status deployment/aegis-gateway -n aegis
```

---

## Security Hardening

### Network Isolation

**VPC Architecture** (AWS Example):
```
VPC: 10.0.0.0/16

Public Subnets (DMZ):
  - 10.0.1.0/24 (us-east-1a) - ALB only
  - 10.0.2.0/24 (us-east-1b) - ALB only

Private Subnets (Application):
  - 10.0.10.0/24 (us-east-1a) - ECS tasks, Lambda
  - 10.0.11.0/24 (us-east-1b) - ECS tasks, Lambda

Data Subnets (Database):
  - 10.0.20.0/24 (us-east-1a) - RDS, Redis
  - 10.0.21.0/24 (us-east-1b) - RDS, Redis

Security Groups:
  - ALB-SG: Inbound 443 from 0.0.0.0/0, outbound to ECS-SG
  - ECS-SG: Inbound 8080 from ALB-SG, outbound 443 to 0.0.0.0/0 (APIs), 5432/6379 to Data-SG
  - Data-SG: Inbound 5432/6379 from ECS-SG only

Network ACLs:
  - Public: Allow 443 inbound, deny known bad IPs (blocklist)
  - Private: Allow all from VPC, deny external inbound
  - Data: Allow only from private subnets
```

### Secrets Management

**AWS Secrets Manager**:
```bash
# Store API keys
aws secretsmanager create-secret \
  --name aegis/openai-api-key \
  --secret-string "sk-proj-..." \
  --kms-key-id alias/aegis-secrets

# Enable auto-rotation (Lambda function)
aws secretsmanager rotate-secret \
  --secret-id aegis/openai-api-key \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:ACCOUNT:function:aegis-secret-rotator \
  --rotation-rules AutomaticallyAfterDays=30

# Application retrieval (IAM role attached)
aws secretsmanager get-secret-value --secret-id aegis/openai-api-key
```

**HashiCorp Vault** (On-Premise):
```bash
# Initialize Vault
vault operator init

# Enable KV secrets engine
vault secrets enable -path=aegis kv-v2

# Store secret
vault kv put aegis/openai-api-key value="sk-proj-..."

# Application retrieval (AppRole auth)
vault login -method=approle role_id=... secret_id=...
vault kv get -field=value aegis/openai-api-key
```

### TLS/SSL Configuration

**Certificate Management**:
- **Cloud**: AWS ACM, GCP Certificate Manager (free, auto-renewal)
- **On-Premise**: Let's Encrypt (free, auto-renewal via Certbot)

**TLS Configuration** (NGINX Example):
```nginx
server {
    listen 443 ssl http2;
    server_name aegis.example.com;

    # Modern configuration (TLS 1.2+)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;

    # HSTS (force HTTPS for 1 year)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Certificate
    ssl_certificate /etc/letsencrypt/live/aegis.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aegis.example.com/privkey.pem;

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/aegis.example.com/chain.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### WAF Rules

**AWS WAF**:
```json
{
  "Name": "AegisWAF",
  "Rules": [
    {
      "Name": "RateLimitPerIP",
      "Priority": 1,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": { "Block": {} }
    },
    {
      "Name": "GeoBlocking",
      "Priority": 2,
      "Statement": {
        "GeoMatchStatement": {
          "CountryCodes": ["CN", "RU", "KP", "IR"]
        }
      },
      "Action": { "Block": {} }
    },
    {
      "Name": "SQLInjection",
      "Priority": 3,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesSQLiRuleSet"
        }
      },
      "OverrideAction": { "None": {} }
    }
  ]
}
```

### Compliance Considerations

**SOC2 Requirements**:
- ✅ Audit logging: All requests logged with user context, timestamps
- ✅ Encryption: At rest (KMS/CMEK), in transit (TLS 1.2+)
- ✅ Access control: RBAC, MFA for admin access
- ✅ Change management: All infrastructure changes via CI/CD, reviewed
- ✅ Monitoring: 24/7 monitoring, incident response procedures

**GDPR**:
- ✅ Data minimization: Only collect necessary data (API keys, usage metrics)
- ✅ Right to erasure: API to delete user data (`DELETE /users/{id}`)
- ✅ Data portability: Export user data in JSON format
- ✅ Consent: API key creation implies consent (terms of service)
- ✅ Data residency: Deploy in EU region (eu-west-1, europe-west1) for EU users

**HIPAA** (Healthcare):
- ✅ PHI protection: Content filtering blocks PII/PHI in prompts
- ✅ Audit logs: Tamper-proof logs (S3 Object Lock, CloudTrail)
- ✅ Encryption: FIPS 140-2 compliant encryption modules
- ✅ Access control: Role-based access, MFA
- ✅ Business Associate Agreement (BAA): Sign BAA with cloud provider (AWS, GCP, Azure support HIPAA workloads)

---

## Cost Optimization

### Reserved Instances vs. Spot

**Reserved Instances** (1-year or 3-year commitment):
- Discount: 30-40% (1-year), 50-60% (3-year)
- Use for: Baseline load (always-on instances)
- Example: RDS (always running), ElastiCache, EKS control plane

**Spot Instances** (Variable pricing, can be interrupted):
- Discount: 50-90% vs. on-demand
- Use for: EKS worker nodes (with graceful shutdown), batch jobs
- Not for: Database, Redis, critical services

**Savings Plans** (AWS):
- Compute Savings Plan: 66% discount, applies to Lambda, Fargate, EC2
- Flexible: Change instance types, regions

**Strategy**:
- Small deployment: 100% on-demand (flexibility)
- Medium deployment: 50% reserved (baseline), 50% on-demand (peaks)
- Large deployment: 60% reserved, 30% spot, 10% on-demand

### Auto-Scaling Policies

**Aggressive Scale-Up** (handle traffic spikes):
```yaml
# Scale up when CPU > 70% for 1 minute
ScaleUpPolicy:
  metric: CPUUtilization
  threshold: 70
  duration: 60s
  adjustment: +50% (add half the current instances)
  cooldown: 60s
```

**Conservative Scale-Down** (avoid thrashing):
```yaml
# Scale down when CPU < 30% for 10 minutes
ScaleDownPolicy:
  metric: CPUUtilization
  threshold: 30
  duration: 600s
  adjustment: -1 instance at a time
  cooldown: 300s
```

### Cache Warming Strategies

**Problem**: Cold cache = high upstream API costs  
**Solution**: Pre-warm frequently accessed data

```python
# Cache warming script (run after deployment)
import redis
import requests

r = redis.Redis(host='redis.example.com', port=6379)

# Pre-populate common model metadata
models = ['gpt-4o', 'gpt-4o-mini', 'claude-3.5-sonnet']
for model in models:
    metadata = fetch_model_metadata(model)
    r.setex(f"model:{model}", 3600, json.dumps(metadata))

# Pre-populate API key lookups (from database)
api_keys = db.query("SELECT api_key_hash, organization FROM api_keys WHERE active=true LIMIT 1000")
for key in api_keys:
    r.setex(f"auth:{key.api_key_hash}", 3600, key.organization)
```

### Database Connection Pooling

**Problem**: Lambda/Fargate create many connections, exhaust database  
**Solution**: Connection pooling + RDS Proxy

**Go Application** (built-in pooling):
```go
db, err := sql.Open("postgres", connString)
db.SetMaxOpenConns(25)         // Limit total connections
db.SetMaxIdleConns(10)         // Keep 10 idle connections
db.SetConnMaxLifetime(5 * time.Minute) // Recycle connections
```

**RDS Proxy** (AWS):
- Pools connections from thousands of Lambda invocations → 100 database connections
- IAM authentication (no password management)
- Failover in < 1 second (vs. 60s for RDS Multi-AZ)

### CDN for Static Assets

**Use Case**: Serve model metadata, documentation, OpenAPI specs  
**Solution**: CloudFront (AWS), Cloud CDN (GCP), Azure CDN

**Cost Savings**:
- Without CDN: 10M requests × 5KB = 50GB data transfer × $0.09/GB = $4.50
- With CDN: 90% cache hit rate, only 10% origin requests = $0.45 (10× savings)

**Configuration**:
```yaml
# CloudFront distribution
Origins:
  - DomainName: aegis-api.example.com
    CustomHeaders:
      - HeaderName: X-Origin-Verify
        HeaderValue: secret-token

Behaviors:
  - PathPattern: /v1/models
    CacheTTL: 3600s
    Compress: true
  - PathPattern: /docs/*
    CacheTTL: 86400s
    Compress: true
```

---

## Migration Paths

### Migration 1: Local Dev → AWS Cloud

**Scenario**: Moving from `mise run dev` (local Docker Compose) to AWS ECS Fargate.

#### Pre-Migration Checklist

- [ ] Export database schema: `pg_dump --schema-only aegis > schema.sql`
- [ ] Backup data: `pg_dump aegis > backup.sql`
- [ ] Document environment variables (`.env` file)
- [ ] Test Docker image builds: `docker build -t aegis-gateway .`
- [ ] Estimate costs (use AWS Pricing Calculator)

#### Migration Steps

**1. Provision Infrastructure** (Terraform):
```hcl
# main.tf
resource "aws_vpc" "aegis" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_db_instance" "aegis" {
  identifier           = "aegis-db"
  engine               = "postgres"
  engine_version       = "16"
  instance_class       = "db.t4g.medium"
  allocated_storage    = 100
  storage_type         = "gp3"
  multi_az             = true
  # ... (full config omitted)
}

resource "aws_elasticache_replication_group" "aegis" {
  replication_group_id = "aegis-redis"
  engine               = "redis"
  node_type            = "cache.t4g.medium"
  num_cache_clusters   = 2
  # ... (full config omitted)
}

resource "aws_ecs_cluster" "aegis" {
  name = "aegis-cluster"
}

resource "aws_ecs_task_definition" "aegis" {
  family                   = "aegis-gateway"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "512"
  memory                   = "1024"
  container_definitions    = jsonencode([{
    name  = "gateway"
    image = "${aws_ecr_repository.aegis.repository_url}:latest"
    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]
    environment = [
      { name = "DB_HOST", value = aws_db_instance.aegis.endpoint },
      { name = "REDIS_HOST", value = aws_elasticache_replication_group.aegis.primary_endpoint_address }
    ]
    secrets = [
      { name = "OPENAI_API_KEY", valueFrom = aws_secretsmanager_secret.openai.arn }
    ]
  }])
}
```

**2. Migrate Database**:
```bash
# Restore schema
psql -h aegis-db.abc123.us-east-1.rds.amazonaws.com -U aegis -d aegis < schema.sql

# Restore data (small dataset)
psql -h aegis-db.abc123.us-east-1.rds.amazonaws.com -U aegis -d aegis < backup.sql

# For large datasets, use AWS DMS (Database Migration Service)
```

**3. Deploy Application**:
```bash
# Build and push Docker image
docker build -t 123456789.dkr.ecr.us-east-1.amazonaws.com/aegis-gateway:v1.0.0 .
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/aegis-gateway:v1.0.0

# Create ECS service
aws ecs create-service \
  --cluster aegis-cluster \
  --service-name aegis-gateway \
  --task-definition aegis-gateway \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-123],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=gateway,containerPort=8080"
```

**4. Cutover**:
```bash
# Update DNS (Route 53)
# Point aegis.example.com → ALB DNS name

# Monitor metrics
# Watch CloudWatch for errors, latency

# Decommission local setup (after 7 days of stable cloud operation)
```

**Rollback Plan**:
- Keep local Docker Compose running for 7 days
- If cloud issues, revert DNS to local IP
- Export cloud database changes, re-import locally

---

### Migration 2: Single Cloud (AWS) → Multi-Cloud (AWS + GCP)

**Scenario**: Adding GCP as secondary region for redundancy.

#### Pre-Migration

- [ ] Assess data transfer costs (AWS → GCP egress)
- [ ] Plan database replication (Aurora Global Database or custom)
- [ ] Decide on traffic split (90/10, 50/50, active-passive)

#### Migration Steps

**1. Provision GCP Infrastructure**:
```bash
# GCP Cloud Run deployment
gcloud run deploy aegis-gateway \
  --image gcr.io/PROJECT/aegis-gateway:v1.0.0 \
  --region us-central1 \
  --min-instances 1 \
  --max-instances 10 \
  --set-env-vars DB_HOST=CLOUD_SQL_IP,REDIS_HOST=MEMORYSTORE_IP
```

**2. Replicate Database**:
- **Option A**: Logical replication (PostgreSQL native)
  ```sql
  -- On AWS (publisher)
  CREATE PUBLICATION aegis_pub FOR ALL TABLES;
  
  -- On GCP (subscriber)
  CREATE SUBSCRIPTION aegis_sub CONNECTION 'host=aws-db.com...' PUBLICATION aegis_pub;
  ```
- **Option B**: Third-party tool (Debezium, AWS DMS → GCS → Cloud SQL)

**3. Sync Redis** (Custom Script):
```python
# One-way sync AWS Redis → GCP Redis
import redis

aws_redis = redis.Redis(host='aws-redis.com', port=6379)
gcp_redis = redis.Redis(host='gcp-redis.com', port=6379)

# Initial full sync
for key in aws_redis.scan_iter():
    value = aws_redis.get(key)
    ttl = aws_redis.ttl(key)
    if ttl > 0:
        gcp_redis.setex(key, ttl, value)
    else:
        gcp_redis.set(key, value)

# Continuous sync (use Redis Streams or custom pub/sub)
```

**4. Traffic Management**:
```yaml
# Route 53 Weighted Routing
- AWS ALB: Weight 90
- GCP Load Balancer: Weight 10

# Gradually shift to 50/50 over 7 days
# Monitor metrics on both platforms
```

**Challenges**:
- Database replication lag (1-5 seconds)
- Redis cache inconsistency (accept eventually consistent cache)
- Increased operational complexity (two platforms to monitor)

---

### Migration 3: Zero-Downtime Migration (Blue/Green)

**Scenario**: Upgrade from old infrastructure to new (e.g., EC2 → ECS, or major version upgrade).

#### Steps

**1. Deploy New Environment** (Green):
```bash
# Green environment: Identical to production (Blue) but updated
terraform apply -var-file=green.tfvars
```

**2. Database Migration**:
```bash
# Option A: Shared database (Blue and Green use same RDS)
# - Run migrations before cutover
# - Ensure backward compatibility

# Option B: Separate databases
# - Replicate Blue DB → Green DB
# - Keep in sync with logical replication
```

**3. Test Green Environment**:
```bash
# Internal testing (before public traffic)
curl -H "Host: aegis.example.com" http://green-alb.amazonaws.com/aegis/v1/health

# Load testing
hey -n 10000 -c 100 -H "Authorization: Bearer test-key" http://green-alb.amazonaws.com/v1/chat/completions
```

**4. Cutover** (Instant Switch):
```bash
# Route 53: Update A record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "aegis.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z789012",
          "DNSName": "green-alb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# DNS propagation: 60 seconds (TTL)
```

**5. Monitor & Rollback Window**:
```bash
# Watch metrics for 1 hour
# If error rate > 0.5%, rollback (switch DNS back to Blue)

# After 1 hour of stable operation:
# - Decommission Blue environment
# - Rename Green → Blue (new baseline)
```

---

## Decision Framework

### How to Choose a Deployment Option

Use this flowchart to select the best option:

```
START
  |
  ├─ Budget < $1,000/mo?
  │   └─ YES → Cloud Run (GCP) or Lambda (AWS)
  │   └─ NO → Continue
  |
  ├─ K8s expertise in team?
  │   └─ YES → GKE or EKS
  │   └─ NO → Continue
  |
  ├─ Data sovereignty required (cannot use cloud)?
  │   └─ YES → On-Premise (VMs or Bare Metal)
  │   └─ NO → Continue
  |
  ├─ Multi-cloud strategy?
  │   └─ YES → GKE (portable) or Hybrid Cloud
  │   └─ NO → Continue
  |
  ├─ Variable/spiky traffic?
  │   └─ YES → Cloud Run, Lambda, or Container Apps
  │   └─ NO → Continue
  |
  ├─ Azure-first organization?
  │   └─ YES → Azure Container Apps or AKS
  │   └─ NO → Continue
  |
  └─ DEFAULT → AWS ECS Fargate (best balance of simplicity + features)
```

### Comparison Matrix

| Factor | AWS ECS | AWS EKS | AWS Lambda | GCP Cloud Run | GCP GKE | Azure Container Apps | On-Premise |
|--------|---------|---------|------------|---------------|---------|----------------------|------------|
| **Ease of Deployment** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **Operational Overhead** | Low | High | Very Low | Very Low | High | Low | Very High |
| **Cost (Small)** | $600 | $730 | $400 | $500 | $650 | $520 | $300* |
| **Cost (Large)** | $11k | $9.5k | $7.5k | $30k | $9k | $18k | $2.5k* |
| **Vendor Lock-In** | High | Medium | Very High | Very High | Low | High | None |
| **Portability** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scaling Speed** | Fast | Medium | Instant | Instant | Medium | Fast | Slow |
| **Compliance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

*On-premise costs exclude CapEx and assume amortization.

---

## Conclusion

This deployment strategy provides comprehensive options for AEGIS AI Gateway across all scales, from startups to enterprises. Key takeaways:

1. **Small deployments (<10M req/mo)**: Choose serverless (Cloud Run, Lambda, Container Apps) for simplicity and low cost.
2. **Medium deployments (10M-100M req/mo)**: Choose managed containers (ECS Fargate, Cloud Run) for balance.
3. **Large deployments (100M-1B+ req/mo)**: Choose Kubernetes (EKS, GKE, AKS) for maximum control and cost efficiency at scale.
4. **Compliance-driven**: On-premise or hybrid cloud for data sovereignty.
5. **Multi-cloud**: GKE or hybrid for vendor independence.

**Next Steps**:
1. Review your organization's requirements (scale, budget, compliance, expertise).
2. Select deployment option using the decision framework.
3. Provision infrastructure (use provided Terraform/IaC examples).
4. Deploy using CI/CD pipeline (GitHub Actions workflow).
5. Test thoroughly (load testing, security scans, compliance audits).
6. Monitor and optimize (cost, performance, security).

**Support**:
- GitHub Issues: https://github.com/kommunication/aegis-ai-gateway/issues
- Documentation: https://docs.aegis-gateway.io
- Slack: #aegis-deployments

**Version History**:
- v1.0 (March 2026): Initial comprehensive deployment strategy

---

*Generated by Jason (Deployment & Infrastructure Expert), March 25, 2026*
