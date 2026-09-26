# Spec 001 – Identity & Family Profiles

Status: Ready · Depends on: 000 · Module: `members`

## Goal
Parents can sign up, sign in, and manage their family and children's profiles. Admins can approve or suspend families. Every later feature relies on "who is the current family, and are they allowed to borrow?"

## Scope
**In:** Cognito dev user pool (Terraform module, applied manually by you), SPA login/logout, API JWT security, family onboarding, child profiles, admin approval, consent capture, data export, and account deletion requests.
**Out:** child logins, social login (possible later), SMS verification.

## Identity setup
- `infra/modules/cognito`:
  - User pool with email as the username, auto-verified email, and a password policy of at least 10 characters.
  - App client (public, PKCE, no secret) with callback URLs `http://localhost:5173/auth/callback` and `https://<domain>/auth/callback`.
  - Groups `PARENT` and `ADMIN`. Managed Login domain.
  - Outputs `issuer_uri`, `client_id` and `domain`.
- `infra/envs/dev` uses the module. **You run `terraform apply`**, not the agent.
- A new user is added to `PARENT` by a Cognito **post-confirmation Lambda** (tiny Node/Python function in `infra/lambdas/`). Alternative, to avoid the Lambda: the API treats any authenticated user without the `ADMIN` group as a parent. **Default: that alternative.** It's simpler, with no Lambda to maintain.

## API security
- Spring Security resource server with `issuer-uri` from config. `cognito:groups` maps to `ROLE_ADMIN`. Every authenticated user also gets `ROLE_USER`.
- `CurrentUser` helper resolves `sub`, `email` and the family ID (if onboarded).
- Public endpoints: `GET /api/v1/system/info`, `GET /api/v1/books/**` (from 002), `GET /api/v1/slots/availability` (from 004). Everything else needs authentication, and `/api/v1/admin/**` needs `ADMIN`.

## Domain
- `Family`:
  - `status`: `PENDING_APPROVAL` → `ACTIVE` ↔ `SUSPENDED`, and `DELETION_REQUESTED` → `DELETED` (anonymised).
  - `displayName` (e.g. "The Kumar family"), `phone` (UK format validated), `postcode` (UK format validated).
- `Guardian`:
  - `cognitoSub` (unique), `email` (copied from the token on each login), `fullName`.
  - Consent `{termsVersion, privacyVersion, acceptedAt}`.
  - MVP has one guardian per family. The model allows a second guardian later.
- `ChildProfile`:
  - `nickname` (1–30 chars), `birthYear` (between current year −16 and current year).
  - `readingNotes` (≤ 200 chars, optional), `active`. A family can have at most 8 children.
- If `littletrinity.members.auto-approve=true`, a new family is `ACTIVE` immediately.
- Events: `FamilyRegistered`, `FamilyApproved`, `FamilySuspended`, `FamilyDeletionRequested`.

## Endpoints
| Method & path | Role | Purpose |
|---|---|---|
| GET `/api/v1/me` | USER | `{ email, roles, family?: {id,status,displayName}, onboardingComplete }` |
| POST `/api/v1/me/family` | USER | Onboard: `{displayName, fullName, phone, postcode, children[], acceptTerms, acceptPrivacy}` → 201. Returns 409 if already onboarded |
| PUT `/api/v1/me/family` | USER | Update contact details |
| GET/POST `/api/v1/me/children` | USER | List / add child |
| PUT/DELETE `/api/v1/me/children/{id}` | USER | Update / deactivate. Deactivation is blocked if the child has active loans or holds (409 `CHILD_HAS_ACTIVE_ITEMS`) |
| GET `/api/v1/me/export` | USER | JSON export of all the family's data |
| POST `/api/v1/me/deletion-request` | USER | Sets `DELETION_REQUESTED`. The admin completes it once no items are out |
| GET `/api/v1/admin/families?status=` | ADMIN | Paged list with search by name, email or postcode |
| GET `/api/v1/admin/families/{id}` | ADMIN | Detail including children |
| POST `/api/v1/admin/families/{id}/approve` · `/suspend` · `/reactivate` · `/anonymise` | ADMIN | State transitions. Invalid transitions return 409 `INVALID_STATE_TRANSITION` |

All admin actions write to `audit_log`.

## UI
- Header shows "Sign in" / "Join" or the user's menu. `/auth/callback` completes login. Silent token renewal.
- `/join` is an onboarding wizard:
  1. Your details.
  2. Your children (add or remove rows).
  3. Agree to the terms and privacy notice (links to static pages).
  4. Done, with a "waiting for approval" message.
- A signed-in user without a family is always redirected to `/join`.
- `/account` has contact details, children (add/edit/deactivate), "Download my data" and "Delete my account".
- `/admin/families` shows tabs (Pending, Active, Suspended), search, and approve/suspend buttons with confirm dialogs.
- Static pages: `/how-it-works`, `/terms`, `/privacy`, containing placeholder text for you to write.

## Acceptance criteria
- AC1: A new user signs up through Cognito, lands on `/join`, completes the wizard and sees "Pending approval". The DB has a family with status `PENDING_APPROVAL`, the consent versions and a timestamp.
- AC2: An admin sees the family under Pending and approves it. The parent's `/me` then shows `ACTIVE`.
- AC3: Parent A can't read or change parent B's family or children. The API returns 404, not 403, to avoid leaking whether IDs exist. Integration tests prove this.
- AC4: A non-admin calling any `/api/v1/admin/**` endpoint gets 403.
- AC5: Invalid phone, postcode or birth year returns 400 with field errors, and the UI shows them inline.
- AC6: A child with active items can't be deactivated (tested once 005/006 exist; stubbed via the `loans`/`reservations` API port until then).
- AC7: The export contains the family, guardian, children, reservations and loans (sections are empty until those modules exist).
- AC8: Anonymising replaces personal data with placeholders and keeps loan statistics.
- AC9: Every endpoint has security tests for anonymous, other-family, parent and admin access.

## Open questions
- Q1: Auto-approve new families, or manual approval? Default: manual.
- Q2: Allow a second guardian per family in MVP? Default: no.
