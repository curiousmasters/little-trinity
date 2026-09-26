# Product Requirements – Little Trinity Kids' Library

Status: Draft v1 · Owner: Rajkumar · Last updated: 2026-09-26

## 1. Vision
A simple, trustworthy website where local families can find children's books, reserve them at any
time, and collect and return them at fixed daily times from a home-based library run by 1–2 people.

## 2. Users & roles
| Role | Has login? | Can do |
|------|-----------|--------|
| Visitor | No | Browse the catalogue, read "how it works", register |
| Parent/Guardian (`PARENT`) | Yes (email + password, Cognito) | Manage family and child profiles, reserve, book slots, renew, see history |
| Child | **No (MVP)**. Profile only. | Reservations are made *for* a child by the parent. A "kid mode" browsing view may come later |
| Librarian/Admin (`ADMIN`) | Yes | Everything: catalogue, copies, slots, check-out/in, approve families, settings, reports |

## 3. Core journeys
1. **Join:** Parent registers → verifies email → completes family profile (phone, postcode, children) → accepts terms and privacy notice → status `PENDING_APPROVAL` → admin approves → `ACTIVE`.
2. **Find:** Anyone searches or filters books by title, author, series, age range, genre and availability.
3. **Reserve (anytime):** Parent picks a child and a book → system holds a copy → parent books a collection slot (today or a future day, 17:00 or 19:00) → confirmation email.
4. **Collect:** At the slot, the admin opens the slot run-sheet, finds the family and scans or ticks copies → loans start and due dates are set.
5. **Return:** Parent books a return slot → admin checks books in at that slot → copies become available (or go to the next waitlisted family).
6. **Waitlist:** If no copy is free, the parent joins the waitlist and is emailed when a copy is held for them.

## 4. Business rules (defaults; admin-configurable where marked ⚙)
| ID | Rule | Default |
|----|------|---------|
| BR-01 | Slot times each open day ⚙ | 17:00 and 19:00 Europe/London, 30-minute window each |
| BR-02 | Families per slot (capacity) ⚙ | 8 |
| BR-03 | Booking horizon ⚙ | Slots can be booked up to 14 days ahead |
| BR-04 | Booking cut-off ⚙ | A slot can't be booked less than 60 minutes before it starts |
| BR-05 | Open days ⚙ | Every day, minus closure dates set by the admin |
| BR-06 | Max active items per child (holds + loans) ⚙ | 3 |
| BR-07 | Max active items per family ⚙ | 10 |
| BR-08 | Loan period ⚙ | 14 days, due at the last slot on the due date |
| BR-09 | Renewals ⚙ | 1 renewal of +14 days, only if nobody is waitlisted for the title and the loan isn't overdue |
| BR-10 | Hold expiry | Once a copy is held, the family must book a collection slot within 48 h ⚙. A no-show at the booked slot keeps the hold until the end of the next open day, then it is released |
| BR-11 | Overdue | A loan becomes overdue the day after its due date. A family with any item overdue more than 7 days ⚙ can't make new reservations |
| BR-12 | Fees/fines | None in MVP (free library). Payments are out of scope |
| BR-13 | Family approval ⚙ | New families need admin approval before reserving (switchable to auto-approve) |
| BR-14 | One family, one booking per slot | A family books one slot per visit. Multiple collections and returns share that booking |
| BR-15 | Waitlist order | First come, first served per title. The held offer lasts 48 h ⚙ |

## 5. Privacy & safeguarding (UK GDPR, ICO Children's Code)
- Children don't have accounts. Store only a child's **first name or nickname**, **birth year** (for age suggestions) and optional reading notes. No photos, surnames or schools.
- The parent account holds the contact data: name, email, phone and **postcode only** (no full address needed, because collection is in person).
- Record explicit acceptance of terms and privacy notice (with version and timestamp).
- Parents can export their data and request account deletion. Loan history is anonymised on deletion.
- Retention: inactive families are anonymised after 24 months ⚙.

## 6. Non-functional requirements
| Area | Target |
|------|--------|
| Availability | 99.5% (single region, eu-west-2) |
| Performance | p95 API < 300 ms; catalogue search < 500 ms at 10k titles |
| Scale | 10k titles, 20k copies, 2k families, 50 concurrent users |
| Security | OWASP ASVS L1, HTTPS only, least-privilege IAM, secrets in AWS Secrets Manager |
| Accessibility | WCAG 2.2 AA |
| Browsers | Last 2 versions of Chrome, Safari (iOS), Edge, Firefox. Mobile-first |
| Backups | RDS automated backups for 7 days (dev 1 day), point-in-time restore |
| Observability | Structured JSON logs, metrics, alarms on 5xx rate, latency, DB CPU/storage |
| Cost | Target under £60/month at launch |

## 7. Out of scope (MVP)
Payments and fines, native mobile apps, SMS/WhatsApp, child logins, multiple branches, delivery.

## 8. Assumptions to confirm
- A1: Free membership, no payments. *(If fees are wanted → add spec 011 Stripe.)*
- A2: Parents manage everything and children have no login.
- A3: MUI for UI components.
- A4: Terraform for infrastructure as code.
- A5: Domain name to be purchased, e.g. `littletrinity.co.uk`.
- A6: UI designs are made in Figma and are the input to UI development.
- A7: Automated testing: Cucumber acceptance tests for the backend, Playwright end-to-end tests for the UI.
