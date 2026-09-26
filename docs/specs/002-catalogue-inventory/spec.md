# Spec 002 – Catalogue & Inventory

Status: Ready · Depends on: 001 · Module: `catalogue`

## Goal
Admins can quickly add books (scan or type an ISBN, and the details fill themselves in) and record physical copies with barcodes. Everyone can search and browse the catalogue in a child-friendly way.

## Domain
- `Book` (a title):
  - `isbn13` (optional; some kids' books have none), `title`, `subtitle`, `authors[]`, `seriesName`, `seriesNumber`, `publisher`, `publishedYear`.
  - `description`, `ageMin`, `ageMax` (0–16), `language` (default `en`), `genres[]`, `coverKey`.
- `BookCopy`:
  - `barcode` (format `LT-000001`, generated sequentially and unique), `status`, `conditionNote`, `acquiredOn`.
  - Status is `AVAILABLE` | `ON_HOLD` | `ON_LOAN` | `DAMAGED` | `LOST` | `WITHDRAWN`.
  - Status changes caused by holds and loans happen **only** through the catalogue module's API (`CopyInventory` port), called by reservations and loans. Admins can manually set `DAMAGED`, `LOST`, `WITHDRAWN` or `AVAILABLE`.
- Availability shown to users is `AVAILABLE_NOW` (at least one copy AVAILABLE), `ALL_OUT` (copies exist but none available → waitlist, 008) or `NO_COPIES`.
- Genre list is seeded with: Picture books, Early readers, Adventure, Fantasy, Animals, Funny, Mystery, Non-fiction, Science, History, Poetry, Graphic novels. Admins can manage it.

## ISBN lookup
- `GET /api/v1/admin/isbn/{isbn}` → a draft book from **Open Library** (`/isbn/{isbn}.json` + works + authors), falling back to **Google Books** (`volumes?q=isbn:`).
- 3 s timeout per provider. The result is cached for 24 h in memory (Caffeine).
- Returns 404 `ISBN_NOT_FOUND` so the admin can type the details manually.
- It **never** saves automatically. The admin reviews and edits first.
- Cover: download the provider's cover image, validate it (JPEG/PNG/WebP, ≤ 2 MB), resize to 400 px wide, and store it through the `MediaStorage` port. There's a local filesystem implementation for `local`/`test` and an S3 implementation for `dev`/`prod`. Covers are served from `/media/covers/{key}` (CloudFront → S3 in AWS).

## Search
- `GET /api/v1/books?q=&ageFrom=&ageTo=&genre=&availableOnly=&series=&page=&size=&sort=`
  - `q` uses full-text search plus trigram similarity (ADR 0004). Sort is by relevance (when `q` is given), `title` or `newest`.
  - Response items: `{id, title, authors, seriesName, seriesNumber, ageMin, ageMax, genres, coverUrl, availability, availableCopies}`.
- `GET /api/v1/books/{id}` → full detail plus the availability summary. Copy barcodes are not shown to the public.
- `GET /api/v1/genres`.
- Performance: seed 10k synthetic books in a performance test profile. p95 search is under 200 ms at the DB level.

## Admin endpoints
| Method & path | Purpose |
|---|---|
| POST `/api/v1/admin/books` | Create a book (plus `initialCopies` count, default 1) |
| PUT `/api/v1/admin/books/{id}` | Edit |
| POST `/api/v1/admin/books/{id}/cover` | Upload a cover (multipart) |
| POST `/api/v1/admin/books/{id}/copies` | Add N copies → returns barcodes |
| GET `/api/v1/admin/copies/{barcode}` | Look up a copy by barcode (used by the scanner) |
| PATCH `/api/v1/admin/copies/{id}` | Change status (manual statuses only) or condition note |
| DELETE `/api/v1/admin/books/{id}` | Only if no copy has ever been loaned. Otherwise withdraw the copies |

Duplicate ISBN on create → 409 `DUPLICATE_ISBN`, including the existing book ID so the UI can offer "add a copy instead".

## UI
- `/books`: search box (debounced), filter chips (age bands 0–3, 4–6, 7–9, 10–12, 13+), genre, and "Available now" toggle. Responsive grid of `BookCard`s (cover, title, author, age badge, availability pill). Infinite scroll or pagination. The URL reflects filters so searches can be shared.
- `/books/:id`: cover, details, series ("Book 2 of …" linking to others in the series), availability, and a **Reserve** button. The button is disabled for visitors and says "Sign in to reserve"; it's wired up in 005.
- `/admin/books`: table with search, and an "Add book" dialog. The ISBN field supports keyboard-wedge scanners and a phone camera scan (html5-qrcode). It fetches the draft into an editable form, with a copies count.
- `/admin/books/:id`: edit the book, list copies with status, add copies, and print barcode labels (label printing in 009).

## Acceptance criteria
- AC1: The admin scans an ISBN and the form fills in with title, author and cover within 3 s. The admin edits and saves, and a copy with barcode `LT-000001` exists.
- AC2: An unknown ISBN shows a manual-entry form, not an error page.
- AC3: Adding a duplicate ISBN offers to add a copy to the existing title.
- AC4: Searching "gruffalo" and a misspelling such as "grufalo" both find "The Gruffalo".
- AC5: Filters combine (age 4–6 + Animals + Available now), and the URL reproduces the result.
- AC6: Visitors can browse and search without logging in. Admin endpoints return 403 for parents.
- AC7: Copy status can't be set to `ON_HOLD`/`ON_LOAN` via the admin PATCH (400).
- AC8: ISBN lookup tests use WireMock stubs, with no real network access in tests.
- AC9: The 10k-book performance test meets the p95 target.

## Open questions
- Q1: Barcode prefix `LT-`? Do you already have labels?
- Q2: Store age as a range (default) or as fixed bands?
