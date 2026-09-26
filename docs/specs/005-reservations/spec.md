# Spec 005 – Reservations (reserve anytime, collect at a slot)

Status: Ready · Depends on: 002, 004 · Module: `reservations`

## Goal
A parent can reserve a book for a child at any hour. The system immediately holds a physical copy and asks the parent to choose a collection slot. The admin sees exactly which copies to put aside for each slot.

## Reservation lifecycle
```
            ┌─────────── (no copy free: see 008) ───────► WAITLISTED
 request ──►│
            └─ copy allocated ─► HELD ──(slot booked)──► READY ──(checked out, 006)──► COLLECTED
                                  │                        │
                                  │ no slot within 48h     │ no-show → hold until end of next open day
                                  ▼                        ▼
                               EXPIRED                  EXPIRED
 Parent can cancel HELD/READY/WAITLISTED → CANCELLED. Admin can cancel any open reservation (with a reason).
```
- **Allocation:** in one transaction, pick one `AVAILABLE` copy of the title (`FOR UPDATE SKIP LOCKED`) → catalogue port sets it to `ON_HOLD`. When a reservation ends without collection, the copy goes back to `AVAILABLE` (or to the next waitlisted family, 008).
- **Eligibility checks** (each failure has a stable error code):
  - `FAMILY_NOT_ACTIVE`: the family isn't active.
  - `CHILD_NOT_IN_FAMILY`: the child doesn't belong to this family.
  - `CHILD_LIMIT_REACHED` (BR-06) and `FAMILY_LIMIT_REACHED` (BR-07).
  - `ALREADY_RESERVED_OR_ON_LOAN`: this child already has this title.
  - `FAMILY_HAS_OVERDUE` (BR-11, active once 008 is in place; interface stubbed now).
- **Hold expiry:** `holdExpiresAt = heldAt + 48h` while `HELD`. When a slot is booked the reservation becomes `READY`, and `holdExpiresAt` becomes the end of the next open day after the slot. The job runs every 15 minutes and publishes `HoldExpired`.
- **Booking a collection slot:** `POST /me/reservations/{id}/collection` with `{slotId}` calls `SlotBookingService.findOrCreateBooking(family, slot, COLLECT)`. Several reservations can share a booking (BR-14). Changing the slot is allowed until the slot starts.
- **Basket flow (UX):** reserving several books in one go uses `POST /me/reservations/batch`. It's all-or-nothing, with per-item errors, so the parent picks one slot for everything.
- Events: `ReservationConfirmed` (HELD), `ReservationReady` (slot chosen), `HoldExpired`, `ReservationCancelled`.
- Listens to `SlotBookingCancelled` (reservations go back to `HELD` with a fresh 48 h window) and `SlotNoShow`.

## Endpoints
| Method & path | Role | Purpose |
|---|---|---|
| POST `/api/v1/me/reservations` | PARENT | `{childId, bookId}` + `Idempotency-Key` → 201 `{id, status, copy?, holdExpiresAt}` |
| POST `/api/v1/me/reservations/batch` | PARENT | `{items:[{childId, bookId}], slotId?}` |
| GET `/api/v1/me/reservations?status=` | PARENT | Mine, with book summary, child, slot |
| POST `/api/v1/me/reservations/{id}/collection` | PARENT | Set/change the collection slot |
| DELETE `/api/v1/me/reservations/{id}` | PARENT | Cancel |
| GET `/api/v1/admin/reservations?status=&date=` | ADMIN | List/filter |
| GET `/api/v1/admin/slots/{slotId}/pick-list` | ADMIN | Copies to prepare: barcode, title, child nickname, family, grouped by family |
| POST `/api/v1/admin/reservations/{id}/cancel` | ADMIN | `{reason}` |

## UI
- **Book detail page:** "Reserve for…" with a child selector. It shows eligibility problems before submitting (e.g. "Aarav already has 3 books").
- **Basket:** "Add to basket" on book cards, a basket drawer with a child per item, then a SlotPicker (from 004), then confirm.
- **`/my/reservations`:** tabs Waiting for slot / Ready to collect / Past. Each card shows cover, child, status, "Collect on Tue 5:00 pm" or "Choose a collection time (by Thu 18:40)", and change-slot or cancel actions.
- **`/admin/slots` day view:** a **Pick list** per slot (printable) so the admin can put books aside before 17:00.
- Friendly error messages mapped from the error `code`s.

## Acceptance criteria
- AC1: With 1 available copy, reserving puts the reservation in `HELD` and the copy in `ON_HOLD`. The book then shows "All out" to others.
- AC2: 10 parallel reservation requests for a title with 1 copy produce exactly 1 `HELD` reservation. The rest are `WAITLISTED` (once 008 exists), or get 409 `NO_COPY_AVAILABLE` before 008 (concurrency test).
- AC3: Choosing a slot moves the reservation to `READY`, and the pick list for that slot includes the copy barcode.
- AC4: A reservation with no slot chosen for 48 h becomes `EXPIRED` and the copy becomes `AVAILABLE` (frozen-clock test).
- AC5: A no-show keeps `READY` until the end of the next open day, then expires.
- AC6: Limits BR-06 and BR-07 are enforced, and cancelled or expired reservations don't count towards them.
- AC7: Double-submitting with the same `Idempotency-Key` returns the same reservation and doesn't create a second one.
- AC8: A batch with one ineligible item creates nothing and returns per-item errors.
- AC9: A parent can't view or cancel another family's reservation (404).
- AC10: Playwright journey: sign in → search → add 2 books to the basket → choose tomorrow at 17:00 → see them under "Ready to collect".

## Open questions
- Q1: Should reserving require choosing a slot immediately (simpler), or allow "choose later" within 48 h (default, more flexible)?
