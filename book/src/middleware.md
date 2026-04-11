# Middleware

Every web application ends up with a bunch of logic that doesn't belong in any single handler but needs to run on many (or all) requests. Logging, authentication, compression, rate limiting, request IDs, timeouts. You know the list. In Axum, we handle all of this through middleware, and because Axum's middleware system is built on Tower, we get access to a huge ecosystem of pre-built components plus a clean way to compose our own.

The thing that trips people up most often isn't writing middleware. It's getting the ordering right. Let's start there.

## The Tower layer model

Tower thinks of middleware as "layers" that wrap a service. When a request comes in, it passes through each layer in order, from the outermost to the innermost (which is your handler). The response then travels back out through the layers in reverse. So the first layer you add is the first to see the request and the last to see the response.

```
Request → Compression → Tracing → Timeout → Auth → Handler
Response ← Compression ← Tracing ← Timeout ← Auth ← Handler
```

This ordering matters more than you might expect. If you want tracing to record how long the entire request took, including time spent in authentication, the tracing layer has to be outside the auth layer. Get it backwards and your timing data will be wrong in subtle ways that are annoying to debug.

## A recommended middleware stack

Let's look at a middleware stack I'd reach for in a production application, built with `ServiceBuilder`:

```rust
use tower::ServiceBuilder;
use tower_http::{
    compression::CompressionLayer,
    cors::CorsLayer,
    limit::RequestBodyLimitLayer,
    request_id::{MakeRequestUuid, SetRequestIdLayer, PropagateRequestIdLayer},
    timeout::TimeoutLayer,
    trace::TraceLayer,
};
use std::time::Duration;

let app = Router::new()
    .merge(api_routes())
    .layer(
        ServiceBuilder::new()
            // Layers execute top-to-bottom on the request path.
            .layer(CompressionLayer::new())
            .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
            .layer(TraceLayer::new_for_http())
            .layer(PropagateRequestIdLayer::x_request_id())
            .layer(TimeoutLayer::new(Duration::from_secs(30)))
            .layer(RequestBodyLimitLayer::new(1024 * 1024)) // 1 MB
            .layer(cors_layer())
    )
    .with_state(state);
```

Let's walk through what each layer does and why it sits where it does.

**CompressionLayer** compresses response bodies using gzip (or brotli, or deflate, depending on what the client supports) and sets the `Content-Encoding` header. We put it at the outermost position so it compresses the final response body after all inner layers have finished producing it. Worth noting: compression applies to the body only, not to HTTP headers.

**SetRequestIdLayer** generates a unique UUID and attaches it to the incoming request. We put it before `TraceLayer` so the request ID is already available when the tracing span gets created.

**TraceLayer** records structured log entries for each request, including the method, path, status code, and duration. Because it runs after the request ID is set, our tracing span can include the request ID, which is incredibly useful when you're digging through logs later.

**PropagateRequestIdLayer** copies the request ID onto the outgoing response. It sits after `TraceLayer` so the response header is set before tracing finalizes the response side of the span. This ordering (set, then trace, then propagate) follows tower-http's own documentation and ensures that request IDs show up consistently in both request logs and response headers.

**TimeoutLayer** cancels requests that take longer than the specified duration. This protects against slow clients, runaway queries, and other situations where a request just hangs forever.

**RequestBodyLimitLayer** rejects request bodies that exceed a size limit. It's a basic defense against denial-of-service attacks where a client sends an enormous payload.

**CorsLayer** handles Cross-Origin Resource Sharing headers. Its exact position in the stack is less critical than the others, but it needs to be applied to the routes that serve your API.

## CORS configuration

CORS deserves its own section because misconfiguring it is one of those things that will have you pulling your hair out. You know the symptom: requests work perfectly from Postman but fail mysteriously from the browser. And on the flip side, allowing all origins in production is a real security hole.

```rust
use tower_http::cors::CorsLayer;
use http::{Method, header};

fn cors_layer(config: &Config) -> CorsLayer {
    if config.environment == Environment::Development {
        CorsLayer::permissive()
    } else {
        let origins: Vec<HeaderValue> = config.cors_origins
            .iter()
            .map(|o| o.parse().expect("invalid CORS origin in config"))
            .collect();

        CorsLayer::new()
            .allow_origin(origins)
            .allow_methods([
                Method::GET,
                Method::POST,
                Method::PUT,
                Method::DELETE,
            ])
            .allow_headers([
                header::CONTENT_TYPE,
                header::AUTHORIZATION,
            ])
            .allow_credentials(true)
    }
}
```

One thing I want to call out: we're driving the CORS policy from runtime configuration, not from the build profile. Using `cfg!(debug_assertions)` would be wrong here. Think about it: a release build deployed to a staging environment should still have relaxed CORS, and a debug build running against production data should still be restrictive. The environment and allowed origins come from the `Config` struct we set up in the [Configuration](./configuration.md) chapter.

## Writing custom middleware with `from_fn`

Most of the time, you don't need middleware that's reusable across projects. You just need something that runs on requests in this application. For that, Axum gives us `middleware::from_fn`, which lets you write middleware as a plain async function. It's so much simpler than implementing the full Tower `Layer` and `Service` traits, and in my experience it covers the vast majority of custom middleware needs.

```rust
use axum::{
    extract::Request,
    middleware::Next,
    response::Response,
};

async fn timing_middleware(req: Request, next: Next) -> Response {
    let start = std::time::Instant::now();
    let method = req.method().clone();
    let uri = req.uri().clone();

    let response = next.run(req).await;

    let duration = start.elapsed();
    tracing::info!(
        method = %method,
        uri = %uri,
        status = %response.status(),
        duration_ms = %duration.as_millis(),
        "request completed"
    );

    response
}
```

Applying it to your router is straightforward:

```rust
use axum::middleware;

let app = Router::new()
    .merge(api_routes())
    .layer(middleware::from_fn(timing_middleware))
    .with_state(state);
```

But what if your middleware needs access to the application state, say, to look up a JWT secret for authentication? That's where `from_fn_with_state` comes in:

```rust
async fn auth_middleware(
    State(state): State<AppState>,
    mut req: Request,
    next: Next,
) -> Result<Response, AppError> {
    let token = req.headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(AppError::Unauthorized)?;

    let claims = decode_jwt(token, state.config.jwt_secret.expose_secret())
        .map_err(|_| AppError::Unauthorized)?;

    // Store the authenticated user in request extensions so handlers can access it
    req.extensions_mut().insert(AuthUser::from(claims));

    Ok(next.run(req).await)
}

// Apply to specific routes
let protected_routes = Router::new()
    .route("/profile", get(get_profile))
    .route("/settings", put(update_settings))
    .route_layer(middleware::from_fn_with_state(state.clone(), auth_middleware));
```

## `route_layer` vs. `layer`

This distinction trips up a lot of people, and it really does matter for correctness.

**`.layer()`** applies the middleware to all routes on the router, including fallback handlers for unmatched paths. If you put an authentication layer here, even 404 responses for nonexistent paths will require authentication. That's usually not what you want, and it leads to confusing behavior where unauthenticated users get a 401 instead of a 404 for paths that don't exist.

**`.route_layer()`** applies the middleware only to routes that actually matched. Unmatched paths pass through to the fallback handler without hitting the middleware at all. This is what you want for authentication and authorization.

```rust
let app = Router::new()
    // Public routes
    .route("/health", get(health_check))
    .route("/api/v1/auth/login", post(login))
    // Protected routes
    .nest("/api/v1", protected_routes)
    // Global middleware (applied to everything)
    .layer(TraceLayer::new_for_http())
    .with_state(state);

let protected_routes = Router::new()
    .nest("/users", user_routes())
    .nest("/posts", post_routes())
    // Auth middleware only applies to matched routes
    .route_layer(middleware::from_fn_with_state(state, auth_middleware));
```

## Available middleware crates

Before you write custom middleware, it's worth checking whether someone has already solved your problem. The Tower and tower-http ecosystems are surprisingly rich:

- **tower-http** includes `CorsLayer`, `CompressionLayer`, `TraceLayer`, `TimeoutLayer`, `RequestBodyLimitLayer`, `SetRequestIdLayer`, and many more
- **tower-governor** provides rate limiting based on the governor algorithm
- **tower-sessions** handles server-side sessions
- **tower-cookies** provides cookie management
- **axum-csrf-sync-pattern** implements the OWASP CSRF Synchronizer Token Pattern

In my experience, the Tower ecosystem covers most of what you'll need out of the box. It's one of the best reasons to use Axum in the first place. When we get to the [Putting It All Together](./putting-it-all-together.md) chapter, you'll see how all these middleware layers come together in a complete application.
