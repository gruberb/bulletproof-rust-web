# Skill: Add Endpoint

Add a new API endpoint following the project's layered architecture.

## Steps

1. **Domain types.** Define or reuse the domain entity, newtypes, and error enum in `src/domain/`.
   - If the entity is new, create `src/domain/models/<entity>.rs` with newtypes and a constructor.
   - If new errors are needed, add variants to the relevant error enum or create a new one.

2. **Database migration.** If the endpoint requires schema changes:
   - Create a new migration file in `migrations/` with a timestamp prefix.
   - Use `gen_random_uuid()` for UUIDs, `TIMESTAMPTZ` for timestamps.
   - Run `cargo sqlx migrate run` and `cargo sqlx prepare`.

3. **Repository.** Add the query method to the repository in `src/infra/repositories/`.
   - Use `sqlx::query_as!` with compile-time checking.
   - Map database errors to domain errors.
   - Return domain types via `TryFrom`.

4. **Service.** If business logic is needed beyond a simple CRUD operation, add a method to the service in `src/domain/services/`.

5. **Request/Response DTOs.** Create request and response types in `src/api/dtos/`.
   - Request DTOs: `Deserialize` + `Validate`.
   - Response DTOs: `Serialize`, exclude sensitive fields.
   - Implement `From<DomainEntity>` for the response DTO.

6. **Handler.** Add the handler in `src/api/handlers/`.
   - Extract with `ValidatedJson` or `ValidatedQuery`.
   - Parse raw fields into domain types.
   - Call the service. Return the response.
   - Keep it under 20 lines.

7. **Route.** Register the handler in `src/api/routes.rs`.
   - Use the correct HTTP method (POST for create, PATCH for partial update, etc.).
   - Apply `route_layer` for auth if the endpoint is protected.

8. **Tests.** Write at minimum:
   - One integration test for the happy path using `oneshot`.
   - One test for the primary error case (validation failure, not found, etc.).

9. **Tracing.** Add `#[tracing::instrument(skip(state))]` to the handler.

10. **Verify.** Run the full verification gate:
    ```
    cargo fmt --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace && cargo sqlx prepare --check
    ```
