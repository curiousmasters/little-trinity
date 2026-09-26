# Spec 000 – Foundations & Walking Skeleton

Status: Ready · Depends on: your repo scaffolding (`api/` Spring Boot, `web/` React + Vite)

## Goal
Turn the bare scaffolding into a well-engineered base: conventions enforced by tooling, a local stack running with one command, CI green, and one thin end-to-end path (UI → API → DB) proving the layers connect.

## Scope
**In:** build tooling, code quality gates, docker compose, database connectivity with Flyway, ProblemDetail error handling, OpenAPI → TS client generation, health endpoint, a "System status" page in the UI, GitHub Actions CI, Dockerfile for the API.
**Out:** authentication (001), AWS (003), any business features.

## Requirements

### Backend (`api/`)
1. Gradle Kotlin DSL, Java 25 toolchain. Dependencies: web, validation, data-jpa, flyway, postgresql, actuator, springdoc-openapi, spring-modulith (core + test), shedlock, testcontainers, archunit.
2. **Spotless** (palantir-java-format) and **Error Prone** or Checkstyle. `./gradlew check` fails on violations.
3. Profiles: `local`, `test`, `dev`, `prod`. `application.yml` holds shared settings. Secrets come only from environment variables.
4. Flyway baseline migration creating `setting`, `audit_log` and `shedlock` tables.
5. `shared` module:
   - `GlobalExceptionHandler` returning RFC 9457 ProblemDetail with a `code` property and field `errors[]`.
   - `ClockConfig` providing `Clock` in the Europe/London zone.
   - A `PageResponse<T>` record for pagination.
   - `LittleTrinityProperties` (`@ConfigurationProperties("littletrinity")`).
6. `GET /api/v1/system/info` (public) → `{ "name": "little-trinity", "version": "<git sha or build version>", "time": "<Instant>", "database": "UP" }`.
7. Actuator: only `health` (with liveness/readiness groups) and `info` are exposed. `/actuator/health` doesn't show details publicly.
8. Structured JSON logging (Spring Boot's built-in structured logging, ECS format) in `dev`/`prod`, plain text in `local`. Every request log includes a request ID (`X-Request-Id`, generated if missing and echoed in the response).
9. `ModularityTests` running `ApplicationModules.of(...).verify()`.
10. Multi-stage `Dockerfile` (build with Gradle, run on a JRE 25 slim/distroless ARM64 image), running as a non-root user, exposing 8080, with container-aware JVM flags.

### Frontend (`web/`)
1. TypeScript strict mode, ESLint (typescript-eslint, react-hooks, jsx-a11y), Prettier, Vitest + RTL + MSW, Playwright.
2. npm scripts: `dev`, `build`, `lint`, `typecheck`, `test`, `e2e`, `gen:api` (openapi-typescript + openapi-fetch from `openapi.json`).
3. App shell: MUI theme (friendly, high contrast, large tap targets), responsive layout with header, nav and footer, React Router, TanStack Query provider, error boundary, and a 404 page.
4. The Vite dev server proxies `/api` to `http://localhost:8080`, matching production's single origin.
5. A `/status` page that calls `/api/v1/system/info` and shows the version, time and DB status, with loading and error states.

### Local stack
`docker-compose.yml` at the repo root: `postgres:17` (volume, healthcheck), `mailpit`. A `.env.example` file.
The README has a "Getting started in 5 minutes" section.

### CI (`.github/workflows/ci.yml`)
Runs on PRs and pushes to main. Jobs run in parallel:
- **api:** setup-java 25 (with Gradle cache), then `./gradlew check` (Testcontainers works on GitHub runners), then upload test reports.
- **web:** setup-node LTS, `npm ci`, lint, typecheck, test, build.
- **docker:** build the API image (no push yet).
- **security:** dependency review on PRs, plus Trivy filesystem scan (fails on HIGH/CRITICAL with a fix available).

Branch protection on `main`: PR required and CI must pass. You configure this in GitHub.

## Acceptance criteria
- AC1: `docker compose up -d && (cd api && ./gradlew bootRun --args='--spring.profiles.active=local')` starts the API against local Postgres, and Flyway applies migrations.
- AC2: `cd web && npm run dev` → opening `http://localhost:5173/status` shows version, time and "Database: UP".
- AC3: Stopping Postgres makes `/status` show a friendly error state, not a blank page.
- AC4: An unknown API route returns a `404` ProblemDetail JSON with a `code` of `NOT_FOUND`.
- AC5: Invalid request bodies return `400` with field-level `errors[]` (proven by a test controller in the test scope).
- AC6: `./gradlew check` runs formatting, tests (including a Testcontainers-backed context test) and the Modulith verification.
- AC7: `npm run gen:api` regenerates typed client code from the exported OpenAPI file, and the status page uses it.
- AC8: CI passes on a PR, and a deliberately badly formatted file makes CI fail.
- AC9: `docker build` produces an image under 250 MB that runs as a non-root user and answers `/actuator/health/readiness`.

## Suggested tasks
T1 Gradle quality tooling · T2 profiles, config properties, Clock · T3 Flyway baseline + Testcontainers context test ·
T4 ProblemDetail handling + request ID + JSON logging · T5 system/info endpoint + OpenAPI export task ·
T6 Modulith verification · T7 Dockerfile · T8 web tooling (eslint/prettier/vitest/msw/playwright) ·
T9 app shell + theme + router + query client · T10 OpenAPI client generation + status page ·
T11 docker-compose + README · T12 GitHub Actions CI

## Open questions
- Q1: Which base image do you prefer: `eclipse-temurin:25-jre` (easy to debug) or distroless (smaller, harder to debug)? Default: temurin.
