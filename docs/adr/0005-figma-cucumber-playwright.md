# ADR 0005 – Figma for UI design, Cucumber and Playwright for test automation

- Status: Accepted · Date: 2026-09-26

## Context
The UI is used by parents, children and volunteers, so it has to look consistent and be accessible. Business rules (limits, slots, holds, overdue) are the riskiest part of the system and should be readable by the library owner, not only by developers. The main user journeys must keep working as specs are added.

## Decision
- **Figma** is the source of truth for UI design. Design tokens are Figma variables mirrored in the MUI theme. Each spec's UI tasks start only from frames marked "Ready for dev".
- **Cucumber-JVM** (with `cucumber-spring` on the JUnit Platform) runs backend acceptance tests. Gherkin scenarios drive the REST API of the running app against Testcontainers Postgres, and are tagged with business rule IDs.
- **Playwright** (with axe-core) runs UI end-to-end tests in real browsers against the real stack, on desktop and mobile viewports.

## Consequences
- Designs, business rules and journeys are traceable: Figma frame → spec → Cucumber scenario / Playwright test.
- Extra dependencies (Cucumber, axe-core) and a slower CI (acceptance + e2e jobs). The PR run uses the `@smoke` Playwright subset; the full suite runs nightly.
- Cucumber and Playwright overlap in places. Rule: business rules and edge cases go in Cucumber (fast, no browser); Playwright covers only user journeys and accessibility.
- Figma needs a seat for whoever designs; Dev Mode needs a paid plan (see spec 000, Q2).
