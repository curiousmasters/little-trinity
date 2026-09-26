# CLAUDE.md – Little Trinity

Home-based kids' lending library. Families reserve books online anytime; books are collected and
returned only in scheduled daily slots (17:00 and 19:00, Europe/London). Run by 1–2 volunteers.
Catalogue: ~100 books now, designed for 10,000.

## Read first
- Product rules: `docs/product/requirements.md`
- Architecture & conventions: `docs/architecture/overview.md`
- Decisions: `docs/adr/`
- Current work: `docs/specs/NNN-*/spec.md`, `plan.md`, `tasks.md`

## Working agreement
- Work on **one task from tasks.md at a time**. Stop and summarise when it's done. Don't start the next task unprompted.
- Don't change files outside the task's scope. If you find a problem elsewhere, report it; don't fix it silently.
- If the spec is ambiguous, ask. Don't guess business rules.
- Prefer small, readable code over clever code. No new dependencies without saying why.
- Never commit secrets, `.env` files, or real personal data. Never use production AWS credentials.
- Never run `terraform apply`, `git push`, or anything that touches a remote environment unless explicitly asked.

## Stack
- `api/` – Java 25, Spring Boot 4, Spring Modulith, Spring Data JPA, Flyway, PostgreSQL 17, Gradle (Kotlin DSL)
- `web/` – React 19, TypeScript (strict), Vite, React Router, TanStack Query, React Hook Form + Zod, MUI
- `infra/` – Terraform (AWS: ECS Fargate, RDS, S3, CloudFront, Cognito, SES)
- Local: `docker compose up` → Postgres, Mailpit

## Commands (definition of "verified")
```
# backend
cd api && ./gradlew spotlessApply check        # format, compile, unit + integration tests (Testcontainers), ArchUnit/Modulith
# frontend
cd web && npm run lint && npm run typecheck && npm test
# end-to-end (when stack is running)
cd web && npm run e2e
# api contract → TS client
cd web && npm run gen:api
```
A task is done only when all relevant commands pass and you've shown the output.

## Backend conventions
- Package by module: `com.littletrinity.<module>.{api,application,domain,infrastructure}`.
  Modules: `members`, `catalogue`, `slots`, `reservations`, `loans`, `notifications`, `admin`, `shared`.
- Other modules can only use a module's `api` package (public interfaces/DTOs/events). This is enforced by the Spring Modulith verification test.
- Cross-module side effects go through **domain events** (`ApplicationEventPublisher` + `@ApplicationModuleListener`), not direct calls into other modules' internals.
- Controllers stay thin. Business rules live in domain/application services and get unit tests.
- REST under `/api/v1`. JSON camelCase. IDs are UUIDs. Timestamps are `Instant` (UTC) in the API; dates/slots are `LocalDate`/`LocalTime` in Europe/London.
- Errors use RFC 9457 `ProblemDetail` with a stable `code` property (e.g. `RESERVATION_LIMIT_REACHED`).
- Every schema change is a new Flyway migration `V{yyyyMMddHHmm}__description.sql`. Never edit an applied migration.
- Entities that face concurrent updates use `@Version` (optimistic locking).
- Validate input with Jakarta Validation on request DTOs. Never expose entities directly.
- Security: every endpoint must declare access (`PARENT`, `ADMIN`, or public). Parents can only touch their own family's data. Test that.
- Config goes through typed `@ConfigurationProperties` (`littletrinity.*`), not scattered `@Value`.
- Tests: JUnit 5, AssertJ, Testcontainers Postgres for repository and integration tests, `spring-security-test` `jwt()` for auth. Name tests after behaviour: `shouldRejectReservationWhenChildLimitReached`.

## Frontend conventions
- Feature folders: `src/features/<module>/{components,pages,hooks,api}`. Shared UI in `src/components`.
- Server state with TanStack Query only (no ad-hoc fetch in components). API types come from the generated OpenAPI client.
- Forms: React Hook Form + Zod schemas that mirror backend validation.
- Accessible by default: semantic elements, labels, keyboard navigation, colour contrast (WCAG 2.2 AA).
- Mobile-first layouts. Friendly, simple language (users include children).
- Tests: Vitest + React Testing Library for components/hooks. MSW for API mocking. Playwright for critical journeys.

## Git
- Conventional commits: `feat(module): …`, `fix(module): …`, `docs(NNN): …`, `test(module): …`, `chore: …`.
- Suggest a commit message when a task finishes. Don't commit unless asked.
