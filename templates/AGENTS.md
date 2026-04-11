# Project Rules

This is a Rust web service built with Axum, SQLx, and Tokio.

## Architecture

- Three layers: `src/api/` (HTTP), `src/domain/` (business logic), `src/infra/` (database, external APIs).
- Dependencies point inward: api depends on domain, infra depends on domain. Domain depends on nothing external.
- Domain types never import axum, sqlx, reqwest, or any framework crate.
- Handlers are thin: extract request data, call a service method, return a response. No SQL, no business logic.
- Shared state lives in `AppState` and is extracted via `State<AppState>` or `FromRef`.

## Code Style

- Use `thiserror` for domain error enums. Use `anyhow` only for the catch-all `AppError::Internal` variant.
- Every handler returns `Result<T, AppError>`. Never `unwrap()` or `expect()` in request-handling code.
- Use newtypes for domain identifiers and validated values (Email, UserId, etc.). Keep inner fields private.
- Separate types at boundaries: request DTOs, domain entities, database rows, response DTOs. Only share types when they are structurally identical and have identical invariants.
- No emdashes in documentation or comments. Write in clear, direct sentences.

## Database

- All SQL goes through repository or query structs, never in handlers.
- Use `sqlx::query_as!` for compile-time checked queries.
- Use `TryFrom<Row>` (not `From`) when converting database rows to domain types.
- Use parameterized queries exclusively. No string interpolation in SQL.
- Run `cargo sqlx prepare --check` in CI.

## Testing

- Unit tests for domain logic with in-memory or mock repositories.
- Integration tests using `tower::ServiceExt::oneshot` against the real router.
- Database tests using `#[sqlx::test]` for isolation.
- Test error paths, not just happy paths.

## Security

- Validate input at the API boundary with `ValidatedJson` / `ValidatedQuery`.
- Hash passwords with Argon2id via `spawn_blocking`.
- Never log secrets. Use `secrecy` crate for sensitive config values.
- CORS, rate limiting, body size limits, and request timeouts are required in production.

## Verification

A change is done when:
```
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo sqlx prepare --check
```
All four pass. Do not skip any.

## Nested Instructions

Layer-specific rules are in:
- `src/http/AGENTS.md` (handlers, middleware, extractors, DTOs)
- `src/domain/AGENTS.md` (models, services, ports, errors)
- `src/infra/db/AGENTS.md` (repositories, queries, migrations)
