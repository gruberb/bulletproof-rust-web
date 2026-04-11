# Database Layer

If you've ever worked on a project where SQL queries were scattered across handler functions, you know how quickly that becomes painful. At first it feels productive, you're shipping features, things work. But a few months in, you're hunting across dozens of files to find where a column name needs updating, and your tests require a running database for everything. I've been there, and it's not fun.

In this chapter, we'll build a clean, well-separated database layer using SQLx, which is the most common choice for Axum applications. The principles apply no matter which database crate you end up using, though.

## Connection pool setup

You don't want every incoming request to open a fresh database connection. That means a TCP handshake, TLS negotiation, and possibly authentication, all before you've even run your query. Instead, we maintain a pool of pre-established connections that requests can borrow from and return to.

```rust
use anyhow::Context;
use sqlx::postgres::{PgPool, PgPoolOptions};
use std::time::Duration;

pub async fn create_pool(database_url: &str) -> anyhow::Result<PgPool> {
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .min_connections(2)
        .acquire_timeout(Duration::from_secs(3))
        .idle_timeout(Duration::from_secs(600))
        .connect(database_url)
        .await
        .context("failed to connect to the database")?;

    Ok(pool)
}
```

How many connections should you set? It depends on your concurrency and what the database server can handle. PostgreSQL defaults to 100 maximum connections, and each one takes up memory on the server side. A decent starting point is 2 to 4 connections per CPU core on your application server, then adjust from there based on how much time your requests actually spend waiting on the database versus doing other work. On a typical 4-core instance, that's 8 to 16 connections. If you're running multiple instances behind a load balancer, don't forget that the total connections across all of them still has to stay under the database's limit.

## Migrations

Our migrations should be versioned, reproducible, and ideally run automatically at startup. SQLx gives us a migration system that reads `.sql` files from a directory and applies them in order.

```
migrations/
├── 20240101000000_create_users.sql
├── 20240102000000_create_posts.sql
└── 20240103000000_add_user_bio.sql
```

What does a well-written migration look like? I try to include sensible defaults for new columns, use UUIDs instead of auto-incrementing integers for primary keys (which makes ID guessing harder, though authorization is still what actually protects your resources), and store timestamps with timezone information:

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    name        VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    bio         TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- Automatically update the updated_at timestamp on row modification
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

Why a database trigger for `updated_at`? Because it keeps things consistent no matter which code path modifies the row. Even if someone applies a manual SQL fix directly to the database (and believe me, that happens), the timestamp still gets updated correctly.

### Schema craftsmanship

The migration above is a starting point. There are a few more techniques that are easy to add now in migrations and really annoying to retrofit later. Let me walk through the ones I've found most useful.

**Case-insensitive uniqueness.** If usernames and emails should be unique regardless of casing, enforce that at the database level. PostgreSQL's `citext` extension or a `COLLATE "und-ci-ai-ks-kc"` ICU collation prevents "Alice" and "alice" from both being accepted. I've seen bugs from relying on application-level lowercasing, because direct SQL inserts or other services can bypass it entirely.

**Reusable trigger helpers.** If multiple tables need `updated_at` behavior, extract the trigger function once and reuse it across tables. That's exactly what the `update_updated_at_column()` function above does. Just apply the same trigger to every table that has an `updated_at` column.

**Appropriate index types.** For array columns like tags, a GIN index enables fast containment queries (`@>`, `&&`). For text search columns, consider `tsvector` with a GIN index. Standard B-tree indexes are the right choice for equality and range queries on scalar columns.

**Deliberate ID type choices.** UUIDs are what I recommend by default in this guide, but there are situations where sequential IDs (bigserial) work better, like external specs that expect integers or high-write tables where UUID randomness causes B-tree page splits. UUIDv7 is worth a look for write-heavy tables where you want both uniqueness and index-friendly ordering, since it embeds a timestamp and sorts chronologically. PostgreSQL 18+ supports `uuidv7()` natively. For PostgreSQL 17 and earlier, you can generate UUIDv7 values in the application layer using the `uuid` crate with the `v7` feature, or use a database extension.

If you want to see these patterns applied to a real application, the [launchbadge/realworld-axum-sqlx](https://github.com/launchbadge/realworld-axum-sqlx) repository has great migration commentary worth reading through.

### Running migrations

I like to run migrations at application startup:

```rust
sqlx::migrate!("./migrations")
    .run(&pool)
    .await
    .context("failed to run database migrations")?;
```

The `migrate!` macro embeds the migration files into your binary at compile time, so you don't need to ship the migration directory alongside your application. One less thing to think about during deployment.

### Compile-time query checking in CI

Here's something that might trip you up: SQLx's `query!` and `query_as!` macros validate your SQL against a real database at compile time. That means your CI environment needs database access during builds. There are two ways to handle this.

The approach I'd recommend for CI is **offline mode**. You run `cargo sqlx prepare` locally (which connects to a running database and generates a `.sqlx/` directory with cached query metadata), then check that directory into version control. In CI, set `SQLX_OFFLINE=true` and the macros will use the cached metadata instead of connecting to a database. Add `cargo sqlx prepare --check` as a CI step to verify that the cached metadata stays in sync with your queries and migrations.

The alternative is to spin up a database in CI (via Docker or a CI service) and set `DATABASE_URL` so the macros can connect directly. It's simpler to set up but slower and more fragile.

### A note on migrations at startup

Running migrations at startup works well for single-instance deployments and during development. But if you're running multiple instances in production, you should know that all of them will try to run migrations at the same time on startup. SQLx handles this with an advisory lock, so only one instance actually runs them, but in some scenarios (particularly with rolling deployments) a dedicated migration job or Kubernetes init container gives you more control and clearer error handling. We'll dig into this more in the [Deployment](./deployment.md) chapter.

## The repository pattern

This is one of my favorite patterns for keeping database logic organized. A repository is just a struct that owns all database access for a particular domain concept. It holds a reference to the connection pool, and its methods map to the things you actually need to do: creating records, finding them by various criteria, updating, and deleting.

```rust
#[derive(Clone)]
pub struct UserRepository {
    pool: PgPool,
}

impl UserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    pub async fn create(&self, req: &CreateUserRequest) -> Result<User, CreateUserError> {
        let row = sqlx::query_as!(
            UserRow,
            r#"
            INSERT INTO users (id, email, name, password_hash)
            VALUES ($1, $2, $3, $4)
            RETURNING id, email, name, password_hash, bio, created_at, updated_at
            "#,
            Uuid::new_v4(),
            req.email.as_str(),
            req.name.as_str(),
            &req.password_hash,
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| match e {
            sqlx::Error::Database(ref db_err) if db_err.is_unique_violation() => {
                CreateUserError::Duplicate { email: req.email.clone() }
            }
            other => CreateUserError::Unknown(other.into()),
        })?;

        row.try_into()
            .map_err(|e: anyhow::Error| CreateUserError::Unknown(e))
    }

    pub async fn find_by_id(&self, id: &UserId) -> anyhow::Result<Option<User>> {
        let row = sqlx::query_as!(
            UserRow,
            r#"
            SELECT id, email, name, password_hash, bio, created_at, updated_at
            FROM users
            WHERE id = $1
            "#,
            id.as_uuid(),
        )
        .fetch_optional(&self.pool)
        .await
        .context("failed to fetch user by id")?;

        row.map(TryInto::try_into).transpose()
    }

    pub async fn find_by_email(&self, email: &Email) -> anyhow::Result<Option<User>> {
        let row = sqlx::query_as!(
            UserRow,
            r#"
            SELECT id, email, name, password_hash, bio, created_at, updated_at
            FROM users
            WHERE email = $1
            "#,
            email.as_str(),
        )
        .fetch_optional(&self.pool)
        .await
        .context("failed to fetch user by email")?;

        row.map(TryInto::try_into).transpose()
    }

    pub async fn update(
        &self,
        id: &UserId,
        name: Option<&UserName>,
        bio: Option<&str>,
    ) -> anyhow::Result<Option<User>> {
        let row = sqlx::query_as!(
            UserRow,
            r#"
            UPDATE users
            SET name = COALESCE($2, name),
                bio = COALESCE($3, bio)
            WHERE id = $1
            RETURNING id, email, name, password_hash, bio, created_at, updated_at
            "#,
            id.as_uuid(),
            name.map(|n| n.as_str()),
            bio,
        )
        .fetch_optional(&self.pool)
        .await
        .context("failed to update user")?;

        row.map(TryInto::try_into).transpose()
    }
}
```

Let me highlight a few patterns worth noticing here.

**Compile-time checked queries.** The `query_as!` macro verifies your SQL against the actual database schema at compile time and generates its own type mapping from the query result columns to the struct fields. It doesn't use the `FromRow` trait; the mapping is built directly by the macro based on column names and types. If you rename a column in a migration but forget to update the query, the build fails. I find this to be one of SQLx's most useful features. (If you prefer the non-macro `query_as::<_, UserRow>(sql)` function, that one does use `FromRow` at runtime.)

**Separate row types.** The `UserRow` struct maps directly to the database columns. Deriving `sqlx::FromRow` isn't required for `query_as!`, but it's harmless to include and useful if you also use the non-macro query functions elsewhere. The `User` domain type has newtypes for its fields and possibly different field names. The `TryFrom<UserRow> for User` conversion bridges the gap, as we covered in the [Domain Modeling](./domain-modeling.md) chapter. We use `TryFrom` rather than `From` because stored data might not satisfy current domain invariants.

**`fetch_optional` for lookups.** When querying by ID or email, reach for `fetch_optional` instead of `fetch_one`. A missing record is a normal case, not an error, and returning `Option<User>` lets the calling code decide whether "not found" is actually a problem in that particular context.

**COALESCE for partial updates.** The `COALESCE` SQL function keeps existing values when the update parameter is `NULL`, which lets you implement partial updates without needing to read the current values first. There's a limitation worth knowing about, though: this pattern can't distinguish between "the client sent null to clear this field" and "the client didn't include this field." Both arrive as `None` at the Rust level, and both get treated as "keep the existing value." If your API needs to support explicitly clearing optional fields, you'll need either a sentinel value, a separate "fields to clear" parameter, or a more explicit SQL construction.

**Parameterized queries everywhere.** Every user-supplied value goes through a `.bind()` parameter placeholder (`$1`, `$2`, etc.), never through string interpolation. This prevents SQL injection by design.

## The FromRef pattern for ergonomic access

You might find yourself extracting the full `AppState` in every handler just to reach the database pool or a repository. That gets tedious fast. The `FromRef` pattern offers a cleaner way, letting you extract sub-components of your state directly.

```rust
use axum::extract::FromRef;

#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub user_repo: UserRepository,
    // ... other fields
}

impl FromRef<AppState> for UserRepository {
    fn from_ref(state: &AppState) -> Self {
        state.user_repo.clone()
    }
}

impl FromRef<AppState> for PgPool {
    fn from_ref(state: &AppState) -> Self {
        state.db.clone()
    }
}
```

Now our handlers can extract just the repository they need:

```rust
async fn get_user(
    State(repo): State<UserRepository>,
    Path(id): Path<Uuid>,
) -> AppResult<Json<UserResponse>> {
    let user = repo.find_by_id(&UserId::from_uuid(id))
        .await?
        .ok_or(AppError::NotFound)?;
    Ok(Json(user.into()))
}
```

I like this because it keeps handler signatures focused on what the handler actually uses, instead of pulling in the entire application state every time.

## Trait-based repositories for testability

For simpler cases, a concrete `UserRepository` struct with SQLx calls works perfectly fine. But if you want to unit-test your service layer without spinning up a database, you can define the repository as a trait and provide both a real implementation and a test one.

We'll cover this in detail in the [Architectural Patterns](./architecture.md) chapter. The short version: define a trait in your domain layer, implement it with SQLx in your infrastructure layer, and use generics or trait objects in your service layer. Your tests can then provide a simple in-memory implementation that just stores data in a `HashMap`.

## Transactions

When you have multiple database writes that need to succeed or fail together, wrap them in a transaction:

```rust
pub async fn transfer_credits(
    &self,
    from: &UserId,
    to: &UserId,
    amount: i64,
) -> anyhow::Result<()> {
    let mut tx = self.pool.begin().await.context("failed to begin transaction")?;

    sqlx::query!(
        "UPDATE accounts SET credits = credits - $1 WHERE user_id = $2",
        amount,
        from.as_uuid(),
    )
    .execute(&mut *tx)
    .await
    .context("failed to debit source account")?;

    sqlx::query!(
        "UPDATE accounts SET credits = credits + $1 WHERE user_id = $2",
        amount,
        to.as_uuid(),
    )
    .execute(&mut *tx)
    .await
    .context("failed to credit target account")?;

    tx.commit().await.context("failed to commit transfer")?;

    Ok(())
}
```

If any step fails (or the function returns early via `?`), the transaction is automatically rolled back when the `tx` variable is dropped. No cleanup code needed.

If you want the entire handler to run inside a single transaction that commits on success and rolls back on error, take a look at the `axum-sqlx-tx` crate. It provides an extractor that handles this for you, which is really convenient for request-scoped transactions.

## Choosing a database crate

You might wonder which database crate to use. The Rust ecosystem has several solid options, each with different trade-offs.

**SQLx** is what most people reach for with Axum. It's async-native, supports compile-time query checking, and works with raw SQL. You write the SQL yourself, which gives you full control and avoids the impedance mismatch that ORMs sometimes introduce. The compile-time checking catches column type mismatches, missing columns, and syntax errors before your code ever runs.

**Diesel** is the oldest and most mature Rust ORM. It gives you a type-safe query builder that catches errors at compile time through the type system rather than by connecting to the database. Diesel was traditionally synchronous, but the `diesel-async` crate provides async support. It's a good choice if you prefer a query builder over raw SQL and want strong compile-time guarantees.

**SeaORM** takes an ActiveRecord-inspired approach with async support built in. It generates entity types from your database schema and provides a high-level query API. It's well-suited for rapid development and projects where writing raw SQL feels like overkill, though you get less fine-grained control than with SQLx.

For most new Axum projects, I'd go with SQLx as the default. It's a natural fit with Axum's philosophy of being explicit and composable rather than magical, and the compile-time query checking becomes a real productivity boost once you get used to it.
