# Crate Reference

Throughout this book, we pull in quite a few crates. If you ever find yourself thinking "wait, which crate was that again?", this is the page to come back to. I've organized everything by concern so you can quickly find what you need and understand why it's in our stack.

## Core Framework

| Crate | Purpose | Notes |
|-------|---------|-------|
| [axum](https://crates.io/crates/axum) | HTTP routing, extractors, and handler framework | The foundation. Use with the `macros` feature for `#[debug_handler]`. |
| [tokio](https://crates.io/crates/tokio) | Async runtime | Use `features = ["full"]` unless you need to minimize dependencies. |
| [tower](https://crates.io/crates/tower) | Middleware abstractions and utilities | Provides `ServiceBuilder`, `ServiceExt` (for `oneshot` in tests), and composable layers. |
| [tower-http](https://crates.io/crates/tower-http) | HTTP-specific middleware | CORS, compression, tracing, timeouts, request IDs, body limits, and more. Enable features selectively. |
| [hyper](https://crates.io/crates/hyper) | HTTP implementation | You'll rarely interact with hyper directly when using Axum, but it's the HTTP engine under the hood. |

## Serialization and Data

| Crate | Purpose | Notes |
|-------|---------|-------|
| [serde](https://crates.io/crates/serde) | Serialization framework | Use `features = ["derive"]` for `#[derive(Serialize, Deserialize)]`. |
| [serde_json](https://crates.io/crates/serde_json) | JSON serialization | Axum's `Json` extractor uses this under the hood. |
| [uuid](https://crates.io/crates/uuid) | UUID generation and parsing | Use `features = ["v4", "serde"]` for random UUIDs with serde support. |
| [chrono](https://crates.io/crates/chrono) | Date and time handling | Use `features = ["serde"]` for serialization. Prefer `DateTime<Utc>` for timestamps. |

## Database

| Crate | Purpose | Notes |
|-------|---------|-------|
| [sqlx](https://crates.io/crates/sqlx) | Async SQL with compile-time checking | Our go-to for most Axum projects. Supports PostgreSQL, MySQL, and SQLite. |
| [diesel](https://crates.io/crates/diesel) | Type-safe ORM and query builder | Mature and well-tested. Use `diesel-async` for async support. |
| [sea-orm](https://crates.io/crates/sea-orm) | ActiveRecord-style async ORM | Worth considering if raw SQL feels like too much ceremony for your use case. |
| [axum-sqlx-tx](https://crates.io/crates/axum-sqlx-tx) | Request-scoped SQLx transactions | Begins a transaction for each request automatically, then commits or rolls back based on the response status. |

## Error Handling

| Crate | Purpose | Notes |
|-------|---------|-------|
| [thiserror](https://crates.io/crates/thiserror) | Derive macros for custom error types | We use this for domain errors and our `AppError` enum where we need to match on variants. |
| [anyhow](https://crates.io/crates/anyhow) | Flexible error type with context chaining | Great for the catch-all `Internal` variant and in infrastructure code. Don't be shy with `.context()`. |

## Validation

| Crate | Purpose | Notes |
|-------|---------|-------|
| [validator](https://crates.io/crates/validator) | Derive-based input validation | Supports `email`, `url`, `length`, `range`, and custom validators. |
| [garde](https://crates.io/crates/garde) | Alternative validation with const generics | A newer take on `validator` with a different API style. |
| [axum-valid](https://crates.io/crates/axum-valid) | Pre-built validated extractors for Axum | Integrates with `validator`, `garde`, and `validify`. Saves you from writing your own extractors. |

## Authentication and Security

| Crate | Purpose | Notes |
|-------|---------|-------|
| [jsonwebtoken](https://crates.io/crates/jsonwebtoken) | JWT encoding and decoding | The go-to for JWT-based auth in Rust. |
| [argon2](https://crates.io/crates/argon2) | Password hashing | What I'd recommend for password hashing today. Prefer it over bcrypt for new projects. |
| [axum-login](https://crates.io/crates/axum-login) | Session-based authentication | A good fit for traditional web apps with server-side sessions. |
| [axum-csrf-sync-pattern](https://crates.io/crates/axum-csrf-sync-pattern) | CSRF protection | Implements the OWASP Synchronizer Token Pattern. |
| [tower-governor](https://crates.io/crates/tower-governor) | Rate limiting | Per-IP rate limiting using the governor algorithm. |
| [tower-sessions](https://crates.io/crates/tower-sessions) | Session middleware | Server-side session management with pluggable backends. |

## Configuration

| Crate | Purpose | Notes |
|-------|---------|-------|
| [config](https://crates.io/crates/config) | Layered configuration loading | Supports files, environment variables, and multiple formats. |
| [dotenvy](https://crates.io/crates/dotenvy) | Load `.env` files | Fork of the original `dotenv` crate. Call `.ok()` instead of `.unwrap()` so it doesn't blow up in production. |
| [secrecy](https://crates.io/crates/secrecy) | Sensitive value protection | Wrap secrets in `SecretString`. It'll redact them in debug output and zero memory on drop. |

## Observability

| Crate | Purpose | Notes |
|-------|---------|-------|
| [tracing](https://crates.io/crates/tracing) | Structured logging and spans | If you're doing structured logging in Rust, this is what everyone reaches for. |
| [tracing-subscriber](https://crates.io/crates/tracing-subscriber) | Tracing output configuration | Use `features = ["env-filter", "json"]` for environment-based filtering and JSON output. |
| [tracing-opentelemetry](https://crates.io/crates/tracing-opentelemetry) | Bridge tracing to OpenTelemetry | Converts tracing spans into OpenTelemetry spans for distributed tracing. |
| [opentelemetry](https://crates.io/crates/opentelemetry) | OpenTelemetry API | Core API for distributed tracing and metrics. |
| [opentelemetry-otlp](https://crates.io/crates/opentelemetry-otlp) | OTLP exporter | Exports traces to Jaeger, Tempo, Datadog, and other OTLP-compatible backends. |
| [axum-tracing-opentelemetry](https://crates.io/crates/axum-tracing-opentelemetry) | Axum OpenTelemetry integration | Automatic trace context propagation for Axum. |

## API Documentation

| Crate | Purpose | Notes |
|-------|---------|-------|
| [utoipa](https://crates.io/crates/utoipa) | Compile-time OpenAPI spec generation | Generates OpenAPI 3.0 specs from code annotations. |
| [utoipa-swagger-ui](https://crates.io/crates/utoipa-swagger-ui) | Swagger UI for utoipa | Serves an interactive API documentation UI at a configurable path. |

## Testing

| Crate | Purpose | Notes |
|-------|---------|-------|
| tower (ServiceExt) | Integration testing without a server | Use `oneshot()` to send requests through the router directly. |
| [mockall](https://crates.io/crates/mockall) | Automatic mock generation | Generates mock implementations of your traits for unit testing. |
| [axum-test](https://crates.io/crates/axum-test) | Testing library for Axum | Gives you a `TestClient` with a nicer API than raw `oneshot`. |

## HTTP Client

| Crate | Purpose | Notes |
|-------|---------|-------|
| [reqwest](https://crates.io/crates/reqwest) | Async HTTP client | The HTTP client you'll see in most Rust projects. Async-native with connection pooling. |

## gRPC

| Crate | Purpose | Notes |
|-------|---------|-------|
| [tonic](https://crates.io/crates/tonic) | gRPC framework | Native async gRPC client and server. Integrates with Tower middleware. |
| [tonic-prost](https://crates.io/crates/tonic-prost) | Tonic + Prost runtime | Runtime support for tonic services using prost-generated types. |
| [prost](https://crates.io/crates/prost) | Protocol Buffers | Protobuf serialization/deserialization. Used by tonic for message types. |
| [tonic-prost-build](https://crates.io/crates/tonic-prost-build) | Proto code generation | Compiles `.proto` files into Rust code at build time. Goes in `[build-dependencies]`. This replaced direct `tonic-build` usage as of tonic 0.14. |
| [tonic-reflection](https://crates.io/crates/tonic-reflection) | gRPC server reflection | Lets `grpcurl` and other tools discover your service's API. |

## Async Utilities

| Crate | Purpose | Notes |
|-------|---------|-------|
| [tokio-util](https://crates.io/crates/tokio-util) | Extended Tokio utilities | Provides `CancellationToken` for structured shutdown, plus codecs and other helpers. |
| [futures](https://crates.io/crates/futures) | Future combinators | Provides `FuturesUnordered`, `StreamExt`, and other utilities for working with async streams. |

## Background Jobs

| Crate | Purpose | Notes |
|-------|---------|-------|
| [apalis](https://crates.io/crates/apalis) | Background task processing | Type-safe, extensible, supports PostgreSQL/Redis/SQLite backends. Built-in monitoring and graceful shutdown. |
| [fang](https://crates.io/crates/fang) | Persistent job queue | Workers in separate Tokio tasks with auto-restart on panic. Cron scheduling and retry backoff. |
| [backie](https://crates.io/crates/backie) | Async task queue | PostgreSQL-backed, designed for horizontal scaling across multiple processes. |
