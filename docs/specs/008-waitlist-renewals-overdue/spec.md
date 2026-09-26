# Spec 008 – Waitlist, Renewals & Overdue

Status: Ready · Depends on: 006, 007 · Modules: `reservations`, `loans`

## Goal
Popular books are shared fairly, families can keep a book longer when nobody is waiting, and overdue books are chased gently and automatically.

## Waitlist
- Reserving a title with no available copy creates a `WAITLISTED` reservation. Eligibility rules still apply, and waitlisted reservations **count** towards the child and family limits.
  - *Decision point:* default is to count them. Change to not count if you prefer.
- The queue is per title, FIFO by `createdAt`. The parent sees their position: "You are 2nd in the queue".
- On `CopyBecameAvailable` (check-in, hold expiry, cancellation), in one transaction:
  1. Find the oldest `WAITLISTED` reservation for the title whose family is still eligible (active, not blocked for overdue). Skip ineligible ones and leave them in place.
  2. Allocate the copy and set the reservation to `HELD` (48 h to choose a slot). Publish `WaitlistOfferMade`.
  3. If nobody is waiting, the copy becomes `AVAILABLE`.
- Check-in response includes "Put aside for: Kumar family (Aarav)" so the admin keeps the book on the hold shelf instead of the main shelf.
- Parents can leave the queue (cancel).
- Limit: a title's queue has at most 20 entries (`WAITLIST_FULL`).

## Renewals (BR-09)
- `POST /api/v1/me/loans/{id}/renew` → `dueDate += loanDays`, `renewals += 1`.
- Rejected with:
  - `RENEWAL_LIMIT_REACHED` if already renewed once.
  - `TITLE_HAS_WAITLIST` if anyone is waiting.
  - `LOAN_OVERDUE` if the loan is already overdue.
  - `FAMILY_NOT_ACTIVE` if the family isn't active.
- The admin can renew on the parent's behalf and can override the waitlist rule, with a reason recorded in audit.

## Overdue (BR-11)
- A daily job (06:00) sets `ACTIVE` loans with `dueDate < today` to `OVERDUE` and publishes `LoanOverdue(daysOverdue)`. It also publishes on day 7 and day 14 for reminder emails.
- A family is **blocked** from new reservations and renewals while any loan is more than 7 days overdue (`FAMILY_HAS_OVERDUE`). Returning the book lifts the block immediately.
- Admin list `/admin/overdue`: family, contact, books, days overdue, last reminder sent, "Mark lost" action.
- No fines.

## UI
- Book detail when all copies are out: "Join the waiting list (2 families ahead)".
- `/my/reservations` shows a "Waiting list" tab with queue position.
- `/my/loans` has a "Renew" button with the reason shown when it's disabled, and an overdue banner at the top of `/my`.
- Admin check-in toast: "Hold for Aarav (Kumar) – place on hold shelf".

## Acceptance criteria
- AC1: With 1 copy on loan and families A then B waitlisted, checking in the copy puts A's reservation in `HELD` with an email sent, while B stays at position 1.
- AC2: If A's hold expires, the copy passes to B automatically.
- AC3: A family blocked for overdue is skipped in the queue (keeping their place), and the next eligible family gets the copy.
- AC4: Renewal succeeds once, then fails with `RENEWAL_LIMIT_REACHED`, and fails with `TITLE_HAS_WAITLIST` if someone is waiting.
- AC5: A loan due yesterday is `OVERDUE` after the 06:00 job, and the family receives the day-1 email.
- AC6: A family with a loan 8 days overdue gets `FAMILY_HAS_OVERDUE` when reserving. After check-in they can reserve again.
- AC7: A concurrency test on check-in plus a parallel new reservation for the same title never double-allocates a copy.

## Open questions
- Q1: Should waitlisted reservations count towards limits? Default: yes.
- Q2: Overdue block threshold: 7 days (default)?
