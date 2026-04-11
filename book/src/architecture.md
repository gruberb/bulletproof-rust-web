# Architectural Patterns

If you've ever worked on a codebase where the database query logic was mixed into the HTTP handler, and the business rules lived in three different places, you know how painful that gets over time. I've been there more than once, and it always ends the same way: you're afraid to change anything because you can't tell what depends on what.

This chapter is about the patterns that prevent that kind of mess. We'll look at how to structure a Rust web application so the pieces stay clean and independent. I'm not going to prescribe one true way, because the right level of structure depends on your project. But I do want you to understand the options well enough to make that call yourself.

The common thread across all of these patterns is the **dependency rule**: dependencies point inward, from infrastructure toward the domain. Your business logic should never know about HTTP status codes, SQL queries, or which web framework you're using. When you get this right, your domain becomes portable, testable, and surprisingly easy to evolve as things change around it.

## The layered model

The simplest architecture that actually holds up in production divides your code into three layers. You've probably seen some version of this before, but let's walk through what each layer does and, more importantly, what it should not be doing.

```
┌──────────────────────────────────────────────┐
│   API Layer                                  │
│   Handlers, routes, middleware, DTOs         │
│   Knows about: Axum, HTTP, JSON              │
├──────────────────────────────────────────────┤
│   Domain Layer                               │
│   Models, services, business rules, ports    │
│   Knows about: nothing external              │
├──────────────────────────────────────────────┤
│   Infrastructure Layer                       │
│   Repositories, database, external APIs      │
│   Knows about: SQLx, Diesel, reqwest, etc.   │
└──────────────────────────────────────────────┘
```

The API layer is the translation layer between HTTP and your domain. It deserializes request bodies into domain types, calls service methods, and serializes the results back into HTTP responses. Keep it thin. If you find yourself writing business logic in a handler, stop. That logic belongs in the domain layer. I've seen enough handlers turn into 200-line monsters to know this is worth being strict about early.

The domain layer is the heart of the whole thing. This is where your business entities live, your value objects, your service functions, and the trait definitions (ports) that describe what the domain needs from the outside world, things like "save a user" or "send a notification." The key part: this layer has no idea how those capabilities are actually implemented. It doesn't import `sqlx` or `axum` or any other framework crate. It's just pure business logic.

The infrastructure layer provides the concrete implementations of those domain traits. It knows about specific databases, external HTTP APIs, email providers, message queues, all of that. It depends on the domain layer (because it implements the domain's traits), but the domain layer never depends on it. That one-way dependency is what makes the whole thing work.

### Why this ordering matters

You might look at this and think "okay, that's a nice diagram, but does it actually matter in practice?" It really does. Let me give you the concrete payoffs.

When your domain doesn't depend on your database, you can test your business logic by plugging in a simple in-memory implementation of the repository trait. No running PostgreSQL instance needed just to verify that your pricing calculation is correct. Your test suite runs in milliseconds, and you can run it on a plane. I love that.

When your domain doesn't depend on Axum, you can reuse it in a CLI tool, a background worker, or a gRPC service without dragging the entire HTTP layer along for the ride.

And when your database implementation changes (say you migrate from PostgreSQL to CockroachDB, or you swap SQLx for Diesel), the domain layer doesn't need to change at all. You rewrite the infrastructure layer and the rest of the application doesn't even notice. I've actually done this migration on a real project, and it went way smoother than I expected precisely because we had this separation in place.

## Trait-based abstractions (ports and adapters)

So how do we actually enforce this separation in Rust? Traits. The domain layer defines traits that describe the operations it needs. The infrastructure layer provides structs that implement those traits. And the application layer wires everything together at startup. If you've worked with interfaces in other languages, this will feel familiar, but Rust's trait system gives us some really nice compile-time guarantees on top.

Here is what a repository port looks like in the domain layer:

```rust
// domain/ports/user_repository.rs

use std::future::Future;
use crate::domain::models::{User, UserId, Email};
use crate::domain::errors::CreateUserError;

pub trait UserRepository: Send + Sync + 'static {
    fn create(
        &self,
        req: &CreateUserRequest,
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

A few things to note about the trait bounds. The `Send + Sync + 'static` bounds are there because Axum needs to share state across async tasks that might run on different threads. If you've ever fought a "future is not Send" compiler error, this is why those bounds matter. Since Rust 1.75, `async fn` in traits and return-position `impl Trait` are stable, so you can write these signatures without the `#[async_trait]` macro. That was a really welcome change.

There are two limitations to be aware of, though. First, traits that use `-> impl Future` or `async fn` are **not object-safe**, which means you can't use them as trait objects (`dyn Trait`). Concretely, `Arc<dyn UserRepository>` won't compile. If you need dynamic dispatch (we'll cover that below), you still need either the `async_trait` crate or manually boxed futures. The pattern shown here works with static dispatch (generics), which is what I'd recommend as your default.

Second, for **public library traits** (as opposed to internal application ports), bare `async fn` is problematic because callers can't add bounds like `Send` to the returned future without a breaking API change. If you're writing a trait that will be consumed by code outside your crate, consider using the `trait_variant` crate (from the Rust async working group) to generate a `Send` variant automatically:

```rust
#[trait_variant::make(UserRepositorySend: Send)]
pub trait UserRepository { ... }
```

For internal application ports (which is what most Axum services need), native async traits work great.

One more small thing: notice that `Clone` is left off the trait definition. Cloneability is a property of the holder (the `Arc` wrapper, the `AppState` struct), not a requirement of the domain contract itself. Your concrete repository implementations can still derive `Clone` if they need to. Keeping `Clone` out of the trait keeps the contract focused on what the domain actually cares about.

Now let's look at the concrete implementation in the infrastructure layer:

```rust
// infra/repositories/postgres_user_repo.rs

use domain::ports::UserRepository;

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
    async fn create(&self, req: &CreateUserRequest) -> Result<User, CreateUserError> {
        let row = sqlx::query_as!(
            UserRow,
            r#"INSERT INTO users (id, email, name, password_hash)
               VALUES ($1, $2, $3, $4)
               RETURNING *"#,
            Uuid::new_v4(),
            req.email.as_str(),
            req.name.as_str(),
            req.password_hash,
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| match e {
            sqlx::Error::Database(ref db_err) if db_err.is_unique_violation() => {
                CreateUserError::Duplicate { email: req.email.clone() }
            }
            other => CreateUserError::Unknown(other.into()),
        })?;

        Ok(row.into())
    }

    // ... other methods
}
```

Notice how the infrastructure layer translates database-specific errors (like unique constraint violations) into domain-specific errors. The domain error type knows about "duplicate user," not about SQL error codes. This is a small thing that makes a big difference when you're debugging at 2 AM, because your error messages actually tell you what went wrong in business terms.

## The service layer

Services sit inside the domain layer and orchestrate the business logic. They take the domain's ports as generic parameters (or trait objects, if you go the dynamic dispatch route), call them in the right order, and enforce business rules. This is where the interesting stuff lives. If you want to understand what the application actually does, you should be able to read the service layer and get the full picture without wading through HTTP concerns or SQL queries.

```rust
// domain/services/user_service.rs

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
        let password_hash = hash_password(password)?;

        let req = CreateUserRequest {
            name,
            email,
            password_hash,
        };

        self.repo.create(&req).await
    }

    pub async fn get_by_id(&self, id: &UserId) -> Result<Option<User>, anyhow::Error> {
        self.repo.find_by_id(id).await
    }
}
```

The service has no idea that `R` is a PostgreSQL repository. It could be an in-memory implementation used in tests, or a caching wrapper that checks Redis before hitting the database. The service only knows that whatever `R` is, it satisfies the `UserRepository` contract. That's the whole magic of this pattern, and it's what makes writing tests for your business logic genuinely pleasant.

## Static vs. dynamic dispatch

Once you're using trait-based abstractions, you'll run into this question: should I use generics (static dispatch) or trait objects (dynamic dispatch)? It's worth understanding the tradeoff.

**Static dispatch** uses generics. The compiler monomorphizes the code for each concrete type, which means there's no runtime overhead from virtual dispatch. This is the approach we used above with `UserService<R: UserRepository>`.

```rust
// Static dispatch: zero runtime cost, resolved at compile time
pub struct UserService<R: UserRepository> {
    repo: R,
}
```

**Dynamic dispatch** uses trait objects behind `Arc`. The method calls go through a vtable at runtime, which adds a small overhead. In practice this is negligible for web applications where your database round-trips dominate the latency budget anyway. The benefit is simpler type signatures and easier interchangeability at runtime.

Here's the catch, though: traits that use `-> impl Future` or `async fn` are not object-safe in Rust, so you can't use them with `dyn Trait` directly. If you need dynamic dispatch, you have two options. The `async_trait` crate rewrites async methods into boxed futures, making the trait object-safe:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait UserRepository: Send + Sync + 'static {
    async fn create(&self, req: &CreateUserRequest) -> Result<User, CreateUserError>;
    async fn find_by_id(&self, id: UserId) -> anyhow::Result<Option<User>>;
}

// Now this works:
pub struct UserService {
    repo: Arc<dyn UserRepository>,
}
```

You can also write the boxed futures by hand, but honestly the `async_trait` macro does the same thing with less boilerplate and it's not worth the extra typing.

My recommendation: **start with static dispatch**. The compile-time guarantees are stronger, the error messages are clearer, and you sidestep the whole object-safety question. Reach for dynamic dispatch when you genuinely need runtime flexibility, like swapping implementations based on configuration, or when the generic type parameters start making your signatures unwieldy across many layers. You'll know when you hit that point because you'll be staring at a type signature that looks like alphabet soup.

## Hexagonal architecture and onion architecture

If you've spent any time reading about software architecture, you've probably bumped into these terms. They sound fancy, but they're really just variations on the same core idea we've been talking about: put the domain at the center, push framework-specific code to the edges.

**Hexagonal architecture** (also called ports and adapters) emphasizes the symmetry between inbound adapters (things that call into your domain, like HTTP handlers or gRPC servers) and outbound adapters (things your domain calls out to, like databases or email services). The "ports" are the trait definitions, and the "adapters" are the concrete implementations. We've already been doing this.

**Onion architecture** visualizes the same idea as concentric rings. The innermost ring is the domain model. The next ring out is the application/service layer. Then comes the infrastructure layer. And the outermost ring is the presentation layer. Dependencies always point inward.

In practice, these are different names for the same fundamental pattern. The three-layer model we walked through at the beginning of this chapter is a pragmatic implementation of these ideas. You don't need to adopt any particular naming convention or follow the formal structure to the letter. What I care about (and what you should care about) is that the dependency rule is understood and enforced. The labels are just labels.

## Choosing the right level of architecture

I am a pragmatist at heart, and I want to be honest with you: not every project needs a full hexagonal architecture with trait-based ports, generic services, and a five-crate workspace. Over-engineering is a real cost. It adds boilerplate, it increases cognitive overhead, and it slows down development for benefits that may never materialize. I've over-engineered plenty of things myself and regretted it every time.

Here's a rough guide based on what I've found works well:

**Flat modules** are great for prototypes, small APIs, and projects with a handful of endpoints. Put your handlers, models, and queries in separate modules, but don't bother with traits or service layers. You can always refactor later, and for a small service, the simpler approach is genuinely better.

**Layered architecture (single crate)** is the sweet spot for most production applications. Separate your domain from your infrastructure, use services to encapsulate business logic, and keep handlers thin. This gives you testability and maintainability without excessive ceremony. If you're building something that will run in production and be maintained by a team, this is probably where you want to start.

**Hexagonal/onion architecture (workspace)** makes sense when you have a complex domain with significant business logic, multiple teams working in parallel, or a need to support multiple inbound channels (HTTP, gRPC, CLI, message consumers) against the same domain. The upfront investment is real, but it pays off over the lifetime of a long-lived codebase. I've worked on projects like this where having the clean separation saved us weeks of refactoring pain later.

The important thing is to start with clear separation between your layers, even in the simplest structure. The specific patterns you use to enforce that separation can evolve as the project grows. Moving from direct function calls to trait-based abstractions is a pretty straightforward refactor when the module boundaries are already clean. So don't stress about getting the architecture "perfect" from day one. Get the layers right, and the rest will follow.
