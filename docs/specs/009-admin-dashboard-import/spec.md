# Spec 009 – Admin Dashboard, Reports & Bulk Import

Status: Ready · Depends on: 006 (008 recommended) · Module: `admin`

## Goal
Give the 1–2 volunteers a single place to see "what do I need to do today", load the catalogue in bulk, print labels, and understand how the library is used.

## Features
1. **Today dashboard (`/admin`):**
   - Tiles: families booked per slot today, books to put aside (pick list count), expected returns, overdue loans, families pending approval, failed emails.
   - Each tile links to its screen.
   - "Next slot in 1 h 20 m" with a quick link to the desk.
2. **Settings (`/admin/settings`):** a single form for all ⚙ rules in the requirements (loan days, limits, hold hours, overdue threshold, auto-approve, admin email, terms/privacy versions). Validated, audited, and cached with evict-on-change.
3. **Bulk import (`/admin/import`):**
   - Upload a CSV with columns `isbn,title,authors,age_min,age_max,genres,copies,series_name,series_no`. Only `isbn` or `title` is required.
   - Dry-run preview: each row shows new title / add copies to existing / error, with ISBN enrichment from 002 (rate-limited to 1 request/s, run as a background job with progress).
   - Confirm to apply. The import is idempotent per `importId`. A downloadable error report lists failed rows.
4. **Labels (`/admin/labels`):** choose copies (e.g. "created today" or selected books) and generate a **PDF of barcode labels** (Code 128 + short title). Layout for A4 sheets of 21 or 24 labels (configurable), generated server-side with OpenPDF + ZXing.
5. **Reports (`/admin/reports`):**
   - Loans per month, active families, most-borrowed titles, titles never borrowed, borrowing by age band, slot utilisation.
   - Date range filter, CSV export. Charts use Recharts.
   - Queries run on read-only SQL views, with no personal data in exports except for the family list.
6. **Audit log viewer (`/admin/audit`):** filter by actor, entity and date.
7. **Outbox (`/admin/emails`):** failed and pending messages, with retry (from 007).
8. **Admin management:** list admins (Cognito group members). Adding an admin is documented as a runbook step (Cognito console/CLI). No UI in MVP.

## Endpoints (all `ADMIN`)
`GET /api/v1/admin/dashboard/today` · `GET/PUT /api/v1/admin/settings` · `POST /api/v1/admin/imports` (multipart, returns `importId`) ·
`GET /api/v1/admin/imports/{id}` (status, preview rows, progress) · `POST /api/v1/admin/imports/{id}/apply` ·
`GET /api/v1/admin/imports/{id}/errors.csv` · `POST /api/v1/admin/labels.pdf` · `GET /api/v1/admin/reports/{report}?from=&to=` (+ `.csv`) ·
`GET /api/v1/admin/audit`

## Acceptance criteria
- AC1: The dashboard shows correct counts for a seeded day, and each tile's number matches its detail screen (integration test on the seed data).
- AC2: Importing a 500-row CSV with 10 bad rows previews 490 valid rows and 10 errors with reasons. Applying creates the titles and copies. Re-applying the same import creates nothing new.
- AC3: The labels PDF for 30 copies prints on 2 A4 pages, and the barcodes scan correctly with the desk scanner (manual check plus a ZXing decode test).
- AC4: Changing loan days to 21 affects new loans only.
- AC5: Reports return in under 1 s on the 10k-book / 50k-loan seed. The CSV matches the on-screen data.
- AC6: All settings changes appear in the audit log with old and new values.

## Open questions
- Q1: Label sheet type or brand (e.g. Avery L7160 with 21 per sheet)?
- Q2: Do you have an existing book list (spreadsheet) to import? Share the columns so the template can match.
