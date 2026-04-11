# Feature Checklist

Use this when implementing a new feature end-to-end.

## Planning

- [ ] Identify which domain entities are involved
- [ ] Identify which layers need changes (domain, infra, api)
- [ ] Check if existing domain types, services, or repositories can be reused
- [ ] Plan the migration (if any) for backward compatibility

## Domain Layer

- [ ] Domain entity or value object defined with private fields
- [ ] Constructor validates invariants and returns Result
- [ ] Domain error enum covers the failure cases
- [ ] Service method orchestrates the business logic

## Infrastructure Layer

- [ ] Migration created and applied
- [ ] Repository method implemented with compile-time checked queries
- [ ] Row type defined with TryFrom conversion to domain entity
- [ ] Database errors mapped to domain errors

## API Layer

- [ ] Request DTO with validation attributes
- [ ] Response DTO excluding sensitive fields
- [ ] Handler: extract, parse, call service, respond
- [ ] Route registered with correct HTTP method
- [ ] Auth applied if endpoint is protected

## Testing

- [ ] Unit test for domain logic (if non-trivial)
- [ ] Integration test for happy path (oneshot)
- [ ] Integration test for primary error case
- [ ] Database test if query logic is complex

## Operations

- [ ] Handler instrumented with tracing
- [ ] Structured log events for significant business actions
- [ ] OpenAPI annotation if using utoipa

## Verification

- [ ] `cargo fmt --check` passes
- [ ] `cargo clippy --workspace --all-targets -- -D warnings` passes
- [ ] `cargo test --workspace` passes
- [ ] `cargo sqlx prepare --check` passes
