# ADR 0001 – Modular monolith with Spring Boot + Spring Modulith

- Status: Accepted · Date: 2026-09-26

## Context
1–2 maintainers, low traffic now, and it needs to grow to 10k titles and possibly more features. We want enterprise-grade structure without the cost of running microservices.

## Decision
A single Spring Boot 4 (Java 25) deployable, split into business modules (`members`, `catalogue`, `slots`, `reservations`, `loans`, `notifications`, `admin`), with the boundaries enforced by Spring Modulith. Modules communicate through public API packages and domain events.

## Consequences
- One container, one database, and simple operations and cost.
- Clear seams if a module ever needs to be pulled out into its own service.
- A boundary check in CI stops the codebase turning into a "big ball of mud".
- Scaling happens horizontally for the whole app. That is acceptable at the expected load.

## Alternatives considered
Microservices (too much operational cost), and a layered monolith without module rules (erodes over time).
