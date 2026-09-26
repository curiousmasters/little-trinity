# Spec 000 – Foundations & Walking Skeleton

Status: Ready · Depends on: your repo scaffolding (`api/` Spring Boot, `web/` React + Vite)

## Goal
Turn the bare scaffolding into a well-engineered base: conventions enforced by tooling, a local stack running with one command, CI green, and one thin end-to-end path (UI → API → DB) proving the layers connect.

## Scope
**In:** build tooling, code quality gates, docker compose, database connectivity with Flyway, ProblemDetail error handling, OpenAPI → TS client generation, health endpoint, a "System status" page in the UI, GitHub Actions CI, Dockerfile for the API, the Figma design workflow (design tokens → MUI theme), the Cucumber backend acceptance test harness and the Playwright UI automation harness.
**Out:** authentication (001), AWS (003), any business features.

## Requirements

### UI design (Figma)
Figma is the design source of truth and the input to all UI development. See `docs/architecture/overview.md` → "Design & test automation".
1. A Figma file "Little Trinity" with pages: `Foundations` (design tokens), `Components`, one page per spec (`000 Foundations`, `001 Identity`, …) and `Archive`.
2. Design tokens are Figma **variables**: colour (light mode, WCAG 2.2 AA contrast checked), typography scale, spacing (4 px grid), radius, breakpoints (mobile 375, tablet 768, desktop 1280).
3. Frames for this spec: app shell (header, nav, footer) at mobile and desktop widths, the `/status` page (loading, success, error states) and the 404 page. Each frame is marked **Ready for dev** before its UI task starts.
4. `docs/design/README.md` holds the Figma file link, the page/frame links per spec, and the token → code mapping. No personal data in designs; use sample names only.
5. The MUI theme (`web/src/app/theme.ts`) is built from the Figma tokens. Token names in code match the Figma variable names (e.g. `color.primary.main` → `palette.primary.main`).

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
10. **Cucumber acceptance tests** (behaviour-level, black-box over HTTP):
    - Dependencies: `io.cucumber:cucumber-java`, `cucumber-spring`, `cucumber-junit-platform-engine`, `org.junit.platform:junit-platform-suite`. *Why:* business rules are written as Gherkin scenarios the library owner can read and sign off, and each spec's acceptance criteria map to scenarios.
    - A separate Gradle source set and task `acceptanceTest` (`api/src/acceptanceTest/{java,resources}`), wired into `check` so `./gradlew check` runs it.
    - Feature files in `src/acceptanceTest/resources/features/<module>/*.feature`. Step definitions in `com.littletrinity.acceptance.<module>`.
    - The app starts with `@SpringBootTest(webEnvironment = RANDOM_PORT)` and Testcontainers Postgres; steps call the REST API with `RestClient` (no calls into services or repositories).
    - Shared steps: a controllable `Clock` ("Given today is 2026-10-05 at 16:30"), database reset between scenarios, and (from spec 001) signed-in users via mock JWTs.
    - Scenarios are tagged with the module and business rule IDs (e.g. `@reservations @BR-06`). `@wip` scenarios are excluded from CI.
    - HTML and JSON reports in `build/reports/cucumber/`.
    - First feature: `features/system/system-info.feature` covering `GET /api/v1/system/info` and the unknown-route `404` ProblemDetail.
11. Multi-stage `Dockerfile` (build with Gradle, run on a JRE 25 slim/distroless ARM64 image), running as a non-root user, exposing 8080, with container-aware JVM flags.

### Frontend (`web/`)
1. TypeScript strict mode, ESLint (typescript-eslint, react-hooks, jsx-a11y), Prettier, Vitest + RTL + MSW, Playwright (+ `@axe-core/playwright`).
2. npm scripts: `dev`, `build`, `lint`, `typecheck`, `test`, `e2e`, `gen:api` (openapi-typescript + openapi-fetch from `openapi.json`).
3. App shell built to the Figma frames: MUI theme from the Figma tokens (friendly, high contrast, large tap targets), responsive layout with header, nav and footer, React Router, TanStack Query provider, error boundary, and a 404 page.
4. The Vite dev server proxies `/api` to `http://localhost:8080`, matching production's single origin.
5. A `/status` page that calls `/api/v1/system/info` and shows the version, time and DB status, with loading and error states.
6. **Playwright UI automation:**
   - Tests in `web/e2e/`, organised by feature (`e2e/system/status.spec.ts`), using page objects in `e2e/pages/` and fixtures in `e2e/fixtures/`.
   - Locators by role, label or text (`getByRole`, `getByLabel`); `data-testid` only when there's no accessible alternative.
   - Projects: `desktop-chromium` (1280×800) and `mobile-webkit` (iPhone viewport). Firefox runs in the nightly/full suite only (spec 010).
   - Runs against the real stack (web + API + Postgres), not MSW. `playwright.config.ts` has a `webServer` entry for the web app and reads `E2E_BASE_URL` (default `http://localhost:5173`).
   - Every page test includes an axe-core check that fails on `serious` or `critical` violations.
   - Traces, screenshots and video kept on failure; HTML report in `web/playwright-report/`.
   - Tags: `@smoke` for the fast PR subset, untagged tests in the full suite.
   - First journey: open `/status` → see version, time and "Database: UP"; and an unknown route shows the 404 page.

### Local stack
`docker-compose.yml` at the repo root: `postgres:17` (volume, healthcheck), `mailpit`. A `.env.example` file.
The README has a "Getting started in 5 minutes" section.

### CI (`.github/workflows/ci.yml`)
Runs on PRs and pushes to main. Jobs run in parallel:
- **api:** setup-java 25 (with Gradle cache), then `./gradlew check` (Testcontainers works on GitHub runners), then upload test reports.
- **web:** setup-node LTS, `npm ci`, lint, typecheck, test, build.
- **docker:** build the API image (no push yet).
- **e2e:** start `docker compose` (Postgres, Mailpit), run the API (`local` profile) and the web app, then `npx playwright test`. Upload the Playwright HTML report and traces as artifacts on failure.

The **api** job's `./gradlew check` also runs the Cucumber `acceptanceTest` task; upload `build/reports/cucumber/` as an artifact.
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
- AC10: `./gradlew acceptanceTest` runs the Cucumber features against a Testcontainers database; `system-info.feature` passes, a deliberately broken step fails the build, and an HTML report appears in `build/reports/cucumber/`. `./gradlew check` includes it.
- AC11: With the local stack running, `npm run e2e` passes the status and 404 journeys on `desktop-chromium` and `mobile-webkit`, including the axe-core check. CI's **e2e** job runs them and uploads the report on failure.
- AC12: `docs/design/README.md` links the Figma file and the 000 frames. The MUI theme values match the Figma variables, and the app shell, `/status` and 404 pages match their Figma frames at 375 px and 1280 px.

## Suggested tasks
T1 Gradle quality tooling · T2 profiles, config properties, Clock · T3 Flyway baseline + Testcontainers context test ·
T4 ProblemDetail handling + request ID + JSON logging · T5 system/info endpoint + OpenAPI export task ·
T6 Modulith verification · T7 Dockerfile · T8 web tooling (eslint/prettier/vitest/msw/playwright) ·
T9 app shell + theme + router + query client · T10 OpenAPI client generation + status page ·
T11 docker-compose + README · T12 GitHub Actions CI ·
T13 Figma tokens → `docs/design/README.md` + MUI theme mapping (before T9) ·
T14 Cucumber acceptance test harness + `system-info.feature` · T15 Playwright e2e harness + status/404 journeys + CI e2e job

## Open questions
- Q1: Which base image do you prefer: `eclipse-temurin:25-jre` (easy to debug) or distroless (smaller, harder to debug)? Default: temurin.
- Q2: Who produces the Figma designs (you, a volunteer designer, or Claude drafting from the Figma file via the Figma MCP server)? And is the Figma plan Free or Professional? Dev Mode (inspect, "Ready for dev" status) needs a paid seat; on Free we use a "Ready for dev" section label instead. Default: Free plan with section labels.
- Q3: Should Cucumber scenarios be written/reviewed by you before implementation of each spec (BDD-style, scenarios come first), or written by Claude from the acceptance criteria during the plan step? Default: Claude drafts them in `plan.md`, you approve.
- Q4: For the CI **e2e** job, should the API run from the built Docker image (closer to production, slower) or `bootRun` (faster)? Default: the Docker image from the **docker** job.
