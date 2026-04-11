# HTTP Layer Rules

## Handlers

- A handler should be 5-20 lines: extract, call service, respond.
- Parse raw input into domain types inside the handler (e.g., `UserName::parse(&payload.name)`).
- Return `AppResult<T>` (type alias for `Result<T, AppError>`).
- Use `StatusCode::CREATED` for POST, `StatusCode::NO_CONTENT` for DELETE.
- Use `#[debug_handler]` during development for clearer compile errors.

## Extractors

- Use `ValidatedJson<T>` (not bare `Json<T>`) for request bodies that need validation.
- Use `ValidatedQuery<T>` for query parameters that need validation.
- Use `AuthUser` extractor for authenticated endpoints.
- Use `MaybeAuthUser` when authentication is optional (distinguishes "no header" from "invalid header").
- Body-consuming extractors (`Json`, `ValidatedJson`) must be last in the handler signature.

## Middleware

- `route_layer()` for authentication (only applies to matched routes).
- `.layer()` for global concerns (tracing, compression, timeouts).
- Order matters: Set request ID -> TraceLayer -> Propagate request ID -> Timeout -> Body limit -> CORS.

## DTOs

- Request DTOs derive `Deserialize` and `Validate`.
- Response DTOs derive `Serialize` (and `ToSchema` if using utoipa).
- Response DTOs must not include sensitive fields (password_hash, internal IDs, etc.).
- Use `From<DomainEntity>` to convert domain types to response DTOs.

## Routes

- Use PATCH (not PUT) for partial updates with optional fields.
- Nest routes: `/api/v1/users`, `/api/v1/posts`.
- Health endpoints live outside the versioned API: `/health`, `/health/ready`.
