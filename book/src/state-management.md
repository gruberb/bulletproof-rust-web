# State Management

If you've built even a small web service, you know the problem: your handlers need access to shared things like a database pool, configuration, service instances, maybe an in-memory cache. Every request needs some or all of that, and you need a way to get it there without passing it through seventeen function arguments. Axum gives us a type-safe way to handle this through the `State` extractor, and once you get the hang of it, it becomes second nature.

## The AppState struct

The pattern I reach for every time is a single struct that holds everything our handlers need. We pass it to the router via `.with_state()`, and Axum takes care of the rest. One thing to keep in mind: the struct has to implement `Clone`, because Axum clones it for each handler invocation.

```rust
use sqlx::PgPool;
use std::sync::Arc;

#[derive(Clone)]
pub struct AppState {
    pub config: Arc<Config>,
    pub db: PgPool,
    pub user_service: UserService<PostgresUserRepo>,
    pub post_service: PostService<PostgresPostRepo>,
}

impl AppState {
    pub fn new(config: Config, pool: PgPool) -> Self {
        let user_repo = PostgresUserRepo::new(pool.clone());
        let post_repo = PostgresPostRepo::new(pool.clone());

        Self {
            config: Arc::new(config),
            db: pool,
            user_service: UserService::new(user_repo),
            post_service: PostService::new(post_repo),
        }
    }
}
```

Let me walk through a few design decisions here.

We wrap `Config` in `Arc` because it's read-only after startup and could be fairly large. `Arc` gives us cheap clones (it just bumps a reference count) instead of deep-copying the whole config struct every time `AppState` gets cloned. That difference matters when you're handling thousands of requests per second.

`PgPool` is already reference-counted internally, so cloning it is cheap too. You'll find that most connection pool types in the Rust ecosystem work the same way.

The services and repositories get constructed once at startup, then shared across all requests. I think of this as the composition root of our application, the one place where we wire all the dependencies together. If you need to swap in a test double for a repository, this is where you'd do it.

## Extracting state in handlers

So how do our handlers actually get at the state? Through the `State` extractor:

```rust
async fn list_users(
    State(state): State<AppState>,
) -> AppResult<Json<Vec<UserResponse>>> {
    let users = state.user_service.list_all().await?;
    Ok(Json(users.into_iter().map(Into::into).collect()))
}
```

Pretty straightforward, right? But you might notice that every handler receives the entire `AppState`, even if it only touches one field. For a small application, that's totally fine. Once things grow, though, you'll probably want something more granular.

## Sub-state with FromRef

This is where `FromRef` comes in. It lets you extract individual fields from your state directly, so the handler doesn't need to know about the full `AppState` struct at all. I find this really helpful for keeping handler signatures focused and limiting what each handler can actually reach.

Axum gives us a derive macro that generates the `FromRef` implementations for each field automatically:

```rust
use axum::extract::FromRef;

#[derive(Clone, FromRef)]
pub struct AppState {
    config: Arc<Config>,
    db: PgPool,
    user_service: UserService<PostgresUserRepo>,
    post_service: PostService<PostgresPostRepo>,
}
```

That one `derive` saves us from writing a manual `impl FromRef<AppState> for ...` block for every single field. It generates an implementation for each field's type, using the field name to find it in the struct.

Now a handler that only needs the user service can ask for just that:

```rust
async fn get_user(
    State(user_service): State<UserService<PostgresUserRepo>>,
    Path(id): Path<Uuid>,
) -> AppResult<Json<UserResponse>> {
    let user = user_service.get_by_id(UserId::from_uuid(id))
        .await?
        .ok_or(AppError::NotFound)?;
    Ok(Json(user.into()))
}
```

This pattern plays nicely with custom extractors too. For example, our `AuthUser` extractor (which we'll build in the [Authentication](./authentication.md) chapter) can pull the JWT secret from the state via `FromRef` without knowing about any other fields. Clean separation.

## State vs. Extension

You might wonder why Axum has both `State` and `Extension` for passing data to handlers. I've seen people reach for `Extension` out of habit (maybe from other frameworks), so let me explain when to use which.

**State** is type-checked at compile time. If you forget to call `.with_state()`, or if the types don't match, the compiler catches it. That alone makes it my default choice.

**Extension** stores values in a type-map on the request itself. It's only checked at runtime, which means a mismatch shows up as a 500 error when an actual request hits the handler. Not great. But extensions are genuinely useful for values that middleware attaches on a per-request basis, like an authenticated user extracted from a JWT.

So here's how I think about it: use `State` for application-wide dependencies that you set up at startup (config, database pools, services). Use `Extension` for per-request data that middleware produces on the fly.

## Avoiding common pitfalls

Before we move on, let me share a few things that have bitten me (or people on my team) in production.

**Don't put `Mutex<T>` in your state unless you've really thought it through.** I've seen enough bugs from this one. A mutex held across an `.await` point can deadlock or cause brutal contention in async code. If you need shared mutable state (like an in-memory cache), reach for `tokio::sync::RwLock` which is async-aware, or better yet, use a crate like `moka` or `dashmap` that's built for concurrent access.

**Be intentional about what goes into AppState.** It's tempting to keep tossing fields in there as the application grows, but a state struct with dozens of fields gets hard to reason about fast. If your state is getting unwieldy, that's usually a signal that your app could benefit from splitting into more focused modules, each with their own state type that you compose together at the router level.

**Construct your state once, at startup.** Don't create new service or repository instances inside handlers. The whole point of the `AppState` pattern is that we wire dependencies together in one place (our `main` function or a setup function it calls) and then share them immutably across all requests. Once you internalize that, the structure of your application becomes much easier to follow.
