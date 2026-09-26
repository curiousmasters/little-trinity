# Spec 006 – Loans: Check-out & Check-in

Status: Ready · Depends on: 005 · Module: `loans`

## Goal
During a slot, the admin can hand over held books and take back returned ones in seconds, from a phone, by scanning barcodes. Parents always know what's out and when it's due.

## Domain
- **Loan:**
  - `copyId`, `bookId`, `childId`, `familyId`, `reservationId?`, `checkedOutAt`, `dueDate`.
  - `renewals` (0), `returnedAt?`, `returnBookingId?`, `status` (`ACTIVE`|`OVERDUE`|`RETURNED`|`LOST`), `checkInCondition?`.
- **Check-out** (admin, at a slot, for a family), in one transaction:
  - Allowed only for copies in a `READY` reservation of that family.
  - An "over-the-counter" loan of an `AVAILABLE` copy is also allowed. It needs the admin to pick a child and passes the same eligibility checks.
  - Effects: reservation → `COLLECTED`; copy → `ON_LOAN`; `dueDate = today + loanDays` (BR-08); slot booking → `ATTENDED`.
  - Publishes `LoanStarted`.
- **Check-in** (admin, by barcode, at any slot, with or without a return booking):
  - Loan → `RETURNED`, `returnedAt` set, condition recorded (`OK`|`DAMAGED`).
  - Copy → `AVAILABLE` (or `DAMAGED`, removed from circulation).
  - Publishes `LoanReturned` and `CopyBecameAvailable` (008 uses this to offer the copy to the waitlist).
- **Return booking:** parent books a slot with purpose `RETURN` and ticks which loans they're bringing. This gives the admin an "expected returns" list. It's optional for check-in to work.
- **Lost:** admin marks a loan `LOST`, which sets the copy to `LOST`. No fines (BR-12).

## Endpoints
| Method & path | Role | Purpose |
|---|---|---|
| GET `/api/v1/me/loans?status=` | PARENT | Loans with book, child, due date, days left, renewable? |
| POST `/api/v1/me/returns` | PARENT | `{slotId, loanIds[]}` → return booking |
| GET `/api/v1/admin/slots/{slotId}/desk` | ADMIN | **Desk view:** per family, the READY reservations to hand over and the expected returns |
| POST `/api/v1/admin/checkout` | ADMIN | `{familyId, slotBookingId?, items:[{barcode, childId?}]}` → per-item result |
| POST `/api/v1/admin/checkin` | ADMIN | `{barcode, condition, note?}` → loan summary and "next: held for X" hint (after 008) |
| POST `/api/v1/admin/loans/{id}/lost` | ADMIN | Mark lost |
| GET `/api/v1/admin/loans?status=&familyId=&dueBefore=` | ADMIN | Search loans |

Scanning a barcode that doesn't match the family's READY reservations returns a clear per-item error (`COPY_NOT_RESERVED_FOR_FAMILY`) so the admin can fix it on the spot.

## UI
- **`/admin/desk` (the most important admin screen, optimised for phone):**
  - Choose today's slot. The families booked are shown as cards: "Kumar family – collect 3, return 2".
  - Tap a family, then scan each book (camera or keyboard wedge). Items tick green or show a red error inline. A "Complete handover" button finishes.
  - A separate "Returns" scan mode: scan continuously, each scan checks the book in and shows toast feedback. Condition defaults to OK, with a quick "Damaged" toggle.
- **`/my/loans`:** cards per child showing "Due in 5 days (Sun 12 Oct)" or an overdue badge, and a "Book a return visit" flow (select loans, then SlotPicker).
- **Household view `/my`:** summary with "Next visit", "Books at home", "Ready to collect".

## Acceptance criteria
- AC1: At a slot, scanning the 2 READY copies for the Kumar family creates 2 ACTIVE loans due in 14 days, and sets both reservations to COLLECTED and the booking to ATTENDED.
- AC2: Scanning a copy reserved for another family is rejected with a clear message, and nothing changes.
- AC3: Check-in of an on-loan copy sets the loan to RETURNED and the copy to AVAILABLE. Checking in the same barcode again returns 409 `COPY_NOT_ON_LOAN`, and the idempotent UI treats it as "already returned".
- AC4: Check-in marked Damaged puts the copy in `DAMAGED`, and it doesn't count as available in the catalogue.
- AC5: A parent's return booking lists the selected loans in the admin desk view under "expected returns".
- AC6: An over-the-counter loan of an AVAILABLE copy respects child and family limits.
- AC7: The desk flow works on a 375 px wide screen and is fully keyboard-operable (Playwright mobile viewport test).
- AC8: All checkout and check-in actions are recorded in `audit_log` with the admin as actor.

## Open questions
- Q1: Should due dates skip closure days (move to the next open day)? Default: yes.
