# Database Layer Rules

## Repositories

- One repository struct per domain aggregate (e.g., `UserRepository`, `PostRepository`).
- Repositories hold a `PgPool` and implement the domain's port traits.
- Return domain types (via `TryFrom<Row>`), never raw database rows.
- Use `fetch_optional()` for lookups. "Not found" is `Ok(None)`, not an error.
- Map database-specific errors (unique constraint violations) to domain errors in the repository.

## Queries

- Use `sqlx::query_as!` for compile-time checked queries.
- Use parameterized queries (`$1`, `$2`) exclusively. Never interpolate values into SQL strings.
- Use `RETURNING` clause for INSERT/UPDATE to get the result in one round-trip.
- Use `COALESCE` for partial updates, but document the null-clearing limitation.

## Migrations

- Migrations live in `migrations/` with timestamp prefixes: `YYYYMMDDHHMMSS_description.sql`.
- Use `gen_random_uuid()` for UUID primary keys (built-in, no extension needed).
- Use `TIMESTAMPTZ` (not `TIMESTAMP`) for all time columns.
- Create indexes on columns used in WHERE clauses and JOIN conditions.
- Use database triggers for `updated_at` columns.
- Run `cargo sqlx prepare` after changing queries or migrations. Commit the `.sqlx/` directory.

## Transactions

- Wrap multi-step writes in explicit transactions.
- If any step fails (or the function returns early via `?`), the transaction auto-rolls-back on drop.
- For request-scoped transactions, consider `axum-sqlx-tx`.

## Row Types

- Database row structs derive `sqlx::FromRow` and live in the infra layer.
- They are separate from domain entities and API response types.
- Implement `TryFrom<Row> for DomainEntity` with proper error handling for corrupt data.
