# ADR 0002 – AWS: ECS Fargate for API, S3 + CloudFront for web

- Status: Accepted · Date: 2026-09-26

## Context
We want containers and managed, enterprise-standard AWS services, with no servers to patch and low cost at small scale.
AWS App Runner is being discontinued for new customers, so it isn't an option.

## Decision
- The API runs as a container on **ECS Fargate (ARM64/Graviton)** behind an **Application Load Balancer**.
- The React SPA is built into **S3** and served by **CloudFront**. CloudFront also routes `/api/*` to the ALB.
- **RDS PostgreSQL** (db.t4g.micro in dev and at launch) sits in private subnets.
- There is no NAT Gateway (it would cost about £30 a month). Tasks run in public subnets and only accept inbound traffic from the ALB security group.
- All infrastructure is defined in **Terraform**. Deployments come from **GitHub Actions** using OIDC federation, with no long-lived keys.

## Consequences
- Approximately £50–60 a month for prod. Dev can scale to 0 tasks when idle.
- The same container image runs everywhere and could move to EKS later if needed.
- Needs an ACM certificate in us-east-1 for CloudFront.
