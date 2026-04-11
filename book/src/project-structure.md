# Project Structure

If you've ever opened a project and had no idea where to put your new file, you know how much folder structure matters. It sounds like a small thing, but I've watched teams burn hours arguing over where code should live, and I've seen projects where everything ended up in one giant `src/` directory because nobody made a decision early on. Not fun.

There's no single correct way to structure a Rust web application, but there are patterns that work well in practice. In this chapter we'll look at two: a single-crate layout for small-to-medium projects, and a workspace layout for when things get bigger. They follow the same underlying principles, just at different scales.

## Principles

Before we look at specific directory trees, let me walk you through the ideas that drive them.

**Group by architectural layer first, then by concern within each layer.** Our primary division is between `api`, `domain`, and `infra`, which enforces the dependency rule we talked about earlier. Within each layer, we organize modules by concern (handlers, models, repositories). Now, I'll be honest: this still means jumping between `api/handlers/users.rs`, `domain/models/user.rs`, and `infra/repositories/user_repo.rs` when you're working on a user feature. That's a real tradeoff. As a codebase grows, some teams find it helpful to organize by feature slice within the layers (like `domain/users/model.rs`, `domain/users/service.rs`) rather than by role. Either approach works. What matters most is that the layer boundaries stay clear.

**Enforce dependency direction.** This one is the big one. Dependencies should flow inward, from the HTTP layer toward the domain. Your domain module should never import anything from `api` or `infra`. That's what keeps your business logic portable and testable in isolation. In a single crate, you enforce this by convention and code review (which, yes, means trusting your team). In a workspace, Cargo enforces it for you through the crate dependency graph, which is much nicer.

**Keep `main.rs` thin.** Your entry point should do exactly three things: load configuration, construct the application (wiring up dependencies), and start the server. That's it. If your `main.rs` is getting long, that's usually a sign that setup logic needs to move into its own modules.

**Use visibility to create boundaries.** Rust's module visibility system (`pub`, `pub(crate)`, `pub(super)`, and the default private) is your tool for controlling what code can reach what. Expose only what needs to be public through your `mod.rs` files, and keep implementation details private. If you've come from JavaScript, think of it as the barrel-export pattern, but enforced by the compiler instead of by good intentions.

## Single-crate layout

This is what I reach for on most projects. It gives you clean separation between layers without the overhead of managing multiple crates, and you can always evolve it into a workspace later if the project grows. Let's take a look.

```
my-app/
├── Cargo.toml
├── .env.example
├── migrations/
│   ├── 20240101_create_users.sql
│   └── 20240102_create_posts.sql
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── config.rs
│   ├── error.rs
│   │
│   ├── domain/
│   │   ├── mod.rs
│   │   ├── models/
│   │   │   ├── mod.rs
│   │   │   ├── user.rs
│   │   │   └── post.rs
│   │   ├── errors.rs
│   │   ├── ports/          # Trait definitions (optional, for trait-based abstractions)
│   │   │   └── user_repository.rs
│   │   └── services/
│   │       ├── mod.rs
│   │       ├── user_service.rs
│   │       └── post_service.rs
│   │
│   ├── infra/
│   │   ├── mod.rs
│   │   ├── db.rs
│   │   └── repositories/
│   │       ├── mod.rs
│   │       ├── user_repo.rs
│   │       └── post_repo.rs
│   │
│   └── api/
│       ├── mod.rs
│       ├── routes.rs
│       ├── state.rs
│       ├── handlers/
│       │   ├── mod.rs
│       │   ├── health.rs
│       │   ├── users.rs
│       │   └── posts.rs
│       ├── extractors/
│       │   ├── mod.rs
│       │   ├── validated_json.rs
│       │   └── auth_user.rs
│       ├── middleware/
│       │   ├── mod.rs
│       │   └── auth.rs
│       └── dtos/
│           ├── mod.rs
│           ├── user_dto.rs
│           └── post_dto.rs
│
├── tests/
│   ├── common/
│   │   ├── mod.rs
│   │   └── test_app.rs
│   └── api/
│       ├── health_test.rs
│       └── users_test.rs
```

Let's walk through what each piece does.

### `main.rs` and `lib.rs`

The `main.rs` file is our entry point. It loads configuration, initializes tracing, builds the application, and starts the server. That's all it does, and I want to keep it that way.

```rust
use my_app::{config::Config, create_app, observability};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    observability::init_tracing(&config.log_level);

    let app = create_app(config.clone()).await?;

    let addr = format!("0.0.0.0:{}", config.port);
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    tracing::info!("listening on {}", listener.local_addr()?);
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal()) // defined in the Deployment chapter
        .await?;

    Ok(())
}
```

The `lib.rs` file declares our modules and provides the `create_app` function. This is also what your integration tests will call, which means your tests exercise the exact same wiring as production. I really like this pattern because it means there's no "test-only" setup path that can silently diverge from the real thing.

```rust
pub mod api;
pub mod config;
pub mod domain;
pub mod error;
pub mod infra;
pub mod observability;

use axum::Router;
use config::Config;

pub async fn create_app(config: Config) -> anyhow::Result<Router> {
    let pool = infra::db::create_pool(
        config.database_url.expose_secret()
    ).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;

    let state = AppState::new(config, pool);
    let router = api::routes::router(state);
    Ok(router)
}
```

### The `domain` module

This is the heart of your application, and it should have **zero dependencies on Axum, SQLx, or any other framework crate**. The only external crates it typically needs are `serde` (for serialization), `thiserror` (for error types), and basic utility crates like `uuid` and `chrono`. That might seem limiting, but it's exactly the constraint that keeps this layer honest.

The `models/` directory contains your domain entities and value objects, the types that represent the core concepts your application works with. The `services/` directory is where the business logic lives, the code that operates on those models. And `errors.rs` defines domain-specific error types.

Now, what if your domain needs to talk to the outside world? A database, an email service, some third-party API? It defines trait contracts in a `ports/` directory. The actual implementations of those traits live over in `infra`. We'll see more of this pattern later.

### The `infra` module

This is where your database code lives. `db.rs` handles connection pool creation, and `repositories/` contains the concrete implementations of your domain's repository traits.

Each repository struct holds a reference to the connection pool and implements the corresponding trait from the domain layer. Its job is translating between the database's row types and the domain's model types. You might think of it as a translator that speaks both SQL and Rust domain language.

### The `api` module

This is the Axum-specific layer. It knows about HTTP concepts like status codes, JSON bodies, headers, and middleware. It does not contain business logic. If you find yourself writing an `if` statement about business rules in a handler, that's your cue to move it into a service.

The `handlers/` are thin functions that extract data from the request, call a service method, and return a response. I like to keep them really short, ideally under 20 lines. The `dtos/` define the shapes of request and response bodies, kept separate from the domain models so that your API contract can evolve independently of your internal representation. The `extractors/` contain custom Axum extractors like `ValidatedJson` or `AuthUser` (we'll build those in later chapters). And `middleware/` holds any custom Tower middleware.

Finally, `routes.rs` assembles the router, composing the individual route groups and applying middleware layers.

## Workspace layout

At some point your project might grow large enough that compile times start to hurt, or maybe you have multiple teams working on different parts of the system. That's when I'd consider splitting into a Cargo workspace. The workspace enforces the dependency rules at the crate level: if the `domain` crate doesn't list `sqlx` in its `Cargo.toml`, no one can accidentally import it there. The compiler has your back.

```
my-app/
├── Cargo.toml                # [workspace] definition
├── .env.example
├── migrations/
│
├── crates/
│   ├── domain/
│   │   ├── Cargo.toml        # deps: serde, thiserror, uuid, chrono
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── models/
│   │       ├── services/
│   │       ├── ports/        # trait definitions
│   │       └── errors.rs
│   │
│   ├── infra/
│   │   ├── Cargo.toml        # deps: domain, sqlx, reqwest, etc.
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── postgres/
│   │       ├── redis/
│   │       └── email/
│   │
│   ├── api/
│   │   ├── Cargo.toml        # deps: domain, infra, axum, tower, tower-http
│   │   └── src/
│   │       ├── main.rs
│   │       ├── lib.rs
│   │       ├── routes/
│   │       ├── handlers/
│   │       ├── middleware/
│   │       ├── extractors/
│   │       └── dtos/
│   │
│   └── shared/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── config.rs
│           └── observability.rs
```

The root `Cargo.toml` defines the workspace and, just as importantly, the shared dependency versions. This is one of my favorite Cargo features because it means you don't end up with three different versions of `serde` across your crates:

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono", "migrate"] }
axum = { version = "0.8", features = ["macros"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "compression-gzip", "trace", "timeout", "request-id", "limit"] }
thiserror = "2"
anyhow = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

Individual crates then inherit from these shared versions with `.workspace = true`:

```toml
# crates/domain/Cargo.toml
[package]
name = "domain"

[dependencies]
serde.workspace = true
thiserror.workspace = true
uuid.workspace = true
chrono.workspace = true
# Note: no axum, no sqlx
```

The key benefit here is that Cargo itself prevents architectural violations. If someone on your team tries to add a SQLx import in the domain crate, the build just fails because `sqlx` isn't in its dependency list. I've seen enough bugs from "oops, I accidentally coupled the domain to the database" that I really appreciate having the compiler catch this for me. It's a much stronger guarantee than relying on code review alone.

## When to choose which

My advice: start with the single-crate layout. Seriously. It gives you all the structural benefits of clean separation without the overhead of managing a workspace. You can always split into a workspace later when the need actually arises, and it's not a particularly difficult refactor because the module boundaries are already in place.

So when would you actually move to a workspace? In my experience, it's when one of these starts to bite:

- Compile times are getting painful and you want incremental builds to only recompile what changed
- Multiple teams are working on different parts of the system and you want hard boundaries between their areas
- You want to publish some crates independently (for example, a shared domain library used by multiple services)
- The project has grown to the point where a single `src/` directory feels unwieldy

The workspace layout also works well for monorepo setups where you have multiple services (an API server, a background worker, a CLI tool) that share the same domain and infrastructure code. We'll touch on that kind of setup in later chapters when we talk about background jobs.
