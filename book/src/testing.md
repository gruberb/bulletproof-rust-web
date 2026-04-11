# Testing

If you've ever shipped a web service and then spent the next week anxiously watching logs, you know that feeling. Testing is how we sleep at night. But testing a web application isn't just one thing. We need unit tests that verify our business logic in isolation, integration tests that exercise the full request-to-response pipeline, and a solid strategy for managing test databases and external dependencies. In this chapter, I'll walk through the patterns and tools that make testing an Axum application practical and, honestly, kind of enjoyable.

## The testing pyramid

You've probably heard of the testing pyramid before. It's a simple idea, but it holds up well in practice. At the base, we write lots of fast unit tests that verify individual functions and domain logic. In the middle, we have integration tests that exercise the full HTTP handler pipeline, including middleware, extractors, serialization, and database access. At the top, we keep a smaller number of end-to-end tests that verify critical user flows against a running server.

In my experience, most of your testing effort should go into the bottom two tiers. Unit tests catch logic bugs quickly and cheaply. Integration tests verify that all the layers are actually wired together correctly. End-to-end tests are useful for critical paths, but they're slower and more brittle, so I'd use them sparingly.

## Unit testing domain logic

This is where our earlier investment in clean architecture really pays off. Because the domain layer has no dependencies on Axum, SQLx, or any other framework, testing it is delightfully simple. We just construct domain types, call service methods, and assert on the results. No HTTP server, no database, no fuss.

If your services use trait-based repositories (as we set up in the [Architectural Patterns](./architecture.md) chapter), you can provide in-memory implementations for testing:

```rust
#[derive(Clone)]
struct InMemoryUserRepo {
    users: Arc<Mutex<Vec<User>>>,
}

impl InMemoryUserRepo {
    fn new() -> Self {
        Self {
            users: Arc::new(Mutex::new(Vec::new())),
        }
    }
}

impl UserRepository for InMemoryUserRepo {
    async fn create(&self, req: &CreateUserRequest) -> Result<User, CreateUserError> {
        let mut users = self.users.lock().await;

        // Check for duplicates
        if users.iter().any(|u| u.email() == &req.email) {
            return Err(CreateUserError::Duplicate { email: req.email.clone() });
        }

        let user = User::hydrate(
            UserId::new(),
            req.name.clone(),
            req.email.clone(),
            Utc::now(),
            Utc::now(),
        );

        users.push(user.clone());
        Ok(user)
    }

    async fn find_by_id(&self, id: UserId) -> anyhow::Result<Option<User>> {
        let users = self.users.lock().await;
        Ok(users.iter().find(|u| u.id() == &id).cloned())
    }

    // ... other methods
}

#[tokio::test]
async fn registering_a_user_returns_the_user() {
    let repo = InMemoryUserRepo::new();
    let service = UserService::new(repo);

    let name = UserName::parse("Alice").unwrap();
    let email = Email::parse("alice@example.com").unwrap();

    let user = service.register(name, email, "password123").await.unwrap();

    assert_eq!(user.name().as_str(), "Alice");
    assert_eq!(user.email().as_str(), "alice@example.com");
}

#[tokio::test]
async fn registering_duplicate_email_fails() {
    let repo = InMemoryUserRepo::new();
    let service = UserService::new(repo);

    let name = UserName::parse("Alice").unwrap();
    let email = Email::parse("alice@example.com").unwrap();

    service.register(name.clone(), email.clone(), "password123").await.unwrap();

    let result = service.register(name, email, "password456").await;
    assert!(matches!(result, Err(CreateUserError::Duplicate { .. })));
}
```

These tests run in milliseconds because they don't touch the network or the filesystem. They verify that your business logic is correct, independent of how it's invoked or where the data lives. That speed matters. When tests are fast, you actually run them.

If writing those in-memory implementations feels like a lot of boilerplate (and it can be), the `mockall` crate can generate mock implementations of your repository traits automatically. I tend to prefer hand-written fakes for core domain tests because they're easier to reason about, but `mockall` is a perfectly valid choice, especially as your trait surface grows.

## Integration testing with oneshot

Here's where things get really nice. Axum's integration with Tower means we can test our entire HTTP pipeline without ever starting a real server. The `tower::ServiceExt::oneshot` method sends a single request through the router and returns the response. It exercises all your middleware, extractors, serialization, and error handling, just like a real request would.

```rust
use axum::body::Body;
use http::Request;
use tower::ServiceExt;
use http_body_util::BodyExt;

/// Build the application the same way production does, but with a test database.
async fn test_app() -> Router {
    let pool = create_test_pool().await;
    create_app_with_pool(pool).await
}

#[tokio::test]
async fn health_check_returns_200() {
    let app = test_app().await;

    let response = app
        .oneshot(
            Request::builder()
                .uri("/health")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);
}

#[tokio::test]
async fn creating_a_user_returns_201() {
    let app = test_app().await;

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/v1/users")
                .header("Content-Type", "application/json")
                .body(Body::from(serde_json::to_string(&serde_json::json!({
                    "name": "Alice",
                    "email": "alice@example.com",
                    "password": "securepassword"
                })).unwrap()))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);

    let body = response.into_body().collect().await.unwrap().to_bytes();
    let user: serde_json::Value = serde_json::from_slice(&body).unwrap();
    assert_eq!(user["data"]["name"], "Alice");
    assert_eq!(user["data"]["email"], "alice@example.com");
}

#[tokio::test]
async fn creating_a_user_with_invalid_email_returns_400() {
    let app = test_app().await;

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/v1/users")
                .header("Content-Type", "application/json")
                .body(Body::from(r#"{"name":"Alice","email":"not-an-email","password":"securepassword"}"#))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::BAD_REQUEST);
}
```

The important thing here is that `test_app()` builds the router using the same `create_app` function (or something very close to it) that production uses. That means our integration tests exercise the same middleware stack, the same extractors, and the same error handling as production. If something breaks in how those pieces fit together, we'll catch it here.

Now, let's be honest about what these in-process tests don't cover. Because `oneshot` sends a request directly through the router without going through real sockets, we're not testing TLS termination, reverse proxy behavior, load balancer health checks, or anything else about your deployment topology. These tests are excellent for verifying application logic, but they're not a substitute for a smoke test against an actual running server in staging.

## Test database management

Integration tests that hit a real database need a strategy for isolation. You really don't want one test's leftover data interfering with another test's expectations. I've been bitten by this more times than I'd like to admit.

**Per-test databases with SQLx.** This is my go-to approach. SQLx provides a `#[sqlx::test]` attribute that creates a fresh, uniquely-named database for each test and runs your migrations against it. When the test finishes, the database is dropped. Perfect isolation, minimal effort:

```rust
#[sqlx::test(migrations = "./migrations")]
async fn test_user_creation(pool: PgPool) {
    let repo = PostgresUserRepo::new(pool);
    let req = CreateUserRequest { /* ... */ };

    let user = repo.create(&req).await.unwrap();
    assert_eq!(user.email().as_str(), "alice@example.com");
}
```

**Shared test database with cleanup.** An alternative is to use a single test database and clean up between tests, either by truncating tables or by wrapping each test in a transaction that you roll back. This is faster than creating and dropping databases, but it requires more careful management. You might wonder if the speed tradeoff is worth the complexity. In my experience, it usually isn't for most teams, but if your test suite takes minutes to run, it's worth considering.

**In-memory SQLite for fast tests.** If your SQL is compatible with SQLite (and it often is for simple CRUD operations), you can use SQLite in-memory databases for extremely fast test execution. This is particularly useful for repository-level tests where you want to verify query logic without the overhead of PostgreSQL.

## Test helpers

As your test suite grows, you'll start noticing the same setup code showing up everywhere. Building requests, creating test users, parsing response bodies. It might seem tedious at first, but taking the time to extract shared helpers makes a real difference. Here's a pattern I've found works well:

```rust
// tests/common/mod.rs

pub async fn test_app() -> Router {
    let pool = PgPool::connect(&test_database_url()).await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();
    create_app_with_pool(pool).await
}

pub fn test_database_url() -> String {
    std::env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| "postgres://localhost/myapp_test".to_string())
}

pub async fn create_test_user(app: &Router) -> (Uuid, String) {
    // Helper that creates a user and returns (user_id, auth_token)
    // Reduces boilerplate in tests that need an authenticated user
    // ...
}

pub async fn response_json(response: Response) -> serde_json::Value {
    let body = response.into_body().collect().await.unwrap().to_bytes();
    serde_json::from_slice(&body).unwrap()
}
```

## What to test

So what should you actually spend your time testing? Let's focus on what matters most.

**Test the happy path** for every endpoint. Verify that valid input produces the expected response with the correct status code and body shape. This is the baseline.

**Test validation failures.** Send invalid input and verify that the API returns appropriate error responses. This catches regressions where validation rules are accidentally removed or weakened, which happens more often than you'd think.

**Test authorization.** Verify that unauthenticated requests are rejected, that users can't access other users' data, and that role restrictions are enforced. Security bugs are the ones that keep you up at night.

**Test error mapping.** Verify that domain errors (like duplicate email) produce the right HTTP status codes and error messages, not generic 500 responses. We put a lot of work into our error types in the earlier chapters, so let's make sure they actually reach the client correctly.

**Test edge cases in domain logic.** Empty strings, very long strings, boundary values, unusual input combinations. Bugs love to hide here. These are best tested at the unit level, where the tests are fast and focused.

One last thing: don't obsess over code coverage as a metric. A test suite that thoughtfully covers the important behaviors and edge cases is far more valuable than one that chases 90% coverage by testing trivial getters and setters. What I've found is that coverage numbers make you feel good, but well-chosen tests are what actually catch bugs before they ship. With our testing strategy in place, let's move on to getting our application deployed and running in production.
