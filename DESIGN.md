
---

#  DESIGN.md

## Architecture Overview

This design implements a production-grade ECS (EC2) deployment focusing on:

- Zero-downtime deployments
- Secure secret management
- Cost-efficient compute
- High availability across multiple AZs

---

## Core Components

### 1. ECS Cluster (EC2 Launch Type)

- Provides container orchestration
- Enables control over compute layer (required for Spot usage)
- Container Insights enabled for monitoring

---

### 2. ECS Service

- Maintains desired number of tasks
- Uses Capacity Provider for infrastructure scaling
- Configured for zero-downtime deployments

---

### 3. Task Definition

- Uses `nginx:latest`
- Secrets injected via SSM Parameter Store
- No secrets stored in Terraform

---

### 4. Application Load Balancer (ALB)

- Deployed in public subnets
- Handles incoming traffic
- Performs health checks
- Redirects HTTP → HTTPS (301)
- Terminates TLS using ACM

---

### 5. Auto Scaling Group (ASG)

- Deployed in private subnets
- Multi-AZ for high availability
- Uses Mixed Instances Policy:
  - On-Demand base capacity (minimum availability)
  - Spot instances for scaling (cost optimization)

---

### 6. Capacity Provider

- Links ECS Service with ASG
- Enables managed scaling
- Automatically adjusts infrastructure based on task demand

---

## Security Design

### Networking

- ALB in public subnets
- ECS instances in private subnets (no public IP)
- Outbound via NAT Gateway

---

### Security Groups

- ALB SG:
  - Allows inbound HTTP/HTTPS from internet

- ECS SG:
  - Allows traffic only from ALB

---

### IAM

- Execution Role:
  - Pulls images, sends logs

- Task Role:
  - Reads secrets from SSM (least privilege)

---

### Secrets Management

- Stored in SSM Parameter Store
- Injected at runtime into container
- Never stored in:
  - Terraform files
  - tfvars
  - state

---

## Zero-Downtime Deployment

Configured via:

- `minimumHealthyPercent = 100`
- `maximumPercent = 200`

Flow:
1. New tasks start
2. Health checks pass
3. Traffic shifts
4. Old tasks drained
5. Old tasks terminated

---

## Cost Optimization

- On-Demand instances for baseline
- Spot instances for scale
- Capacity Provider ensures availability even during Spot interruptions

---

## Monitoring

- ECS metrics (CPU, memory)
- ALB health checks
- CloudWatch logs

---

## Future Improvements

- Add WAF
- Add autoscaling policies
- Use VPC endpoints instead of NAT
- Enable HTTPS between ALB and ECS 

## Trade-offs

### ECS EC2 vs Fargate
- Chose EC2 to enable Spot instances and cost optimization
- Trade-off: More operational overhead (instance management)

### TLS Termination at ALB
- Simpler architecture and lower latency
- Trade-off: Traffic inside VPC is not encrypted

### No Autoscaling Policies
- Kept scope minimal for assignment
- Trade-off: Scaling is reactive via capacity provider only

---

## Production Readiness Improvements

If extended to production, the following enhancements would be implemented:

### 1. Security
- Enable AWS WAF for ALB.
- Use HTTPS between ALB and ECS (end-to-end encryption)
- Add IAM condition policies (resource-level restrictions)

### 2. Networking
- Replace NAT Gateway with VPC Endpoints (cost optimization)
- Add private DNS routing

### 3. Scaling
- Implement ECS Service Auto Scaling (CPU/Memory based)
- Add predictive scaling for ASG

### 4. Observability
- Add structured logging (FireLens)
- Add distributed tracing (AWS X-Ray)

### 5. Reliability
- Add circuit breaker in ECS deployments
- Add multi-region failover (Route53 health checks)

### 6. Cost Optimization
- Use Savings Plans for baseline capacity
- Tune Spot allocation strategy

### 7. AI Agents
- Will implement AI agents for auto remediation of the issues in ecs.