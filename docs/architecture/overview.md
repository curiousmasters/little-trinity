# Architecture Overview

## System context
```
Parents / Admin (browser, mobile-first)
        │ HTTPS
   Route 53 ─► CloudFront ──┬── /*       ─► S3 (React SPA build, private, Origin Access Control)
                            └── /api/*   ─► ALB ─► ECS Fargate service "api" (Spring Boot, 1..N tasks)
                                                        ├─► RDS PostgreSQL 17 (private subnets)
                                                        ├─► Cognito User Pool (JWT validation via JWKS)
                                                        ├─► SES (email)
                                                        ├─► S3 bucket "media" (book covers)
                                                        └─► Secrets Manager (DB credentials)
   CloudWatch Logs/Alarms · ECR · GitHub Actions (OIDC → IAM role) · Terraform state in S3
```
- Single origin (CloudFront) for UI and API, so there's no CORS in production.
- No NAT Gateway: ECS tasks run in public subnets with public IPs, and their security group only accepts traffic from the ALB. RDS is in private subnets and only accepts traffic from the ECS security group.
- Environments: `local` (docker compose), `dev`, `prod`. Each is its own Terraform workspace/state.

## Backend: modular monolith (Spring Modulith)
| Module | Owns | Publishes events |
|--------|------|------------------|
| `members` | Family, Guardian (Cognito `sub`), ChildProfile, consent, approval | `FamilyApproved`, `FamilyDeleted` |
| `catalogue` | Book (title), BookCopy (physical item), genres, cover storage, ISBN lookup | `CopyBecameAvailable` |
| `slots` | SlotSchedule settings, Slot (date+time, capacity), SlotBooking (family ↔ slot), closures | `SlotBookingCancelled` |
| `reservations` | Reservation (hold on a copy for a child), Waitlist | `ReservationConfirmed`, `HoldExpired`, `WaitlistOfferMade` |
| `loans` | Loan (copy lent to child), due dates, renewals, overdue | `LoanStarted`, `LoanReturned`, `LoanOverdue` |
| `notifications` | Email templates, outbox, SES/SMTP sender | – |
| `admin` | Settings, dashboard queries, reports, CSV import | – |
| `shared` | Clock, ProblemDetail handling, security helpers, pagination | – |

Rules: modules talk only through each other's `api` packages and events. `ApplicationModules.of(App.class).verify()` runs in CI.
Scheduled jobs (hold expiry, reminders, overdue) use `@Scheduled` + **ShedLock** (JDBC) so they run once even with several tasks.
Time comes from an injectable `Clock` (zone Europe/London), so tests can freeze time.

## Data model (logical)
```
family (id, status, display_name, phone, postcode, created_at, approved_at, version)
guardian (id, family_id, cognito_sub UNIQUE, email, full_name, consent_version, consent_at)
child_profile (id, family_id, nickname, birth_year, notes, active)
book (id, isbn13 UNIQUE NULL, title, subtitle, authors[], series_name, series_no, publisher, published_year,
      description, age_min, age_max, language, cover_key, search_vector tsvector, created_at)
genre (id, name) · book_genre (book_id, genre_id)
book_copy (id, book_id, barcode UNIQUE, status[AVAILABLE|ON_HOLD|ON_LOAN|DAMAGED|LOST|WITHDRAWN],
           condition_note, acquired_on, version)
slot (id, date, start_time, end_time, capacity, status[OPEN|CLOSED], UNIQUE(date,start_time), version)
slot_booking (id, slot_id, family_id, status[BOOKED|ATTENDED|NO_SHOW|CANCELLED], UNIQUE(slot_id,family_id) where active)
reservation (id, family_id, child_id, book_id, copy_id NULL, status[WAITLISTED|HELD|READY|COLLECTED|CANCELLED|EXPIRED],
             collection_booking_id NULL, held_at, hold_expires_at, created_at, version)
loan (id, copy_id, child_id, family_id, reservation_id, checked_out_at, due_date, renewals, returned_at,
      return_booking_id NULL, status[ACTIVE|OVERDUE|RETURNED|LOST])
outbox_message (id, type, payload jsonb, status, attempts, next_attempt_at, created_at)
setting (key, value jsonb)   audit_log (id, actor, action, entity, entity_id, at, details jsonb)
shedlock (...)
```

## API conventions
- Base `/api/v1`. The OpenAPI spec is served at `/v3/api-docs` (dev only) and exported to `web/openapi.json` for client generation.
- Pagination: `?page=0&size=20&sort=title,asc` → `{ items, page, size, totalItems, totalPages }`.
- Errors: `application/problem+json` with `type`, `title`, `status`, `detail`, `code`, `errors[]` (field errors).
- Mutating endpoints that users may double-click (reserve, book slot) accept an `Idempotency-Key` header.

## Frontend structure
```
web/src/
  app/          (router, providers, layout, auth guard)
  features/     members/ catalogue/ slots/ reservations/ loans/ admin/
  components/   shared UI (BookCard, EmptyState, ConfirmDialog, SlotPicker…)
  lib/          api client (generated), auth (oidc), query client, date utils (Europe/London)
```
Routes: `/` home · `/books` · `/books/:id` · `/join` · `/account` · `/account/children` · `/my/reservations` ·
`/my/loans` · `/admin/*` (guarded by the ADMIN role).

Auth: Cognito Managed Login with Authorization Code + PKCE through `react-oidc-context`. The access token is sent as a Bearer token.
The API validates JWTs as an OAuth2 resource server and maps the `cognito:groups` claim to roles.

## Design & test automation (applies to every spec)
See ADR 0005.

**Figma → UI.** Figma is the design source of truth. For every spec with UI:
1. Frames for the spec's screens (mobile 375 px first, then desktop 1280 px), including empty, loading and error states, live on that spec's Figma page.
2. The frames are marked **Ready for dev** and linked from `docs/design/README.md` and the spec before the UI tasks start.
3. Components reuse the Figma `Components` page and the tokens (Figma variables → MUI theme). New tokens are added in Figma first, then in `theme.ts`.
4. If the implementation must differ from the design (accessibility, technical limits), note it in the spec and update the frame.

**Test pyramid.**
| Layer | Tool | Where | What it proves |
|-------|------|-------|----------------|
| Unit | JUnit 5 + AssertJ / Vitest + RTL | `api/src/test`, `web/src/**/*.test.tsx` | Domain rules, components, hooks |
| Integration | Testcontainers, `spring-security-test`, MSW | `api/src/test` | Repositories, controllers, security, module boundaries |
| Backend acceptance | **Cucumber** (Gherkin) | `api/src/acceptanceTest` | Business rules (BR-xx) and spec acceptance criteria through the REST API |
| UI end-to-end | **Playwright** (+ axe-core) | `web/e2e` | Critical user journeys in a real browser against the real stack |

**Cucumber conventions.** One feature file per behaviour (`features/<module>/<behaviour>.feature`), written in plain language the library owner can read. No UI wording or database details in steps. Tag scenarios with module and rule IDs (`@slots @BR-02`). Steps drive the public REST API only. Time is set through the test `Clock`. Each scenario sets up its own data.

**Playwright conventions.** One spec file per journey (`e2e/<feature>/<journey>.spec.ts`), page objects in `e2e/pages/`. Role/label locators first. Run on `desktop-chromium` and `mobile-webkit`. Every page visited gets an axe-core check. `@smoke` journeys run on every PR; the full suite runs nightly and before release.

**Definition of done for a spec** (in addition to its acceptance criteria): UI matches the linked Figma frames; each acceptance criterion about behaviour has a Cucumber scenario; each user-facing journey in the spec has a Playwright test.

## Local development
`docker compose up` → postgres:17, mailpit (SMTP 1025 / UI 8025).
The API runs with the `local` profile: SMTP sender goes to Mailpit, covers go to the local filesystem, and JWTs are issued by the dev Cognito pool (spec 001).
Tests never need AWS: they use Testcontainers and mock JWTs.
