# Skill: Review PR

Review a pull request for architecture, correctness, security, and operational readiness.

## Architecture

- [ ] Dependencies flow inward (api -> domain, infra -> domain). Domain has no framework imports.
- [ ] Handlers are thin (extract, call service, respond). No SQL or business logic in handlers.
- [ ] New types are in the right layer (DTOs in api, entities in domain, rows in infra).
- [ ] No database row types leaked into API responses.
- [ ] No framework types (axum extractors, sqlx types) in domain code.

## Correctness

- [ ] No `unwrap()` or `expect()` in request-handling code paths.
- [ ] Error types are appropriate: domain errors for business violations, AppError for HTTP mapping.
- [ ] `?` propagation works through the full chain (infra error -> domain error -> AppError).
- [ ] Newtypes are used for validated domain values (not raw String/Uuid).
- [ ] Partial updates use PATCH, not PUT.

## Database

- [ ] All queries use parameterized placeholders (no string interpolation).
- [ ] New queries use `query_as!` for compile-time checking.
- [ ] Row-to-domain conversions use `TryFrom`, not `From`.
- [ ] Migrations are backward-compatible or use expand/contract.
- [ ] `.sqlx/` metadata is updated if queries changed.

## Security

- [ ] Input validation at the API boundary (ValidatedJson/ValidatedQuery).
- [ ] Sensitive data excluded from response DTOs (password_hash, internal IDs).
- [ ] Authentication checked for protected endpoints (AuthUser or route_layer).
- [ ] No secrets in logs (check tracing::instrument fields).
- [ ] Rate limiting considered for public endpoints.

## Testing

- [ ] Happy path integration test exists for new endpoints.
- [ ] Error path test exists for the primary failure case.
- [ ] Domain logic has unit tests if it contains non-trivial business rules.

## Operations

- [ ] New handlers have `#[tracing::instrument]` with appropriate skip/fields.
- [ ] No blocking calls on the async runtime (spawn_blocking for CPU work).
- [ ] Outbound HTTP calls have timeouts configured.
- [ ] Background tasks handle shutdown via CancellationToken.
