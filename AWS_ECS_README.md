# 🚀 AWS ECS — Elastic Container Service
### Complete Production-Ready Reference | Enterprise Guide

[![AWS ECS](https://img.shields.io/badge/AWS-ECS-FF9900?style=flat&logo=amazonaws)](https://aws.amazon.com/ecs/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?style=flat&logo=terraform)](https://terraform.io)
[![Docker](https://img.shields.io/badge/Container-Docker-2496ED?style=flat&logo=docker)](https://docker.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📋 Table of Contents
1. [What is AWS ECS?](#what-is-aws-ecs)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [GUI — Step-by-Step Setup](#gui--step-by-step-setup)
5. [How It Works](#how-it-works)
6. [Benefits & Features](#benefits--features)
7. [POC — Quick Deploy](#poc--quick-deploy)
8. [SOP Documentation](#sop-documentation)
9. [Costing](#costing)
10. [CI/CD Integration](#cicd-integration)

---

## What is AWS ECS?

**AWS ECS (Elastic Container Service)** is a fully managed container orchestration service that runs, stops, and manages Docker containers on a cluster of EC2 instances or serverless Fargate.

> **Interview Answer:** *"ECS is AWS's native container platform. It abstracts the complexity of container orchestration — you define what to run (Task Definition), how many copies (Service), and ECS handles placement, health monitoring, scaling, and rolling deployments. It's simpler than Kubernetes but production-grade for most workloads."*

---

## Architecture Overview

```
INTERNET
    │
    ▼
Route 53 ──► WAF ──► CloudFront
                          │
                    ┌─────▼──────┐
                    │    ALB      │
                    └─────┬──────┘
              ┌───────────┴───────────┐
              ▼                       ▼
    ┌─── AZ us-east-1a ───┐  ┌─── AZ us-east-1b ───┐
    │  ECS Fargate Tasks  │  │  ECS Fargate Tasks   │
    │  [App][App][App]    │  │  [App][App][App]     │
    └─────────────────────┘  └──────────────────────┘
              │                       │
    ┌─────────▼───────────────────────▼────────┐
    │         Supporting Services               │
    │  RDS (Multi-AZ) │ ElastiCache │ S3 │ ECR │
    └──────────────────────────────────────────┘
    
    ┌──────────── Observability ────────────────┐
    │  CloudWatch │ X-Ray │ Container Insights   │
    └───────────────────────────────────────────┘
    
    ┌──────────────── Security ─────────────────┐
    │  IAM Roles │ Secrets Manager │ VPC │ WAF  │
    └───────────────────────────────────────────┘
```

---

## Core Components

| Component | Description | Key Points |
|-----------|-------------|------------|
| **Cluster** | Logical group of compute resources | Can mix EC2 + Fargate |
| **Task Definition** | Container blueprint (image, CPU, RAM, ports) | Versioned, immutable revisions |
| **Task** | Running instance of a Task Definition | Ephemeral, like a K8s Pod |
| **Service** | Maintains N running tasks + LB integration | Handles rolling updates |
| **ECR** | Private Docker registry | Native ECS integration |
| **Fargate** | Serverless compute mode | No EC2 management |

---

## GUI — Step-by-Step Setup

### Step 1: Create ECR Repository
```
AWS Console → ECR → Create Repository
  Name: my-app
  Visibility: Private
  Image scanning: Enable on push
```
Then push your Docker image:
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com

docker build -t my-app .
docker tag my-app:latest <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
docker push <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### Step 2: Create ECS Cluster
```
AWS Console → ECS → Clusters → Create Cluster
  Name: my-production-cluster
  Infrastructure: ✅ AWS Fargate (serverless)
  Monitoring: ✅ Container Insights (CloudWatch)
  Tags: Environment=production
```

### Step 3: Create Task Definition
```
ECS → Task Definitions → Create New
  Family Name: my-app-task
  Launch Type: AWS Fargate
  CPU: 0.5 vCPU | Memory: 1 GB
  Task Role: ecsTaskExecutionRole
  
  Container:
    Name: my-app
    Image: <ECR URI>:latest
    Port: 80/tcp
    Logging: awslogs (auto)
    Environment Variables: KEY=VALUE
```

### Step 4: Create Application Load Balancer
```
EC2 → Load Balancers → Create ALB
  Name: my-app-alb
  Scheme: Internet-facing
  Subnets: Select 2+ public subnets
  Security Group: Allow HTTP 80 from 0.0.0.0/0
  Target Group: Type=IP, Port=80, Health=/health
```

### Step 5: Create ECS Service
```
ECS → Cluster → Services → Create
  Task Definition: my-app-task (LATEST)
  Service Name: my-app-service
  Desired Tasks: 2
  Deployment: Rolling Update
  
  Networking:
    VPC: your-vpc
    Subnets: private subnets
    Security Groups: ecs-sg (allow 80 from ALB)
  
  Load Balancing:
    ALB: my-app-alb
    Target Group: my-app-tg
  
  Auto Scaling:
    Min: 2 | Desired: 2 | Max: 10
    Scale-out: CPU > 70% for 2 mins
    Scale-in: CPU < 30% for 5 mins
```

✅ **Result:** Access your app via the ALB DNS name within ~3 minutes.

---

## How It Works

### Request Flow
```
User Request
    │
    ▼
Route 53 DNS Resolution
    │
    ▼
ALB (Health-checks tasks every 30s)
    │
    ▼
ECS Task (Container running your app)
    │
    ├── Logs → CloudWatch Log Groups
    ├── Metrics → CloudWatch Container Insights
    └── Traces → AWS X-Ray
    
ECS Service Controller (monitors continuously)
    ├── Replaces unhealthy tasks automatically
    ├── Maintains desired task count
    └── Handles rolling updates (zero downtime)
    
Auto Scaling
    ├── CloudWatch alarm: CPU > 70%
    ├── Application Auto Scaling adds tasks
    └── ALB registers new tasks automatically
```

### Rolling Deployment Flow
```
New image pushed → Update Task Definition (new revision)
    │
    ▼
ECS starts NEW tasks with new revision
    │
    ▼
ALB health checks pass for new tasks
    │
    ▼
Old tasks are drained (connections complete)
    │
    ▼
Old tasks stopped
    ✅ Zero downtime achieved
```

---

## Benefits & Features

| Feature | Description | Business Value |
|---------|-------------|----------------|
| 🔧 **Fully Managed** | No control plane to manage | 60% less DevOps overhead |
| ⚡ **Fargate Serverless** | No EC2 provisioning | Zero server management |
| 📈 **Auto Scaling** | CPU/memory-based scaling | Handle traffic spikes |
| 🔗 **AWS Native** | IAM, ALB, CloudWatch built-in | Faster development |
| 🔄 **Rolling Deploys** | Zero-downtime updates | Production-safe releases |
| 💰 **Fargate Spot** | Up to 70% cost savings | Reduce cloud spend |
| 🌍 **Multi-Region** | Global deployment | Low latency worldwide |
| 🔵🟢 **Blue/Green** | Instant rollback via CodeDeploy | Risk-free deployments |
| 🔍 **Container Insights** | Deep CloudWatch metrics | Full observability |
| 🔐 **Task IAM Roles** | Per-container permissions | Zero shared credentials |

---

## POC — Quick Deploy

### Terraform (10 min deploy)
```hcl
# main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "us-east-1" }

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "poc-cluster"
  setting { name = "containerInsights", value = "enabled" }
}

# IAM Task Execution Role
resource "aws_iam_role" "exec" {
  name = "ecsTaskExecutionRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "exec_policy" {
  role       = aws_iam_role.exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# CloudWatch Logs
resource "aws_cloudwatch_log_group" "ecs" {
  name              = "/ecs/poc-app"
  retention_in_days = 7
}

# Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "poc-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.exec.arn

  container_definitions = jsonencode([{
    name      = "nginx"
    image     = "nginx:latest"
    essential = true
    portMappings = [{ containerPort = 80 }]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/poc-app"
        "awslogs-region"        = "us-east-1"
        "awslogs-stream-prefix" = "ecs"
      }
    }
  }])
}
```

```bash
# Deploy
terraform init
terraform plan
terraform apply -auto-approve

# Cleanup
terraform destroy -auto-approve
```

---

## SOP Documentation

### SOP-001: Production Deployment
```
Trigger:    New release ready for production
Owner:      DevOps Team
SLA:        Deploy within 30 minutes of approval

Pre-checks:
  [ ] Docker image scanned and pushed to ECR
  [ ] Task Definition reviewed (CPU/memory correct)
  [ ] Health check endpoint tested
  [ ] Rollback plan ready

Steps:
  1. Update Task Definition → create new revision
  2. Update ECS Service → select new revision
  3. Monitor: ECS Console → Services → Deployments
  4. Verify: All tasks RUNNING (max 5 min)
  5. Smoke test: curl https://<ALB-DNS>/health
  6. Monitor CloudWatch for 15 minutes post-deploy

Rollback (if needed):
  ECS Service → Update → Previous Task Definition revision
  Time to rollback: ~3-5 minutes
```

### SOP-002: Incident Response
```
P1 (All tasks stopped):    15 min SLA
P2 (>50% unhealthy):       30 min SLA
P3 (Single task failure):   2 hr SLA

Investigation Steps:
  1. ECS → Cluster → Stopped Tasks → Stopped Reason
  2. CloudWatch → /ecs/<service-name> → container logs
  3. Common causes:
     • OOMKilled        → Increase task memory
     • App crash        → Fix code, redeploy
     • CannotPullImage  → Fix ECR permissions
     • ResourceInit Err → Fix IAM task role
```

---

## Costing

### Fargate Pricing (us-east-1)
| Resource | Price |
|----------|-------|
| vCPU per hour | $0.04048 |
| GB Memory per hour | $0.004445 |
| Fargate Spot vCPU | $0.01173 (~70% off) |
| ECR Storage per GB/mo | $0.10 |
| ALB per hour | $0.008 |

### Monthly Estimates
| Scenario | Config | Monthly Cost |
|----------|--------|-------------|
| Dev/Test | 2 tasks × 0.25vCPU × 0.5GB | ~$15-20 |
| Small Prod | 4 tasks × 0.5vCPU × 1GB + ALB | ~$60-80 |
| Enterprise | 20 tasks × 1vCPU × 2GB + ALB | ~$350-450 |
| With Spot | Same as Small Prod on Spot | ~$20-25 |

**💡 Savings Tips:**
- Use Fargate Spot for batch jobs → 70% savings
- Fargate Compute Savings Plans → 52% savings
- Right-size tasks using CloudWatch metrics
- Consolidate services behind one ALB

---

## CI/CD Integration

```yaml
# .github/workflows/deploy.yml
name: Deploy to ECS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/my-app:$IMAGE_TAG .
          docker push $ECR_REGISTRY/my-app:$IMAGE_TAG

      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster my-production-cluster \
            --service my-app-service \
            --force-new-deployment
```

---

## Project Structure
```
.
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
│       ├── ecs/
│       ├── alb/
│       └── ecr/
├── docker/
│   └── Dockerfile
├── .github/
│   └── workflows/
│       └── deploy.yml
└── docs/
    ├── architecture.md
    └── runbook.md
```

---

## Resources
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [Fargate Pricing](https://aws.amazon.com/fargate/pricing/)
- [AWS CDK ECS Patterns](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs_patterns-readme.html)

---

*Production-ready ECS guide | Built for Enterprise Teams | v1.0*
