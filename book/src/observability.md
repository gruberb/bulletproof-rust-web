# Observability

You've probably had that moment where something breaks in production and you're staring at a wall of unhelpful log lines, trying to piece together what happened. I certainly have. Observability is how we avoid that situation. It's how we know what our application is actually doing once it's running out in the world, without having to reproduce the problem on our laptop.

The Rust ecosystem has converged on the `tracing` crate as the standard for structured, contextual logging. When you combine it with OpenTelemetry for distributed tracing and metrics, you get a solid observability stack that covers most of what we'll need.

## Structured logging with tracing

If you've used `log` or `env_logger` before, `tracing` will feel familiar at first, but it's doing something quite different under the hood. Instead of emitting flat text strings, it emits structured events with typed fields. That means your log analysis tool can actually filter, aggregate, and query on specific fields rather than trying to parse them out of a message string. It also supports spans, which represent a period of time (like the duration of an HTTP request or a database query) and carry context that gets automatically attached to all events within them.

### Setting up the subscriber

The tracing subscriber controls how events are formatted and where they go. In development, we want human-readable output with colors so we can actually read what's happening in our terminal. In production, we want JSON output that our log aggregation system can parse.

```rust
use tracing_subscriber::{
    fmt, layer::SubscriberExt, util::SubscriberInitExt, EnvFilter,
};

pub fn init_tracing(log_level: &str) {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(log_level));

    // Use runtime configuration rather than build profile to control
    // log format. A release build in staging should still be readable,
    // and a debug build against production data should still emit JSON.
    let json_output = std::env::var("LOG_JSON")
        .map(|v| v == "true" || v == "1")
        .unwrap_or(false);

    let fmt_layer = if json_output {
        fmt::layer().json().boxed()
    } else {
        fmt::layer().pretty().boxed()
    };

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt_layer)
        .init();
}
```

The `EnvFilter` lets us control log levels at runtime through the `RUST_LOG` environment variable. You can set different levels for different modules, which is really helpful when you need to turn up verbosity on one specific component without drowning in logs from everything else:

```
RUST_LOG=info,my_app::infra=debug,sqlx=warn
```

### Request tracing with TraceLayer

The `TraceLayer` from tower-http automatically creates a span for each HTTP request, recording the method, URI, status code, and duration. When we combine this with a request ID, we can trace a single request through all the log entries it generates. Let's see how that looks.

```rust
use tower_http::trace::TraceLayer;

let trace_layer = TraceLayer::new_for_http()
    .make_span_with(|request: &http::Request<_>| {
        let request_id = request
            .headers()
            .get("x-request-id")
            .and_then(|v| v.to_str().ok())
            .unwrap_or("unknown");

        // Prefer the matched route pattern (e.g., "/api/v1/users/{id}")
        // over the raw URI (e.g., "/api/v1/users/550e8400-...").
        // Raw URIs create high-cardinality span fields that can overwhelm
        // metrics backends and make aggregation impossible.
        let matched_path = request.extensions()
            .get::<axum::extract::MatchedPath>()
            .map(|p| p.as_str().to_string())
            .unwrap_or_else(|| request.uri().path().to_string());

        tracing::info_span!(
            "http_request",
            method = %request.method(),
            path = %matched_path,
            request_id = %request_id,
        )
    });
```

Every log event emitted while processing that request will automatically include the `method`, `path`, and `request_id` fields, because the span acts as ambient context for all work done within it. This is one of those things that sounds small but makes a huge difference. When our application is handling hundreds of concurrent requests, we can still find every log entry related to one specific request just by filtering on its ID.

### Instrumenting handlers and services

We can use the `#[tracing::instrument]` attribute to automatically create spans for individual functions. I find this particularly valuable on service methods and repository calls, because it lets you see exactly where time is being spent without manually wiring up span creation everywhere.

```rust
#[tracing::instrument(
    skip(state, payload),
)]
async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    tracing::info!("creating new user");

    let user = state.user_service.register(payload).await?;

    tracing::info!(user_id = %user.id().as_uuid(), "user created successfully");
    Ok((StatusCode::CREATED, Json(user.into())))
}
```

The `skip(state, payload)` directive tells the instrument macro not to include the state or the request payload in the span's fields. After the service call succeeds, we log the `user_id` as a structured field. Notice that we use the opaque `user_id` rather than `user.email` in the span. This is intentional. In my experience, it's much better to stick with low-PII identifiers (user IDs, tenant IDs, request IDs) over personal data (email addresses, names) in your tracing output. If you need to correlate a trace with a real email for debugging, look it up from the user ID. You don't want every log line in your aggregation system carrying personal data around.

### Logging best practices

**Use structured fields instead of string interpolation.** Rather than `tracing::info!("Created user {}", user_id)`, write `tracing::info!(user_id = %user_id, "user created")`. The structured form lets your log aggregation system index and filter on the `user_id` field, which is way more useful than trying to parse it out of a text string.

**Log at appropriate levels.** This might seem obvious, but it's worth spelling out. Use `error` for things that indicate a bug or a system failure that needs attention. Use `warn` for situations that are unusual but handled (like a rate-limited request or a cache miss). Use `info` for significant business events (user created, order placed). Use `debug` for detailed operational information that helps during development or troubleshooting. Getting this right matters because when something goes wrong at 2 AM, you want your alerts to mean something.

**Never log secrets.** If you're using the `secrecy` crate for your configuration (as we covered in the [Configuration](./configuration.md) chapter), the `Debug` implementation will automatically redact sensitive values. But be careful with request bodies, headers, and other data that might contain tokens, passwords, or personal information.

## OpenTelemetry

Once our application grows into a distributed system where a single user request might touch multiple services, we need a way to follow that request across boundaries. That's where OpenTelemetry comes in. It gives us a standard way to propagate trace context and collect telemetry data.

The key crates are:

- **opentelemetry** and **opentelemetry_sdk** for the core API and SDK
- **opentelemetry-otlp** for exporting traces to an OTLP-compatible backend (Jaeger, Tempo, Datadog, etc.)
- **tracing-opentelemetry** for bridging the `tracing` crate's spans to OpenTelemetry spans
- **axum-tracing-opentelemetry** for automatic trace context propagation in Axum

With this stack, each request gets a trace ID that follows it across service boundaries. When a user reports a problem, we can look up the trace by ID and see the whole picture: which services were involved, how long each step took, and where things went wrong.

## Metrics

Metrics give us a different lens than logs and traces. Where a log entry tells us about one specific request, metrics give us aggregate, time-series data about how our application is behaving overall. They answer questions like: is our 99th percentile latency creeping up? Has the error rate doubled in the last five minutes? Is the database connection pool running hot?

- **axum-otel-metrics** provides OpenTelemetry metrics with Prometheus export
- **axum-prometheus** provides HTTP metrics compatible with the metrics.rs ecosystem

A typical setup exposes a `/metrics` endpoint that Prometheus scrapes at regular intervals. In my experience, the most useful metrics for a web application are request count (by method, path, and status code), request duration histograms, and error rates. You might wonder if you need all three from day one. Honestly, even just request duration and error rates will catch most problems before your users notice them.

## Bringing it together

A production-ready observability setup combines all three of these. Structured logs give us detailed information about individual events. Traces let us follow a request through multiple services and find bottlenecks. And metrics give us the big picture of system health and performance trends.

Setting all this up might seem like a lot of work upfront, and it is some work. But it pays for itself the first time you need to debug a production issue. Instead of guessing what went wrong and deploying speculative fixes, you look at the data and see what actually happened. That's a much better place to be.

With our observability foundation in place, let's look at how we handle errors and communicate them clearly to our API consumers.
