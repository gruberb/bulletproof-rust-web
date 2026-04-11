# Domain Layer Rules

## Zero External Dependencies

This module must not import axum, sqlx, reqwest, tonic, tower, or any framework/infrastructure crate.
Allowed dependencies: serde, thiserror, anyhow, uuid, chrono, and standard library.

## Models

- Use newtypes for identifiers: `struct UserId(Uuid)`, `struct Email(String)`.
- Keep inner fields private. Provide a `parse()` constructor that returns `Result`.
- Implement `AsRef<str>` (or similar) for ergonomic read access.
- Once constructed, a value's validity is guaranteed. No re-validation needed downstream.
- Entities have private fields and getter methods. Mutation goes through methods that enforce business rules.

## Services

- Services are generic over repository traits: `UserService<R: UserRepository>`.
- Services contain business logic: validation, orchestration, authorization checks.
- Services return domain error types, never HTTP status codes or SQL errors.

## Ports (Traits)

- Repository traits live in `domain/ports/`.
- Traits use `-> impl Future<Output = ...> + Send` (not `#[async_trait]`) for internal traits.
- Traits are `Send + Sync + 'static` but NOT `Clone` (cloneability belongs on the holder).
- For dynamic dispatch needs, use `#[async_trait]` or `trait_variant::make`.

## Errors

- Each domain operation has its own error enum (e.g., `CreateUserError`, `AuthError`).
- Use `#[error("...")]` with human-readable messages.
- Include a catch-all `Unknown(#[from] anyhow::Error)` variant for unexpected failures.
- Domain errors describe business violations ("user already exists"), not infrastructure details ("unique constraint violation").
