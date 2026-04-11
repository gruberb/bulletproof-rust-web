# Putting It All Together

If you've been reading the earlier chapters, you've absorbed a lot of concepts: layered architecture, domain modeling, keeping your layers properly separated. That's all great in theory. But what does it actually look like when you sit down and build something? That's what this chapter is for. We're going to walk through a single feature from start to finish, every file, every type, every connection between the layers. By the end you'll have seen one complete vertical slice through the entire application, from the HTTP request arriving at the handler, through the domain service and repository port, down to the database, and back up to the API response.

The feature we're building is user registration: a `POST /api/v1/users` endpoint that accepts a name, email, and password, validates them, hashes the password, persists the user, and returns the created user. Nothing exotic, but it touches every layer, which is exactly why I picked it.

## The files we will create

Before we dive in, here's where each piece lives in our project structure:

```
src/
├── domain/
│   ├── models/
│   │   └── user.rs          # Entity, value objects (Email, UserName, UserId)
│   ├── errors.rs             # CreateUserError
│   ├── ports/
│   │   └── user_repository.rs  # The trait (port)
│   └── services/
│       └── user_service.rs   # Business logic
│
├── infra/
│   └── repositories/
│       └── user_repo.rs      # PostgresUserRepo (implements the port)
│
├── api/
│   ├── dtos/
│   │   └── user_dto.rs       # CreateUserDto, UserResponse
│   ├── handlers/
│   │   └── users.rs          # The HTTP handler
│   ├── routes.rs              # Route registration
│   └── state.rs               # AppState
│
├── error.rs                   # AppError (HTTP error mapping)
└── main.rs                    # Wiring everything together
```

We'll build each piece starting from the inside (domain) and working our way outward. I find this order helps the most, because by the time you reach the handler, all the types it depends on already exist.

## Step 1: Domain models and value objects

These types live in `src/domain/models/user.rs`. The important thing is that they have zero dependencies on Axum, SQLx, or any framework crate. They enforce their own invariants through private fields and validated constructors, which means you can't accidentally create a bogus `Email` or an empty `UserName`.

```rust
use uuid::Uuid;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(Uuid);

impl UserId {
    pub fn new() -> Self { Self(Uuid::new_v4()) }
    pub fn from_uuid(id: Uuid) -> Self { Self(id) }
    pub fn as_uuid(&self) -> &Uuid { &self.0 }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Email(String);

impl Email {
    pub fn parse(raw: &str) -> Result<Self, String> {
        let trimmed = raw.trim().to_lowercase();
        if trimmed.contains('@') && trimmed.len() >= 3 {
            Ok(Self(trimmed))
        } else {
            Err(format!("'{}' is not a valid email", raw))
        }
    }
    pub fn as_str(&self) -> &str { &self.0 }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct UserName(String);

impl UserName {
    pub fn parse(raw: &str) -> Result<Self, String> {
        let trimmed = raw.trim();
        if trimmed.is_empty() {
            return Err("name cannot be empty".into());
        }
        if trimmed.len() > 100 {
            return Err("name cannot exceed 100 characters".into());
        }
        Ok(Self(trimmed.to_string()))
    }
    pub fn as_str(&self) -> &str { &self.0 }
}

/// The domain entity. Fields are private; access is through getters.
#[derive(Debug, Clone)]
pub struct User {
    id: UserId,
    name: UserName,
    email: Email,
    created_at: chrono::DateTime<chrono::Utc>,
}

impl User {
    /// Used by the repository when hydrating from the database.
    pub fn hydrate(
        id: UserId,
        name: UserName,
        email: Email,
        created_at: chrono::DateTime<chrono::Utc>,
    ) -> Self {
        Self { id, name, email, created_at }
    }

    pub fn id(&self) -> &UserId { &self.id }
    pub fn name(&self) -> &UserName { &self.name }
    pub fn email(&self) -> &Email { &self.email }
    pub fn created_at(&self) -> chrono::DateTime<chrono::Utc> { self.created_at }
}

/// The data needed to create a new user.
/// By the time this struct exists, all fields have been validated
/// and the password has been hashed.
pub struct NewUser {
    pub name: UserName,
    pub email: Email,
    pub password_hash: String,
}
```

Notice that `User` has no `password_hash` field. That's intentional. The entity represents the public view of a user, the thing we're comfortable handing back through the API. `NewUser` is a separate struct that carries the data needed to *create* one, including the hash. You might wonder why we don't just slap an `Option<String>` on `User` for the password. In my experience, that leads to exactly the kind of bugs where someone accidentally serializes a hash into a JSON response. Two types, two purposes, no accidents.

## Step 2: Domain errors

These live in `src/domain/errors.rs`. They describe business-level failures, the kind of things your product manager would understand, not infrastructure details like "the TCP connection timed out."

```rust
#[derive(Debug, thiserror::Error)]
pub enum CreateUserError {
    #[error("a user with email {email} already exists")]
    Duplicate { email: Email },

    #[error(transparent)]
    Unknown(#[from] anyhow::Error),
}
```

`CreateUserError::Duplicate` is a business rule violation: someone tried to register with an email that's already taken. `CreateUserError::Unknown` is our escape hatch for unexpected infrastructure failures (database timeouts, connection errors, that sort of thing). The domain doesn't know or care what specific infrastructure failure happened. It just knows something went sideways.

## Step 3: The port (repository trait)

This is the concept that makes the whole layered approach actually work. If you take away one thing from this chapter, let it be this part. A **port** is a trait defined in the domain layer that describes an operation the domain needs from the outside world. The domain says "I need something that can save users and find them," but it doesn't say *how*. The trait lives in `src/domain/ports/user_repository.rs`.

```rust
use std::future::Future;
use crate::domain::models::{User, UserId, Email, NewUser};
use crate::domain::errors::CreateUserError;

pub trait UserRepository: Send + Sync + 'static {
    fn create(
        &self,
        new_user: &NewUser,
    ) -> impl Future<Output = Result<User, CreateUserError>> + Send;

    fn find_by_id(
        &self,
        id: &UserId,
    ) -> impl Future<Output = Result<Option<User>, anyhow::Error>> + Send;

    fn find_by_email(
        &self,
        email: &Email,
    ) -> impl Future<Output = Result<Option<User>, anyhow::Error>> + Send;
}
```

Look at the imports. This trait imports only domain types. No `PgPool`, no `sqlx::query`, no `axum::State`. That's what makes it a port: it defines a boundary between the domain and the infrastructure without being coupled to either side.

The `Send + Sync + 'static` bounds are there because Axum shares state across async tasks on multiple threads. The trait uses `impl Future<...> + Send` as the return type, which works with Rust's native async-in-traits (stable since 1.75) for static dispatch. If you later need dynamic dispatch (`Arc<dyn UserRepository>`), you'd switch to the `async_trait` crate, as I explain in the [Architectural Patterns](./architecture.md) chapter.

**Why bother with a port at all?** I think of it as having two payoffs. First, it lets you test the service layer without a database. You write a simple in-memory implementation of `UserRepository` for your tests, and the service doesn't know the difference. Second, it enforces the dependency rule at the module level: the domain can't accidentally import SQLx because the trait doesn't reference it. If someone tries to add a `PgPool` parameter to this file, they'll quickly realize it doesn't belong here.

**When you don't need a port:** If your application is small, mostly CRUD, and unlikely to have a second implementation of the repository, you can skip the trait and have the service call a concrete repository struct directly. I've done this plenty of times for side projects and internal tools. The [Anti-Patterns](./anti-patterns.md) chapter discusses when that simplification makes sense. The port starts earning its keep when you have real business logic to test, when you want to reuse the domain from multiple entry points, or when the team is large enough that architectural guardrails matter.

## Step 4: The service

The service lives in `src/domain/services/user_service.rs`. This is where the actual business logic happens. It calls the repository through the port trait we just defined.

```rust
use crate::domain::models::{User, UserId, Email, UserName, NewUser};
use crate::domain::errors::CreateUserError;
use crate::domain::ports::UserRepository;

#[derive(Clone)]
pub struct UserService<R: UserRepository> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    pub fn new(repo: R) -> Self {
        Self { repo }
    }

    pub async fn register(
        &self,
        name: UserName,
        email: Email,
        password: &str,
    ) -> Result<User, CreateUserError> {
        // Hash the password on a blocking thread so we do not
        // tie up the async runtime with CPU-intensive work.
        let password = password.to_string();
        let password_hash = tokio::task::spawn_blocking(move || {
            hash_password(&password)
        })
        .await
        .map_err(|e| CreateUserError::Unknown(e.into()))?
        .map_err(|e| CreateUserError::Unknown(e.into()))?;

        let new_user = NewUser { name, email, password_hash };
        self.repo.create(&new_user).await
    }

    pub async fn get_by_id(&self, id: &UserId) -> Result<Option<User>, anyhow::Error> {
        self.repo.find_by_id(id).await
    }
}
```

The service is generic over `R: UserRepository`. It has no idea whether `R` is a real PostgreSQL repository or a test double. It just calls the trait methods and goes about its business. The `register` method is where the interesting stuff happens: it hashes the password on a blocking thread (you really don't want bcrypt tying up your async runtime) and constructs the `NewUser` struct that the repository will persist.

Also notice what the service *doesn't* know about. It has no concept of HTTP status codes, JSON serialization, or request/response types. It accepts domain types (`UserName`, `Email`) and returns domain types (`User`, `CreateUserError`). Translating between HTTP and domain concepts? That's the handler's job, and we'll get to it shortly.

## Step 5: The repository implementation

Now we cross the boundary into infrastructure. This file lives in `src/infra/repositories/user_repo.rs` and implements the port trait using SQLx. This is where the SQL actually lives.

```rust
use anyhow::Context;
use sqlx::PgPool;

use crate::domain::models::{User, UserId, Email, UserName, NewUser};
use crate::domain::errors::CreateUserError;
use crate::domain::ports::UserRepository;

/// The database row type. This is separate from the domain entity
/// because it maps to the database schema, which may differ from
/// the domain's representation.
#[derive(sqlx::FromRow)]
struct UserRow {
    id: uuid::Uuid,
    name: String,
    email: String,
    password_hash: String,
    created_at: chrono::DateTime<chrono::Utc>,
}

/// Convert a database row to a domain entity.
/// Uses TryFrom because stored data may not satisfy current
/// domain invariants (legacy rows, manual SQL fixes, migrations).
impl TryFrom<UserRow> for User {
    type Error = anyhow::Error;

    fn try_from(row: UserRow) -> Result<Self, Self::Error> {
        Ok(User::hydrate(
            UserId::from_uuid(row.id),
            UserName::parse(&row.name)
                .map_err(|e| anyhow::anyhow!("corrupt name in row {}: {}", row.id, e))?,
            Email::parse(&row.email)
                .map_err(|e| anyhow::anyhow!("corrupt email in row {}: {}", row.id, e))?,
            row.created_at,
        ))
    }
}

#[derive(Clone)]
pub struct PostgresUserRepo {
    pool: PgPool,
}

impl PostgresUserRepo {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

impl UserRepository for PostgresUserRepo {
    async fn create(&self, new_user: &NewUser) -> Result<User, CreateUserError> {
        let row = sqlx::query_as!(
            UserRow,
            r#"
            INSERT INTO users (id, email, name, password_hash)
            VALUES ($1, $2, $3, $4)
            RETURNING id, email, name, password_hash, created_at
            "#,
            uuid::Uuid::new_v4(),
            new_user.email.as_str(),
            new_user.name.as_str(),
            &new_user.password_hash,
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| match e {
            sqlx::Error::Database(ref db_err) if db_err.is_unique_violation() => {
                CreateUserError::Duplicate { email: new_user.email.clone() }
            }
            other => CreateUserError::Unknown(
                anyhow::anyhow!(other).context("failed to insert user")
            ),
        })?;

        row.try_into()
            .map_err(|e: anyhow::Error| CreateUserError::Unknown(e))
    }

    async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, anyhow::Error> {
        let row = sqlx::query_as!(
            UserRow,
            "SELECT id, email, name, password_hash, created_at FROM users WHERE id = $1",
            id.as_uuid(),
        )
        .fetch_optional(&self.pool)
        .await
        .context("failed to fetch user by id")?;

        row.map(TryInto::try_into).transpose()
    }

    async fn find_by_email(&self, email: &Email) -> Result<Option<User>, anyhow::Error> {
        let row = sqlx::query_as!(
            UserRow,
            "SELECT id, email, name, password_hash, created_at FROM users WHERE email = $1",
            email.as_str(),
        )
        .fetch_optional(&self.pool)
        .await
        .context("failed to fetch user by email")?;

        row.map(TryInto::try_into).transpose()
    }
}
```

This file, along with the wiring in `main.rs` (which uses `PgPool` and `sqlx::migrate!`), are the only places in our entire application that import `sqlx`. I want to emphasize that because it's the whole point: the **domain and API layers** never see SQLx types. The handler never touches a `UserRow` or a `sqlx::Error`. Those details stay locked inside the infrastructure layer where they belong. If you ever decide to swap Postgres for something else (unlikely, but it happens), you'd change this file and the domain wouldn't even blink.

## Step 6: DTOs and the AppError

The request and response types live in `src/api/dtos/user_dto.rs`. These are the shapes that the outside world sees, and they're deliberately separate from our domain types:

```rust
use serde::{Deserialize, Serialize};
use validator::Validate;

/// What the client sends. Raw strings, validated by the validator crate.
#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserDto {
    #[validate(length(min = 1, max = 100))]
    pub name: String,

    #[validate(email)]
    pub email: String,

    #[validate(length(min = 8))]
    pub password: String,
}

/// What the client receives. No password_hash, no internal IDs.
#[derive(Debug, Serialize)]
pub struct UserResponse {
    pub id: uuid::Uuid,
    pub name: String,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

impl From<crate::domain::models::User> for UserResponse {
    fn from(user: crate::domain::models::User) -> Self {
        Self {
            id: *user.id().as_uuid(),
            name: user.name().as_str().to_string(),
            email: user.email().as_str().to_string(),
            created_at: user.created_at(),
        }
    }
}
```

Then we have the `AppError` in `src/error.rs`, which is the glue between our domain errors and HTTP. This might look like boilerplate, and honestly it kind of is, but it gives you a single place where you decide "this business error becomes that HTTP status code":

```rust
use std::collections::HashMap;
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("not found")]
    NotFound,
    #[error("{0}")]
    Validation(String),
    #[error("unauthorized")]
    Unauthorized,
    #[error("{0}")]
    Conflict(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

pub type AppResult<T> = Result<T, AppError>;

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            Self::NotFound => (StatusCode::NOT_FOUND, self.to_string()),
            Self::Validation(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            Self::Unauthorized => (StatusCode::UNAUTHORIZED, self.to_string()),
            Self::Conflict(msg) => (StatusCode::CONFLICT, msg.clone()),
            Self::Internal(e) => {
                tracing::error!(error = ?e, "internal error");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error".into())
            }
        };
        let body = serde_json::json!({ "error": { "message": message } });
        (status, Json(body)).into_response()
    }
}

// Domain error -> AppError conversion
impl From<crate::domain::errors::CreateUserError> for AppError {
    fn from(err: crate::domain::errors::CreateUserError) -> Self {
        use crate::domain::errors::CreateUserError;
        match err {
            CreateUserError::Duplicate { email } => {
                AppError::Conflict(format!("user with email {} already exists", email.as_str()))
            }
            CreateUserError::Unknown(e) => AppError::Internal(e),
        }
    }
}
```

The handler returns `AppResult<T>`, and when it contains an `Err`, Axum calls `into_response()` to produce the HTTP error. What I like about this pattern is that the handler itself never has to think about status codes for error cases. It just uses `?` and the `From` impls take care of the rest.

## Step 7: The handler

The handler lives in `src/api/handlers/users.rs`. If you've been reading carefully, you might expect this to be the simplest file so far. And it is. The pattern is: extract, parse, delegate, respond.

```rust
use axum::{extract::State, http::StatusCode, Json};

use crate::api::dtos::user_dto::{CreateUserDto, UserResponse};
use crate::api::extractors::ValidatedJson;
use crate::api::state::AppState;
use crate::domain::models::{UserName, Email};
use crate::error::{AppError, AppResult};

pub async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    // Parse raw strings into domain types.
    // If parsing fails, it becomes a validation error.
    let name = UserName::parse(&payload.name)
        .map_err(|e| AppError::Validation(e))?;
    let email = Email::parse(&payload.email)
        .map_err(|e| AppError::Validation(e))?;

    // Delegate to the service. The ? operator converts
    // CreateUserError -> AppError automatically via the From impl.
    let user = state.user_service
        .register(name, email, &payload.password)
        .await?;

    // Convert the domain entity to a response DTO.
    Ok((StatusCode::CREATED, Json(user.into())))
}
```

Fourteen lines of actual logic. That's it. No SQL, no password hashing, no business rules. It extracts the request, parses the fields into domain types, calls the service, and returns the response. If anything fails, the `?` operator propagates the error through the `From` chain until it becomes an HTTP response. I find that when handlers stay this thin, they're almost impossible to get wrong, and the code review basically writes itself.

## Step 8: AppState and wiring

The `AppState` in `src/api/state.rs` holds all the shared dependencies. Think of it as the bag of stuff our handlers need to do their work:

```rust
use axum::extract::FromRef;
use sqlx::PgPool;
use std::sync::Arc;

use crate::config::Config;
use crate::domain::services::UserService;
use crate::infra::repositories::PostgresUserRepo;

#[derive(Clone, FromRef)]
pub struct AppState {
    pub config: Arc<Config>,
    pub db: PgPool,
    pub user_service: UserService<PostgresUserRepo>,
}
```

And then `main.rs` wires everything together. This is the moment where all our abstractions meet the real world:

```rust
use crate::api::state::AppState;
use crate::config::Config;
use crate::infra::repositories::PostgresUserRepo;
use crate::domain::services::UserService;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    init_tracing(&config.log_level);

    // Infrastructure: create the database pool.
    let pool = create_pool(config.database_url.expose_secret()).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;

    // Infrastructure: create the repository (implements the port).
    let user_repo = PostgresUserRepo::new(pool.clone());

    // Domain: create the service, injecting the repository.
    let user_service = UserService::new(user_repo);

    // API: assemble the state and router.
    let state = AppState {
        config: Arc::new(config.clone()),
        db: pool,
        user_service,
    };

    let app = api::routes::router(state);

    let addr = format!("0.0.0.0:{}", config.port);
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    tracing::info!("listening on {}", listener.local_addr()?);
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}
```

This is our **composition root**: the one place in the application where we choose concrete types and wire them together. The domain service receives a `PostgresUserRepo`, but it only knows it as "something that implements `UserRepository`." If you ever wanted to swap in a different database (or even a different backing store entirely), you'd change this file and the `infra/` module. The domain and API layers wouldn't need to change at all, which is a really nice property to have when your application starts growing.

## The complete data flow

Let's trace through the whole thing end to end. Here's what happens when a client sends `POST /api/v1/users`:

```
Client sends:  { "name": "Alice", "email": "alice@example.com", "password": "secret123" }

  1. Axum deserializes into CreateUserDto (api/dtos)
  2. ValidatedJson checks: name length, email format, password length
  3. Handler parses fields into domain types: UserName, Email
  4. Handler calls user_service.register(name, email, &password)
  5. Service hashes password on blocking thread
  6. Service builds NewUser and calls repo.create(&new_user)
  7. Repository executes INSERT via sqlx::query_as!
  8. If email is duplicate: sqlx returns unique violation
     → Repository maps to CreateUserError::Duplicate
     → Handler's ? maps to AppError::Conflict
     → Axum returns 409 with error message
  9. If successful: repository gets UserRow back from RETURNING clause
     → TryFrom<UserRow> converts to User (domain entity)
  10. Service returns User to handler
  11. Handler converts User to UserResponse via From trait
  12. Axum serializes to JSON and returns 201 Created

Client receives:  { "id": "...", "name": "Alice", "email": "alice@example.com", "created_at": "..." }
```

Every boundary is explicit, and I think that's what makes this approach worth the extra files. The handler works with DTOs and domain types. The service works with domain types and ports. The repository works with SQL and row types. And the `From` / `TryFrom` implementations bridge the gaps. No layer reaches into another layer's internals. If that sounds like a lot of ceremony for one endpoint, you're not wrong. But once you've done it once, every subsequent feature follows the same shape, and the consistency pays for itself quickly.

## Scaling up: bigger features, more services

As your application grows, you'll add more features. Each one follows the same pattern: domain types, a port if needed, a service, a repository, DTOs, and a handler. Once you've built two or three features this way, the structure becomes second nature.

At some point you'll notice that the flat `models/`, `services/`, and `repositories/` directories start getting crowded. When that happens, you can regroup by feature instead:

```
src/domain/
├── users/
│   ├── mod.rs
│   ├── model.rs        # User, UserId, Email, UserName, NewUser
│   ├── errors.rs       # CreateUserError
│   ├── port.rs         # UserRepository trait
│   └── service.rs      # UserService
├── posts/
│   ├── mod.rs
│   ├── model.rs
│   ├── errors.rs
│   ├── port.rs
│   └── service.rs
```

Both layouts follow the same principles we've been using throughout this chapter. The choice between them is about file count and how easy it is to find things, not about architecture. My advice: start flat, and regroup by feature when navigating the flat directories starts to annoy you. In my experience that tipping point is somewhere around 5 to 10 domain concepts, but you'll feel it when you get there.
