# Error Handling

If you've worked on a web service for any length of time, you know that error handling is really about two completely different audiences. Your API consumers need clear feedback when something goes wrong: a proper HTTP status code and a message that actually helps them fix the problem. Your team, on the other hand, needs the full picture: stack traces, error chains, and enough context to track down the bug. The trick is serving both without leaking internal details to the outside world.

Axum makes an interesting design choice here. It requires that all handlers are infallible at the framework level, which means they always have to return a valid HTTP response, even when things go sideways. In practice, you do this by returning `Result<T, E>` where `E` implements `IntoResponse`. When your handler returns `Err(e)`, Axum calls `e.into_response()` to produce the error response. This gives us full control over how errors look to the client, which is exactly what we want.

## The AppError pattern

The idea is straightforward: we create one central error enum that covers all the error cases our API can produce. Every handler returns `Result<T, AppError>`, and the `IntoResponse` implementation on `AppError` takes care of mapping each variant to the right HTTP status code and response body. Let's look at what this looks like in practice.

```rust
use std::collections::HashMap;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("resource not found")]
    NotFound,

    #[error("{0}")]
    Validation(String),

    #[error("validation failed")]
    ValidationFields(HashMap<String, Vec<String>>),

    #[error("authentication required")]
    Unauthorized,

    #[error("insufficient permissions")]
    Forbidden,

    #[error("{0}")]
    Conflict(String),

    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, error_type, message) = match &self {
            AppError::NotFound => (
                StatusCode::NOT_FOUND,
                "not_found",
                self.to_string(),
            ),
            AppError::Validation(msg) => (
                StatusCode::BAD_REQUEST,
                "validation_error",
                msg.clone(),
            ),
            AppError::ValidationFields(ref fields) => {
                let body = serde_json::json!({
                    "error": {
                        "type": "validation_error",
                        "message": "request validation failed",
                        "fields": fields,
                    }
                });
                return (StatusCode::BAD_REQUEST, Json(body)).into_response();
            }
            AppError::Unauthorized => (
                StatusCode::UNAUTHORIZED,
                "unauthorized",
                self.to_string(),
            ),
            AppError::Forbidden => (
                StatusCode::FORBIDDEN,
                "forbidden",
                self.to_string(),
            ),
            AppError::Conflict(msg) => (
                StatusCode::CONFLICT,
                "conflict",
                msg.clone(),
            ),
            AppError::Internal(err) => {
                // Log the full error chain for debugging. This is the only
                // place where the real error details are visible.
                tracing::error!(error = ?err, "internal server error");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal_error",
                    "an internal error occurred".to_string(),
                )
            }
        };

        let body = serde_json::json!({
            "error": {
                "type": error_type,
                "message": message,
            }
        });

        (status, Json(body)).into_response()
    }
}
```

The most important piece here is the `Internal` variant. It wraps `anyhow::Error`, which can hold any error type along with a chain of context messages. When an internal error happens, we log the full details (the error chain and, if available, a backtrace) but return only a generic message to the client. This is how we prevent information leakage. You really don't want your database schema details or internal file paths showing up in API responses. I've seen that happen in production, and it's not a fun conversation with the security team.

## A type alias for convenience

This might seem like a small thing, but it makes a real difference for readability. We define a type alias to keep our handler signatures clean:

```rust
pub type AppResult<T> = Result<T, AppError>;
```

Now our handlers look like this:

```rust
async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> AppResult<Json<UserResponse>> {
    let user = state.user_service
        .get_by_id(UserId::from_uuid(id))
        .await?
        .ok_or(AppError::NotFound)?;

    Ok(Json(user.into()))
}
```

Notice how the `?` operator just works here. That's because of the `From` implementations on `AppError`, which we'll look at next.

## The error mapping chain

In our layered architecture, errors start at the bottom (the infrastructure layer), pass through the domain layer, and eventually reach the API layer where they become HTTP responses. Each layer has its own error types, and we use `From` implementations to convert between them.

Here's what the chain looks like for a "create user" operation:

```
sqlx::Error
  → CreateUserError (domain)
      → AppError (api)
          → HTTP Response
```

The infrastructure layer catches the database error and converts it into something the domain understands:

```rust
// In the repository implementation
.map_err(|e| match e {
    sqlx::Error::Database(ref db_err) if db_err.is_unique_violation() => {
        CreateUserError::Duplicate { email: req.email.clone() }
    }
    other => CreateUserError::Unknown(other.into()),
})
```

Then the API layer converts that domain error into an `AppError`:

```rust
impl From<CreateUserError> for AppError {
    fn from(err: CreateUserError) -> Self {
        match err {
            CreateUserError::Duplicate { email } => {
                AppError::Conflict(
                    format!("a user with email {} already exists", email.as_str())
                )
            }
            CreateUserError::Unknown(inner) => AppError::Internal(inner),
        }
    }
}
```

With these `From` implementations in place, our handler can just use `?` and let the compiler figure out the conversions. No manual mapping, no boilerplate:

```rust
async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    let user = state.user_service.register(payload).await?;  // ? handles the full chain
    Ok((StatusCode::CREATED, Json(user.into())))
}
```

## thiserror vs. anyhow

You might wonder why we're using two different error crates. They actually serve complementary purposes, and in my experience, a well-structured application uses both.

**thiserror** is for errors that your code needs to match on. It generates `Display` and `Error` trait implementations from your enum definitions. We use it for domain errors (`CreateUserError`, `AuthError`), for the `AppError` enum, and for any error type where callers need to distinguish between variants and react differently.

**anyhow** is for errors that your code just needs to propagate. It gives you a single `anyhow::Error` type that can hold any error, along with context messages that form a chain explaining what went wrong. We use it for the catch-all `Internal` variant of `AppError`, and in infrastructure code where we want to add context without defining a new error variant for every possible failure mode.

The rule of thumb I follow: use `thiserror` at boundaries where callers need to make decisions based on the error variant, and use `anyhow` for the "everything else" case where the error just needs to be logged and reported.

## Adding context with anyhow

The `anyhow::Context` trait lets you attach human-readable context to errors as they propagate up the call stack. This is one of those things that seems like extra work when you're writing the code, but it's incredibly helpful when you're debugging. It tells you not just what failed, but why the code was trying to do that thing in the first place.

```rust
use anyhow::Context;

pub async fn create_pool(database_url: &str) -> anyhow::Result<PgPool> {
    PgPoolOptions::new()
        .max_connections(10)
        .acquire_timeout(Duration::from_secs(3))
        .connect(database_url)
        .await
        .context("failed to connect to the database")
}
```

When this error reaches the `Internal` variant of `AppError` and gets logged, the output will include both the original error ("connection refused") and the context ("failed to connect to the database"). That's the difference between staring at a cryptic error message and immediately knowing where to look.

## Rules of thumb

**Never call `unwrap()` or `expect()` in code that handles HTTP requests.** A panic in a handler doesn't just affect that one request. If a panic happens while holding a mutex or other shared resource, it can poison the resource and cascade to other requests. I've seen enough bugs from careless `unwrap()` calls to be pretty firm on this one. Always return errors through the `Result` type.

**Never expose internal error messages to clients.** Database errors, file paths, stack traces, and internal service names are all information that an attacker can use. Log them at the `error` level for your team, and return a generic "internal error" message to the client.

**Use `thiserror` for structured errors and `anyhow` for catch-all propagation.** Don't use `anyhow` as your handler's return type directly, because you lose the ability to map different errors to different HTTP status codes. Use it inside the `AppError::Internal` variant instead.

**Use `#[from]` for automatic conversions.** The `thiserror` `#[from]` attribute generates `From` implementations that let the `?` operator convert errors automatically. This keeps your handler code clean and lets you focus on the happy path.

**Add context with `.context()` liberally in infrastructure code.** The small cost of writing a context string pays off enormously when you're debugging something in production at 2 AM and the error log says "failed to persist user registration" instead of just "connection reset by peer". Trust me on this one.
