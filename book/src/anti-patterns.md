# Anti-Patterns

Every Rust web project I've worked on has stumbled into the same handful of mistakes. Some of them I made myself, some I caught in code review, and a few I inherited in codebases that taught me the hard way. This chapter collects the ones I see most often, explains why they'll hurt you, and points to the better alternative covered elsewhere in this book.

## SQL in handlers

**What it looks like:** A handler function that contains `sqlx::query!` calls directly, managing transactions and mapping database errors right there in the HTTP handler body.

**Why this hurts:** It welds your HTTP layer to your database schema. You can't test the handler without a database. You can't reuse the query logic from a different entry point (a CLI tool, a background worker, another handler). And when the query changes, you're editing a file that's supposed to be about HTTP concerns.

**What to do instead:** Move database access into repository or query structs, as we describe in the [Database Layer](./database.md) chapter. The handler calls a service or repository method and works with domain types. It never touches SQL or database-specific error types.

**When this is actually fine:** For small, CRUD-shaped services where each endpoint maps directly to one or two queries, and there's no realistic prospect of a second entry point, handler-local SQL can be a legitimate simplification. The [launchbadge/realworld-axum-sqlx](https://github.com/launchbadge/realworld-axum-sqlx) reference implementation takes this approach deliberately, and its inline queries are well-structured and readable. The important thing is to make that choice on purpose rather than by default, and to recognize when a handler is accumulating enough query logic that it should be extracted. If you find yourself duplicating a query across handlers, or if a handler is orchestrating multiple queries that need to succeed or fail together, that's your signal to introduce a repository or service layer.

## Fat handlers

**What it looks like:** A handler that validates input, hashes passwords, queries the database, sends an email notification, publishes an event, and builds the HTTP response, all in one function.

**Why this hurts:** It makes the handler impossible to test in isolation, difficult to understand, and painful to modify. It also means that adding a second entry point for the same business logic (say, an admin endpoint or a message queue consumer) requires duplicating all of that logic.

**What to do instead:** Keep your handlers thin. Extract business logic into service methods in the domain layer. The handler's job is to pull data from the request, call a service method, and return a response. If your handler is longer than 15 to 20 lines, it's probably doing too much.

## Using `unwrap()` in request paths

**What it looks like:** Calling `.unwrap()` or `.expect()` on a `Result` or `Option` inside a handler or any code that runs during request processing.

**Why this hurts:** A panic in a handler brings down that request with an opaque 500 error. Worse, if the panic happens while holding a mutex, it poisons the mutex and can cascade to other requests. The client gets no useful error information, and your logs may not capture the full context.

**What to do instead:** Always use the `?` operator to propagate errors through the `Result` type. If you need to convert an `Option` to an error, use `.ok_or(AppError::NotFound)?` or something similar. Reserve `unwrap()` for situations where you can prove the value is always present (like a hardcoded regex) and `expect()` for startup code where a missing value means the application can't run.

## Leaking database types into the API

**What it looks like:** Returning the same struct from the database layer and the HTTP handler, or deriving both `sqlx::FromRow` and `serde::Serialize` on the same type and returning it directly as `Json<UserRow>`.

**Why this hurts:** It means that adding a database column automatically adds it to your API response. If that column contains sensitive data (a password hash, an internal flag, a soft-delete marker), it's now visible to every API consumer. It also means that renaming or restructuring a database column becomes a breaking API change.

**What to do instead:** Maintain separate types for each boundary, as we describe in the [Domain Modeling](./domain-modeling.md) chapter. `UserRow` maps to the database. `User` represents the domain entity. `UserResponse` defines the API contract. Explicit `From` implementations bridge the gaps and make the conversions easy to audit.

## Exposing internal errors to clients

**What it looks like:** Returning the raw error message from a database error, a file system error, or an internal service call directly in the HTTP response body.

**Why this hurts:** Internal error messages can contain database table names, file paths, SQL queries, stack traces, and other information that helps an attacker understand your system's internals. Even without malicious intent, these messages confuse API consumers because they describe implementation details rather than the actual business problem.

**What to do instead:** Log the full error details server-side at the `error` level. Return a generic, user-facing message to the client. The `AppError::Internal` variant we describe in the [Error Handling](./error-handling.md) chapter handles this pattern cleanly.

## Hardcoded secrets

**What it looks like:** A JWT secret or database password defined as a string literal in the source code, or committed to version control in a `.env` file.

**Why this hurts:** Anyone with access to the source code (or the git history) now has your production credentials. That includes every developer on the team, every CI system that checks out the repo, and anyone who finds the repo if it ever accidentally goes public.

**What to do instead:** Load secrets from environment variables, as we describe in the [Configuration](./configuration.md) chapter. Use `SecretString` from the `secrecy` crate to protect them in memory. Add `.env` to `.gitignore`. Provide a `.env.example` file that documents the required variables without containing actual values.

## Cross-layer imports

**What it looks like:** The domain module importing something from the `api` module (like an Axum extractor type) or from the `infra` module (like a `PgPool`).

**Why this hurts:** It breaks the dependency rule that's the foundation of clean architecture. If the domain depends on Axum, you can't use it in a CLI tool without pulling in the entire HTTP framework. If it depends on SQLx, you can't test it without a database.

**What to do instead:** Dependencies point inward. The API layer depends on the domain. The infrastructure layer depends on the domain. The domain depends on nothing external. If the domain needs a capability from the outside world, it defines a trait (a port) that the infrastructure layer implements (as an adapter). We cover this in detail in the architecture chapters.

## Blocking the async runtime

**What it looks like:** Calling CPU-intensive functions (like Argon2 password hashing), synchronous file I/O, or blocking library calls directly inside an async handler.

**Why this hurts:** Tokio's runtime uses a small number of worker threads to run many async tasks cooperatively. A task that blocks a thread prevents all other tasks on that thread from making progress. Under load, this causes latency spikes and throughput degradation that are really hard to diagnose.

**What to do instead:** Use `tokio::task::spawn_blocking()` for CPU-intensive or blocking work. Use `tokio::fs` instead of `std::fs`. Make sure your database driver is truly async (SQLx and diesel-async are; the base Diesel is not).

## String-typed domain values

**What it looks like:** Representing emails, usernames, and other domain values as plain `String` types throughout the application.

**Why this hurts:** A function that takes `(String, String, String)` for name, email, and password can be called with the arguments in any order, and the compiler won't catch the mistake. You also lose the ability to enforce invariants (like "email must contain @") in a single place, which leads to validation logic scattered all over the codebase.

**What to do instead:** Use the newtype pattern we describe in the [Domain Modeling](./domain-modeling.md) chapter. Define types like `Email`, `UserName`, and `UserId` that enforce their invariants at construction time. It might seem like extra boilerplate at first, but it pays for itself quickly.

## Missing health checks

**What it looks like:** An application deployed to a container orchestrator with no health check endpoints, or with a liveness probe that checks external dependencies.

**Why this hurts:** Without health checks, the orchestrator has no way to know whether your application is actually functioning. It can't restart a stuck process or stop routing traffic to an instance that's lost its database connection. And a liveness probe that checks the database will cause unnecessary restarts when the database is temporarily down, which makes the situation worse, not better.

**What to do instead:** Implement separate liveness and readiness probes, as we describe in the [Deployment](./deployment.md) chapter. The liveness probe should be unconditional. The readiness probe should check dependencies.

## Over-engineering from the start

**What it looks like:** A new project with three workspace crates, trait-based repositories with generic services, a CQRS pattern, and an event bus, all for an application that has five CRUD endpoints.

**Why this hurts:** Premature abstraction adds complexity that slows you down without providing proportionate benefit. Every additional layer of indirection is another thing to understand, another place where bugs can hide, and another barrier to making changes. In my experience, the projects that ship fastest are the ones that start simple and add structure when they actually need it.

**What to do instead:** Start with the simplest structure that gives you clean separation, as we describe in the [Project Structure](./project-structure.md) chapter. Add abstraction when the complexity of the problem actually demands it, not in anticipation of complexity that may never arrive. You can always refactor from a flat structure to a layered one when the need becomes clear. Don't worry about getting the architecture "perfect" on day one.
