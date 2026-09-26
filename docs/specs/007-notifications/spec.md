# Spec 007 – Notifications (email)

Status: Ready · Depends on: 005, 006 · Module: `notifications`

## Goal
Families get timely, friendly emails for everything that needs action. Emails are never lost or duplicated, even if the app restarts mid-send.

## Design
- **Transactional outbox:** the `notifications` module listens to domain events (`@ApplicationModuleListener`, which runs after commit) and writes an `outbox_message` row.
  - Spring Modulith's event publication registry makes the listener itself reliable.
  - A dispatcher job (every 30 s, ShedLock) sends pending messages, retrying with exponential backoff (1 m, 5 m, 30 m, 2 h, 6 h). After 5 failures the message is marked `FAILED` and appears in admin (009).
- **Sender port:** `EmailSender` has an SMTP implementation (local → Mailpit) and an SES implementation (dev/prod via AWS SDK v2 `SesV2Client`, IAM task role, no keys).
- **Templates:** Thymeleaf HTML + plain-text templates in `resources/templates/email/`, with a shared layout (logo, footer with "why you got this" and preferences link). Plain, warm language.
- **Idempotency:** each message has a `dedupeKey` (e.g. `RESERVATION_READY:{reservationId}`), backed by a unique index.
- **Preferences:** transactional emails are always sent. Reminder emails can be switched off per family (`/account` → Notifications).
- The time zone in all emails is Europe/London.

## Messages
| Trigger | Email | When |
|---|---|---|
| `FamilyRegistered` | Admin: "New family waiting for approval" | Immediately (to admin address from settings) |
| `FamilyApproved` | "Welcome! You can now reserve books" | Immediately |
| `ReservationConfirmed` (HELD) | "We've put *Title* aside for Aarav – choose a collection time by …" | Immediately. Batched: one email per family per 5 minutes |
| `ReservationReady` | "See you Tue 5:00 pm – you're collecting 3 books" | Immediately, one per booking change |
| Slot reminder (job) | "Reminder: library visit today at 5:00 pm" + list | 12:00 on the day of the slot |
| `HoldExpired` | "Your hold on *Title* has ended" | Immediately |
| `SlotBookingCancelled` (closure) | "Sorry – we're closed on … please choose another time" | Immediately |
| Due soon (job) | "3 books due back on Sunday – book a return visit" | 2 days before the due date, 09:00 |
| `LoanOverdue` (008) | "Friendly reminder: … is overdue" | Day 1, day 7 |
| `WaitlistOfferMade` (008) | "Good news! *Title* is now held for Aarav" | Immediately |

## Endpoints / UI
- `GET/PUT /api/v1/me/notification-preferences` (reminders on/off).
- Admin (009 builds the full screen): `GET /api/v1/admin/outbox?status=FAILED`, `POST /api/v1/admin/outbox/{id}/retry`.
- Dev-only `GET /api/v1/admin/email-preview/{template}` renders a template with sample data, for design review.

## Acceptance criteria
- AC1: Locally, reserving a book shows the confirmation email in Mailpit within 1 minute.
- AC2: If SMTP is down, messages stay `PENDING` and send once it's back. There are no duplicates (the test restarts the dispatcher).
- AC3: Reserving 3 books within 5 minutes produces one combined email.
- AC4: The slot reminder is sent once per booking, only for slots with status BOOKED (frozen-clock test).
- AC5: A family with reminders off gets transactional emails but not reminders.
- AC6: In dev, SES (sandbox) delivers to a verified address. Bounces and complaints are routed to SNS and logged (prod setup in 010).
- AC7: Every email has a plain-text part, meets accessible HTML basics, and contains no child surnames (there are none) and no tracking pixels.
- AC8: Template rendering has snapshot tests.

## Open questions
- Q1: The admin notification email address, and the "from" address (e.g. `hello@littletrinity.co.uk`)?
- Q2: Do you want SMS/WhatsApp later? It's out of MVP scope; the outbox design supports adding channels.
