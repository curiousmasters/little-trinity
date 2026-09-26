# Spec 004 – Collection & Return Slots

Status: Ready · Depends on: 001 · Module: `slots`

## Goal
Model the library's opening times as bookable slots (daily 17:00 and 19:00) with limited capacity, so families can book when to come, and the admin can see who's coming and close days.

## Domain
- **Slot schedule settings** (`setting` table, key `slots.schedule`):
  - `times` (default `["17:00","19:00"]`), `durationMinutes` (30), `capacity` (8).
  - `horizonDays` (14), `cutoffMinutes` (60), `weekdaysOpen` (all 7).
- **Slot:** `date`, `startTime`, `endTime`, `capacity`, `status` (`OPEN`|`CLOSED`), `closureReason`, `version`.
  Slots are **materialised** by a daily job (02:00) for the next `horizonDays`, and also on startup and when settings change. This keeps queries and locking simple.
- **SlotBooking:**
  - `slotId`, `familyId`, `purposes` (set of `COLLECT`, `RETURN`), `status` (`BOOKED`|`ATTENDED`|`NO_SHOW`|`CANCELLED`), `createdAt`.
  - A family has at most one active booking per slot (partial unique index).
- Capacity counts active bookings per slot. It's enforced with a transactional check under a pessimistic row lock on the slot (`SELECT … FOR UPDATE`). A concurrency test proves no overbooking.
- Rules:
  - Can't book a `CLOSED` slot, a slot past the cut-off, or a slot beyond the horizon.
  - The family must be `ACTIVE`.
  - Cancelling is allowed until the slot starts. After that, the admin marks `ATTENDED` or `NO_SHOW`.
- Closing a day or slot that has bookings requires `force=true`. Affected bookings are cancelled and a `SlotBookingCancelled` event is published (the email is sent in 007).
- At slot end + 2 h, a job marks remaining `BOOKED` bookings as `NO_SHOW` and publishes `SlotNoShow` (used by reservations in 005).
- Other modules use the public API `SlotBookingService` (`findOrCreateBooking(familyId, slotId, purpose)`, `getBooking`, …). Collect and return purposes merge into one booking (BR-14).

## Endpoints
| Method & path | Role | Purpose |
|---|---|---|
| GET `/api/v1/slots/availability?from=&to=` | public | Days → slots `{id, date, start, end, remaining, status, bookable}` (no family data) |
| GET `/api/v1/me/slot-bookings?upcoming=true` | PARENT | My bookings with linked reservations/loans (filled in by 005/006) |
| POST `/api/v1/me/slot-bookings` | PARENT | `{slotId, purpose}`. Idempotent per family and slot (merges purposes) |
| DELETE `/api/v1/me/slot-bookings/{id}` | PARENT | Cancel. Linked holds keep their `hold_expires_at` (005) |
| GET/PUT `/api/v1/admin/slot-settings` | ADMIN | View / change schedule settings (applies to future non-booked slots) |
| GET `/api/v1/admin/slots?date=` | ADMIN | Slots for a day with bookings (family name, purposes, item counts) |
| POST `/api/v1/admin/closures` | ADMIN | `{from, to, slotTime?, reason, force}` |
| POST `/api/v1/admin/slot-bookings/{id}/attended` · `/no-show` | ADMIN | Mark attendance |

## UI
- `SlotPicker` component (reused in 005/006): the next 14 days as horizontal date chips, two slot buttons per day showing "3 places left" or "Full", and disabled states with reasons such as "Closed – holiday".
- `/my/visits`: upcoming visits (date, time, what to bring or collect) with a cancel button.
- `/admin/slots`: a day view with date navigation, each slot's bookings, capacity bar, close-day dialog and settings page.
- All times are shown in Europe/London with friendly labels such as "Today 5:00 pm" or "Tomorrow 7:00 pm".

## Acceptance criteria
- AC1: With default settings, availability shows two slots per day for the next 14 days.
- AC2: With 8 bookings on a slot, the 9th family gets 409 `SLOT_FULL`. 20 parallel booking attempts on a slot with 1 place produce exactly 1 success (concurrency test).
- AC3: Booking at 16:30 for the 17:00 slot gets 409 `SLOT_CUTOFF_PASSED`. Tests use a fixed `Clock`.
- AC4: Closing Christmas Day hides and disables it. Closing a day that has bookings without `force` returns 409 listing the affected families. With `force`, the bookings are cancelled and an event is published.
- AC5: Changing capacity to 10 affects only future slots. Existing bookings are untouched.
- AC6: A family with status `PENDING_APPROVAL` can't book (403 `FAMILY_NOT_ACTIVE`).
- AC7: The no-show job marks unattended bookings correctly (time-frozen test).
- AC8: Daylight-saving changes (last Sunday of March and October) don't shift the 17:00 slot (test).

## Open questions
- Q1: Is capacity per family (default) or per number of books?
- Q2: Can families return books at *any* open slot without booking? Default: a booking is required so you know who is coming. Walk-in returns can be checked in by the admin anyway (006).
