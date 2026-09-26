# Spec 010 – Production Readiness & Go-Live

Status: Ready · Depends on: all previous specs

## Goal
Launch a production environment that is secure, observable, recoverable, legally sound and affordable, with runbooks so either volunteer can operate it.

## Workstreams

### 1. Production environment
- `infra/envs/prod` reuses the modules with these differences:
  - RDS: 7-day backups, deletion protection, final snapshot. Multi-AZ off at launch (cost), documented as an upgrade.
  - ECS: min 1, max 3 tasks. SES production access requested (out of sandbox), with bounce/complaint handling via SNS → API endpoint → suppress address.
- `deploy-prod.yml`: manual trigger or on release tag `v*`, requiring a **GitHub Environment approval** (you). It deploys the **same image digest** already verified in dev, with no rebuild.
- Seed the prod admin user. Import the real catalogue using 009.

### 2. Security (OWASP ASVS L1)
- A threat-model review of each module, written up in `docs/security/threat-model.md`.
- Rate limiting:
  - CloudFront + **AWS WAF** managed rule sets (Core rule set, Known bad inputs, IP reputation).
  - Per-IP rate rule on `/api/*` (e.g. 300 requests per 5 minutes).
  - App-level rate limiting on reservation endpoints (Bucket4j) per user.
- Dependency updates with Dependabot (weekly grouped PRs). Trivy image scan blocks deploys on CRITICAL findings.
- OWASP ZAP baseline scan against dev in CI, run nightly. Fix findings rated medium or above.
- Security headers and CSP verified. Cookies aren't used (bearer tokens in memory, refreshed via the OIDC library).
- IAM least privilege reviewed. No `*` actions apart from documented exceptions.
- Admin accounts use Cognito MFA (TOTP), which is mandatory for the `ADMIN` group.

### 3. Privacy & legal (UK GDPR, ICO Children's Code)
- Final **Privacy notice** and **Terms** pages. Record their versions, and ask existing users to re-consent when they change.
- Record of processing activities (short), a data retention job (BR: anonymise inactive after 24 months), and a tested export/deletion flow.
- Cookie banner not needed if no non-essential cookies or analytics. If you add analytics, use a cookieless tool (e.g. Plausible) and update the privacy notice.
- Check whether ICO data protection fee registration is required, and record the outcome.

### 4. Observability & operations
- Dashboards (CloudWatch): requests, 5xx, latency p95, ECS CPU/memory, RDS CPU/connections/storage, outbox pending/failed, job last-run times.
- Custom metrics via Micrometer (CloudWatch registry): `reservations.created`, `loans.checked_out`, `emails.failed`, `jobs.<name>.duration`.
- Alarms go to email via SNS, per spec 003, plus: a job hasn't run for 2× its interval, outbox FAILED > 0, and an external uptime check (Route 53 health check on `/api/v1/system/info`).
- Logs: JSON with `requestId`, `userSubHash` (never the email), 90-day retention.

### 5. Resilience & recovery
- **Backups:** RDS PITR plus a weekly logical `pg_dump` to S3 (Glacier after 30 days). Media bucket versioning on.
- **Restore drill:** restore a prod snapshot into a temporary instance, run smoke checks and record the time taken. Target RTO 4 h, RPO 15 min.
- **Load test (k6):**
  - 50 virtual users browsing and searching, and 10 reserving at the same time.
  - Pass criteria: p95 < 300 ms, 0 errors, no overbooking.
  - Scripts live in `perf/`.

### 6. Quality gates
- Coverage targets: domain/application ≥ 80% lines, with PIT mutation testing on reservations, slots and loans (report only).
- Full Playwright suite in CI against docker compose: join, approve, search, reserve, collect, return, waitlist, renew.
- Full Cucumber acceptance suite green, with every business rule BR-01…BR-15 covered by at least one tagged scenario (a CI check lists untagged rules).
- Playwright full suite also runs on Firefox, nightly against dev.
- UI review against the Figma frames for every main page at 375 px and 1280 px; differences are fixed or recorded.
- Accessibility: axe-core checks in Playwright on all main pages, plus a manual keyboard and screen reader pass (VoiceOver) on the reserve and desk journeys.
- Lighthouse on the home page, `/books` and book detail: Performance ≥ 90, Accessibility ≥ 95.

### 7. Documentation & runbooks (`docs/runbooks/`)
`deploy.md` (normal deploy, rollback to the previous task definition), `restore.md`, `add-admin.md`,
`rotate-secrets.md`, `incident.md` (what to check first), `cost.md`, `daily-operations.md` (a non-technical guide for volunteers: approving families, running a slot at the desk, handling lost books).

## Go-live checklist
- [ ] All acceptance criteria for specs 000–009 verified in dev
- [ ] Prod infra applied, DNS live, TLS valid, WAF on
- [ ] SES production access granted, SPF/DKIM/DMARC passing
- [ ] Admin MFA on, test family created and removed
- [ ] Real catalogue imported and labels printed and stuck on books
- [ ] Privacy notice and terms published
- [ ] Backups verified and restore drill done
- [ ] Alarms tested (trigger one on purpose)
- [ ] Budget alarm set
- [ ] Soft launch to 5 friendly families for 2 weeks, then an open announcement

## Acceptance criteria
- AC1: A tagged release deploys to prod only after your approval, using the same image digest as dev.
- AC2: A ZAP baseline scan and Trivy show no unresolved High/Critical findings.
- AC3: The k6 load test meets its targets on a prod-sized environment.
- AC4: The restore drill is documented, with the actual RTO recorded.
- AC5: The axe scan shows zero serious or critical violations.
- AC6: A volunteer who hasn't seen the system before can run a slot using only `daily-operations.md`.
