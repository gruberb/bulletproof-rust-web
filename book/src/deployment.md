# Deployment

So we've built our web service, tested it, and it runs great on our laptop. Now what? Getting it into production involves a few things we need to get right: building a small container image, shutting down gracefully so we don't drop requests that are still in flight, and giving our orchestrator health check endpoints so it knows what's going on with our app. Let's walk through each of these.

## Docker and multi-stage builds

If you compile Rust for the default GNU Linux target (`x86_64-unknown-linux-gnu`), your binary is dynamically linked against glibc but doesn't have any other runtime dependencies. You can also compile with the musl target (`x86_64-unknown-linux-musl`) for a fully static binary. Either way, our production container doesn't need the Rust toolchain, source code, or any build dependencies beyond the C library. That's great news, because it means we can use a multi-stage Docker build: compile in one stage, then copy just the binary into a tiny runtime image.

```dockerfile
# Build stage
FROM rust:1-slim AS builder

WORKDIR /app

# Cache dependencies by copying just the manifest files first.
# For more robust dependency caching, consider the cargo-chef crate,
# which is purpose-built for this problem in Docker builds.
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release && rm -rf src

# Now copy the actual source and rebuild (only changed files recompile)
COPY . .
RUN touch src/main.rs && cargo build --release

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Run as a non-root user
RUN useradd --create-home appuser
USER appuser

COPY --from=builder /app/target/release/my-app /usr/local/bin/my-app

EXPOSE 3000

CMD ["my-app"]
```

The dependency caching trick in the build stage is worth understanding. We copy only `Cargo.toml` and `Cargo.lock` first and run a build, which lets Docker cache the compiled dependencies as a layer. When we change our application code but not our dependencies (which is what happens most of the time), Docker reuses that cached layer and only recompiles our code. Without this, you'd be recompiling every dependency on every build, and Rust compile times being what they are, that gets painful fast.

You might wonder why we install `ca-certificates` in the runtime image. It's because our application probably makes outbound HTTPS requests (to external APIs, payment providers, and so on) and needs the CA certificate bundle to validate TLS connections. If you skip this, you'll get cryptic TLS errors at runtime, which is never fun to debug in production.

If you want to go even smaller, you can compile with `x86_64-unknown-linux-musl` to produce a fully statically linked binary, then use `FROM scratch` as your runtime base. This gives you images under 20 MB, which is impressive. The trade-off is that you have no shell or debugging tools in the container at all, which can make troubleshooting a lot harder when something goes wrong at 2 AM. In my experience, the Debian slim image is a good middle ground for most teams.

## Graceful shutdown

When a container orchestrator like Kubernetes, ECS, or Docker Swarm decides to stop our application, it sends a SIGTERM signal and gives the process a grace period (typically 30 seconds) to finish what it's doing before sending SIGKILL. If our application doesn't handle SIGTERM, bad things happen: in-flight requests get dropped, database transactions are left in an indeterminate state, and clients see connection resets. You really don't want that.

The good news is that Axum gives us a built-in mechanism for graceful shutdown:

```rust
use tokio::signal;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // ... setup code ...

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    tracing::info!("listening on {}", listener.local_addr()?);

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    tracing::info!("server shut down cleanly");
    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    tracing::info!("shutdown signal received, draining connections");
}
```

When the shutdown signal fires, Axum stops accepting new connections but keeps processing requests that are already in flight. Once all active requests finish (or the grace period expires), the server exits cleanly.

This matters a lot for database operations. If a request is in the middle of a transaction when SIGKILL arrives, the database will eventually roll back the transaction after its connection timeout, but the client sees an error. With graceful shutdown, the transaction completes normally and the client gets a proper response. It's one of those things that might seem like a minor detail, but it makes a real difference in production.

## Health checks

Health check endpoints let our container orchestrator or load balancer know whether the application is alive and ready to serve traffic. Most orchestrators distinguish between two types of probes, and it's worth understanding the difference because getting this wrong can cause some really confusing behavior.

**Liveness probes** answer the question "is the process alive and not stuck?" If the liveness probe fails, the orchestrator restarts the container. This probe should be simple and fast. Here's the important part: don't check external dependencies in your liveness probe. If your database is temporarily unavailable, you don't want the orchestrator to kill and restart your app, because that just adds more load to an already stressed database.

```rust
async fn health_live() -> StatusCode {
    StatusCode::OK
}
```

**Readiness probes** answer a different question: "is this instance ready to receive traffic?" If the readiness probe fails, the orchestrator stops routing traffic to this instance but doesn't restart it. This is where we check that the database connection is healthy, that any required caches are reachable, and that the application has finished its startup initialization.

```rust
async fn health_ready(State(state): State<AppState>) -> StatusCode {
    let db_ok = sqlx::query("SELECT 1")
        .execute(&state.db)
        .await
        .is_ok();

    if db_ok {
        StatusCode::OK
    } else {
        StatusCode::SERVICE_UNAVAILABLE
    }
}
```

We mount these endpoints outside our versioned API routes:

```rust
let app = Router::new()
    .route("/health", get(health_live))
    .route("/health/ready", get(health_ready))
    .nest("/api/v1", api_routes())
    .with_state(state);
```

In a Kubernetes deployment, you'd configure the probes in your pod spec like this:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Environment-based configuration

As we covered in the [Configuration](./configuration.md) chapter, our application reads all its configuration from environment variables. This means the same Docker image can be deployed to development, staging, and production. The only thing that changes between environments is the set of environment variables the deployment platform provides.

Don't bake environment-specific configuration into the Docker image. I've seen teams do this and it always leads to trouble. The image should be an immutable artifact that you build once and deploy everywhere. When you promote a tested image from staging to production, you want to know with confidence that the binary is identical.

## Running migrations

Our database migrations should run as part of the application startup, before the server begins accepting traffic. We handle this with the `sqlx::migrate!` macro as described in the [Database Layer](./database.md) chapter.

In a Kubernetes environment, you could alternatively run migrations as an init container or a pre-deployment job. That approach is more explicit and lets you separate migration failures from application startup failures. But for most applications, what I've found is that running migrations at startup is simpler and works well, especially when you combine it with the readiness probe. The readiness probe ensures our application doesn't receive traffic until migrations are complete and the database is accessible, which gives us a nice safety net.

## What about Shuttle?

[Shuttle](https://www.shuttle.dev/) is a deployment platform built specifically for Rust. It handles infrastructure provisioning, database setup, and deployment with minimal configuration. If you don't need fine-grained control over your deployment infrastructure, it can take a lot of the operational burden off your shoulders.

That said, I think understanding how to deploy with Docker and Kubernetes is valuable regardless of which platform you end up using. The underlying concepts we covered here (graceful shutdown, health checks, environment-based configuration, immutable images) apply everywhere. Even if you use Shuttle or a similar platform, knowing what's happening under the hood will help you debug issues when they come up.
