# Security

If there's one thing I want you to take away from this book, it's that security isn't something you bolt on at the end. It's a set of practices that live in every layer of your application, from how you handle user input to how you configure your deployment. We've touched on security throughout the earlier chapters, so this chapter pulls all of those threads together into a single reference. I've also included a few topics here that didn't fit neatly anywhere else.

## Input handling

Our first line of defense is simple: treat all external input as untrusted. You might think that's obvious, but it's easy to let your guard down when you control both the client and the server. Don't.

**Parameterized queries only.** Never construct SQL by concatenating strings. Every database query should use parameter placeholders (`$1`, `$2`, etc.) with bound values. SQLx's `query!` and `query_as!` macros enforce this at compile time, which is honestly one of the strongest reasons to use them.

**Validate at the API boundary.** We covered this in detail in the [Request Validation](./validation.md) chapter. The idea is to reject malformed input before it ever reaches your business logic. Check string lengths, format constraints, and required fields right at the edge.

**Parse into domain types.** If you've read the [Domain Modeling](./domain-modeling.md) chapter, you already know this one. Convert raw input into typed domain objects as early as possible. A `UserName` that's been through its `parse` constructor simply can't contain forbidden characters, by construction. The type system does the heavy lifting for us.

**Limit request body sizes.** Use `RequestBodyLimitLayer` to reject oversized payloads before they eat up memory. A 1 MB limit is reasonable for most JSON APIs.

## Authentication and secrets

**Hash passwords with Argon2.** We covered this in the [Authentication](./authentication.md) chapter, but it's worth repeating: Argon2 is the current best practice for password hashing. Always use a random salt for each password.

**Use long, random JWT secrets.** Your JWT secret should be at least 48 characters of random data. Shorter secrets are vulnerable to brute-force attacks, and you really don't want to find that out the hard way. Store the secret in `SecretString` from the `secrecy` crate and validate its length at startup.

**Set reasonable token expiration times.** Short-lived access tokens (15 minutes to a few hours) limit the damage if a token gets compromised. If you need longer sessions, implement a refresh token mechanism where the refresh token is stored securely and can be revoked.

**Never store tokens in localStorage.** For browser-based clients, use `HttpOnly`, `Secure`, and `SameSite=Strict` cookies. These aren't accessible to JavaScript, which protects against XSS-based token theft.

## CORS

You've probably run into CORS errors during development. Cross-Origin Resource Sharing is a browser security mechanism that controls which origins can make requests to your API. A misconfigured CORS policy is one of the most common security issues I see in web APIs, partly because the quick fix during development ("just make it permissive") has a habit of sneaking into production.

```rust
// Production: explicit allow list
CorsLayer::new()
    .allow_origin([
        "https://myapp.com".parse().unwrap(),
    ])
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION])
    .allow_credentials(true)
```

In development, `CorsLayer::permissive()` is convenient, but make sure it never reaches production. A permissive CORS policy allows any origin to make requests to your API. Whether those requests carry credentials (cookies, authorization headers) depends on the browser's credentials mode and your `allow_credentials` setting, but even without credentials, a permissive policy exposes your API surface to cross-origin probing. `CorsLayer::very_permissive()` is even more dangerous because it also enables credentials. Always restrict origins in production.

## CSRF protection

If your API uses cookies for authentication (as opposed to Bearer tokens), you need to worry about Cross-Site Request Forgery attacks. The `axum-csrf-sync-pattern` crate implements the OWASP-recommended Synchronizer Token Pattern, and it works like this:

1. The server generates a random CSRF token and sends it to the client in a custom response header
2. The client includes that token in a custom request header on every state-changing request
3. The server validates that the token in the request header matches the one it issued

Why does this work? A malicious website can cause the browser to send cookies automatically, but it can't read or set custom headers on cross-origin requests (assuming your CORS policy doesn't expose them).

## Rate limiting

Rate limiting is one of those things that feels optional until someone starts hammering your login endpoint. It protects against brute-force attacks, credential stuffing, and denial-of-service attempts. The `tower-governor` crate gives us a Tower middleware that limits requests based on client IP address:

```rust
use tower_governor::{GovernorConfigBuilder, GovernorLayer};

let governor_config = GovernorConfigBuilder::default()
    .per_second(2)
    .burst_size(10)
    .finish()
    .unwrap();

let app = Router::new()
    .merge(api_routes())
    .layer(GovernorLayer { config: governor_config })
    .with_state(state);
```

For authentication endpoints (login, password reset), I'd recommend applying stricter limits. Something like 5 attempts per minute per IP on the login endpoint makes credential stuffing attacks impractical.

One thing to keep in mind: if you're running multiple application instances in production, IP-based rate limiting at the application level isn't sufficient on its own, since each instance maintains its own counters. You'll want to look at a Redis-backed rate limiter or handle rate limiting at the load balancer or API gateway level instead.

## Security headers

HTTP security headers tell browsers to enable various protective mechanisms. Which headers you need depends on whether your application serves HTML or is a pure JSON API. For an HTML-serving application, headers like `Content-Security-Policy` and `X-Frame-Options` are essential. For a JSON-only API, some of these matter less because there's no browser rendering context to protect, but I'd still recommend setting them defensively. It costs nothing and you won't regret it.

Let's apply them as a middleware layer:

```rust
use axum::http::{HeaderName, HeaderValue};

async fn security_headers(req: Request, next: Next) -> Response {
    let mut response = next.run(req).await;
    let headers = response.headers_mut();

    // Prevent MIME-type sniffing
    headers.insert(
        HeaderName::from_static("x-content-type-options"),
        HeaderValue::from_static("nosniff"),
    );

    // Control framing (prefer CSP frame-ancestors for modern browsers,
    // but X-Frame-Options is still useful for older ones)
    headers.insert(
        HeaderName::from_static("x-frame-options"),
        HeaderValue::from_static("DENY"),
    );

    // Content Security Policy: the primary defense against XSS.
    // Adjust the directives to match your application's needs.
    headers.insert(
        HeaderName::from_static("content-security-policy"),
        HeaderValue::from_static("default-src 'none'; frame-ancestors 'none'"),
    );

    // Enforce HTTPS connections
    headers.insert(
        HeaderName::from_static("strict-transport-security"),
        HeaderValue::from_static("max-age=63072000; includeSubDomains"),
    );

    headers.insert(
        HeaderName::from_static("referrer-policy"),
        HeaderValue::from_static("strict-origin-when-cross-origin"),
    );

    response
}
```

Let me walk through what's included here and what I've deliberately left out.

`Content-Security-Policy` (CSP) is the modern, first-class defense against cross-site scripting. A strict CSP is far more effective than the legacy `X-XSS-Protection` header, which is deprecated by OWASP and MDN. Don't set `X-XSS-Protection`. In some edge cases it can actually introduce vulnerabilities. Use CSP instead.

`Strict-Transport-Security` (HSTS) tells browsers to only connect over HTTPS. This prevents protocol downgrade attacks and is essential for any production deployment behind TLS.

`X-Frame-Options` provides clickjacking protection for older browsers. For modern browsers, the `frame-ancestors` directive in CSP is the preferred mechanism, but both can coexist without issues.

These headers aren't a substitute for writing secure code, but they give us defense-in-depth by enabling browser-side protections against common attack vectors. Think of them as a safety net.

## Dependency auditing

Third-party dependencies can introduce vulnerabilities, and in a Rust project you'll have more of them than you might expect. Run `cargo audit` regularly (and ideally in your CI pipeline) to check for known vulnerabilities in your dependency tree:

```
cargo install cargo-audit
cargo audit
```

This checks your `Cargo.lock` against the RustSec Advisory Database and reports any dependencies with known security issues. In my experience, keeping your dependencies up to date is one of the simplest and most effective security practices you can adopt.

## Security checklist

Here's a condensed reference of the security practices we've covered throughout this book. I'd recommend going through this list before you deploy anything to production:

- [ ] All SQL uses parameterized queries (no string interpolation)
- [ ] Request body size is limited via `RequestBodyLimitLayer`
- [ ] Request timeouts are configured via `TimeoutLayer`
- [ ] Input is validated at the API boundary
- [ ] Domain types enforce invariants through the type system
- [ ] Passwords are hashed with Argon2
- [ ] JWT secrets are at least 48 random characters
- [ ] JWT secrets are stored in `SecretString` and never logged
- [ ] Token expiration times are set and enforced
- [ ] CORS is restricted to specific origins in production
- [ ] CSRF protection is enabled if using cookie-based authentication
- [ ] Rate limiting is applied to all public endpoints
- [ ] Stricter rate limits are applied to authentication endpoints
- [ ] CSP, HSTS, X-Content-Type-Options, and Referrer-Policy headers are set
- [ ] X-XSS-Protection is not set (use CSP instead)
- [ ] CORS is driven by runtime configuration, not build profile
- [ ] Internal error details are never exposed in API responses
- [ ] JWT validation specifies algorithm, issuer, and audience explicitly
- [ ] UUIDs are used for resource IDs (makes guessing harder, but authorization does the real work)
- [ ] `cargo audit` is run regularly or in CI
- [ ] `cargo sqlx prepare --check` is run in CI
- [ ] `.env` files are in `.gitignore` and never committed
