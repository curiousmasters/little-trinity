# ADR 0003 – Amazon Cognito for authentication; parents only have accounts

- Status: Accepted · Date: 2026-09-26

## Context
Users are parents and admins. Children are minors, so we want to avoid collecting their credentials and follow the ICO Children's Code. We don't want to build password storage, email verification or password reset ourselves.

## Decision
- A Cognito User Pool with Managed Login (hosted UI), email as the username, and email verification required.
- Groups: `PARENT` (added automatically after sign-up by a post-confirmation step, or on first API call) and `ADMIN` (assigned manually).
- The SPA uses OIDC Authorization Code + PKCE (`react-oidc-context`). The API is an OAuth2 resource server that validates JWTs and maps `cognito:groups` to roles.
- Children are profiles inside a family and have no credentials.
- Application data (family, children) lives in PostgreSQL, linked to the Cognito `sub`.

## Consequences
- Free up to Cognito's free-tier MAU, and no password handling in our code.
- Because the code only relies on OIDC, the identity provider can be swapped (Keycloak, Entra, Auth0) through configuration.
- Integration tests use mocked JWTs. Only manual and e2e testing needs the dev pool.
