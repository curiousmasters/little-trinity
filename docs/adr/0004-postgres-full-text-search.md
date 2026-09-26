# ADR 0004 – PostgreSQL full-text search for the catalogue

- Status: Accepted · Date: 2026-09-26

## Context
The catalogue will grow to about 10k titles. Searches are by title, author and series, with filters for age, genre and availability.

## Decision
Use a `tsvector` column (weighted: title A, series B, authors B, description C) with a GIN index, plus `pg_trgm` for typo-tolerant matching on title and author. Filters are plain indexed columns.

## Consequences
- No extra infrastructure. At 10k rows, queries stay well under 50 ms.
- If search needs grow (synonyms, facets at scale, "kids-friendly" fuzzy search), revisit OpenSearch. The search sits behind a `CatalogueSearch` port, so it can be replaced.
