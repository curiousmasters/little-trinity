# Spec 003 – AWS Infrastructure & Continuous Delivery (dev)

Status: Ready · Depends on: 000 (best done after 002 so there's something worth deploying)

## Goal
Every merge to `main` deploys automatically to a real AWS **dev** environment. The same Terraform modules are reused for `prod` in spec 010.
**The agent writes Terraform and workflows. You run `terraform plan/apply`.**

## Terraform layout
```
infra/
  bootstrap/            # one-off: S3 state bucket (versioned, encrypted), GitHub OIDC provider + deploy role
  modules/
    network/            # VPC 10.20.0.0/16, 2 public + 2 private subnets (2 AZs), no NAT, S3 gateway endpoint
    ecr/                # repo "little-trinity-api", scan on push, lifecycle: keep last 20 images
    rds/                # PostgreSQL 17, db.t4g.micro, gp3 20 GB (autoscale to 100), encrypted, private,
                        #   master password managed by RDS in Secrets Manager, backups (dev 1 d / prod 7 d),
                        #   deletion protection in prod, Performance Insights free tier
    ecs_api/            # cluster, task def (ARM64, 0.5 vCPU / 1 GB), service (desired 1, min 1, max 3,
                        #   CPU target 60%), ALB + HTTPS listener + target group (health /actuator/health/readiness),
                        #   task role (S3 media, SES send, Secrets read), log group (dev 14 d / prod 90 d)
    web/                # S3 (private) + CloudFront with OAC; behaviours: /api/* → ALB (no cache, all headers),
                        #   /media/* → media bucket, default → SPA with 403/404 → /index.html; security headers policy
    media/              # S3 bucket for covers
    cognito/            # from spec 001
    ses/                # domain identity with DKIM, MAIL FROM; (sandbox in dev)
    dns/                # Route 53 zone records, ACM certs (us-east-1 for CloudFront, eu-west-2 for ALB)
    monitoring/         # alarms: ALB 5xx > 1%, p95 > 1 s, ECS CPU > 80%, RDS CPU > 80%, RDS free storage < 2 GB;
                        #   SNS topic → your email; AWS Budget alarm at £40 and £60
  envs/
    dev/  (main.tf, variables.tf, terraform.tfvars, backend.tf)
    prod/ (created in 010)
```
- Region `eu-west-2`, with CloudFront certs in `us-east-1` via a provider alias.
- State in S3 using native locking (`use_lockfile = true`).
- Tagging: `project=little-trinity`, `env`, `managed-by=terraform`.
- `tflint`, `terraform fmt -check`, `terraform validate` and `checkov` (or `trivy config`) run in CI.

## Delivery pipeline (`.github/workflows/deploy-dev.yml`)
Trigger: push to `main` (after CI passes) and manual dispatch.
1. Assume `gh-deploy-dev` via OIDC. The role trust is limited to `repo:<you>/little-trinity:ref:refs/heads/main`.
2. **API:**
   - Build the ARM64 image (buildx), tag it with the git SHA and push to ECR.
   - Render the task definition with the new image, then `aws ecs deploy`-style update, waiting for the service to stabilise.
   - The ECS deployment circuit breaker has rollback enabled.
3. **Web:** `npm ci && npm run build` with `VITE_*` config (Cognito domain, client ID). `aws s3 sync --delete` with long cache for hashed assets and `no-cache` for `index.html`, then CloudFront invalidation of `/index.html`.
4. **Smoke test:** `curl https://dev.<domain>/api/v1/system/info` must return the new SHA.

Terraform changes are **not** applied by the pipeline in MVP. A separate `infra-plan.yml` posts the `terraform plan` as a PR comment, and you apply manually. (Automated apply is a later improvement.)

## Configuration & secrets
- The API gets `SPRING_PROFILES_ACTIVE=dev`, the DB host and name, the Cognito issuer, the S3 bucket and the SES from-address as plain environment variables. The DB username and password come as ECS `secrets` from Secrets Manager.
- Flyway runs on startup with a single task, which is safe. Revisit this if running more than one task during a deploy.
- No secrets in the repo or in GitHub, apart from the account ID and role ARN as repo variables.

## Cost guardrails
- Dev ECS service can be scaled to 0 by a scheduled GitHub workflow at night (optional). RDS dev instance is stopped when not needed (manual).
- Documented in `docs/runbooks/cost.md`.

## Acceptance criteria
- AC1: `terraform apply` in `infra/bootstrap` then `infra/envs/dev` creates the environment from nothing, with no manual console clicks (apart from buying the domain and confirming the SNS email).
- AC2: Merging a PR changes the version shown on `https://dev.<domain>/status` within about 10 minutes.
- AC3: A broken image (failing readiness) is rolled back automatically and the workflow fails.
- AC4: RDS isn't reachable from the internet (verified by a security group rule review and a connection attempt).
- AC5: The SPA deep link `https://dev.<domain>/books/123` loads (SPA fallback works), and `/api/*` isn't cached.
- AC6: Security headers are present: HSTS, X-Content-Type-Options, frame-ancestors none, and a CSP allowing self, the Cognito domain and the media host.
- AC7: `checkov`/`trivy config` passes with no HIGH findings, or each exception is documented.
- AC8: The monthly cost estimate (AWS Pricing Calculator or `infracost`) is recorded in `docs/runbooks/cost.md`.

## Open questions
- Q1: Domain name and registrar (Route 53 or external)?
- Q2: Should dev be publicly reachable, or restricted (CloudFront + WAF IP allow-list or basic auth)? Default: public but not advertised.
