# Authentication and Authorization

Every web application eventually needs to answer two questions: "who is making this request?" and "are they allowed to do what they're asking?" The first is authentication, the second is authorization. If your app manages user data or exposes anything sensitive, you'll need both.

What I really like about Axum here is the extractor system. We can build custom extractors that validate credentials and check permissions, and then adding auth to a handler is just a matter of putting the right extractor in its argument list. Let's see how that works.

## Choosing an authentication strategy

The right approach depends on who your client is.

**First-party browser applications** should use server-side sessions with cookies. You store session IDs in `HttpOnly`, `Secure`, `SameSite=Strict` cookies, which means JavaScript can't access them. This materially reduces the risk of cookie theft through XSS. I should note that `HttpOnly` doesn't prevent XSS itself, and an injected script can still act on the user's behalf or exfiltrate other in-page data. But it does prevent the most common attack vector of stealing the session token directly, which is why OWASP and MDN both recommend it. The server stores session state (user ID, permissions, expiry) in a database or Redis, so sessions can be revoked immediately. The `tower-sessions` crate provides session middleware for Axum, and `axum-login` layers identification, authentication, and authorization on top. If you go with cookie-based sessions, you'll also need CSRF protection as described in the [Security](./security.md) chapter.

**Service-to-service APIs and third-party integrations** are where stateless JWTs shine. The client includes a token in the `Authorization: Bearer` header, and the server validates it without hitting a database. This is simpler for machine clients that don't have a browser cookie jar, and it scales horizontally because there's no server-side session state to share. The tradeoff is that JWTs can't be revoked before expiry without additional infrastructure (a deny-list or short lifetimes with refresh tokens).

**Public APIs with third-party consumers** often use OAuth2/OIDC, where an identity provider issues tokens and your API validates them. The `jwt-authorizer` crate supports OIDC discovery and automatic key rotation for this use case.

For the rest of this chapter, we'll focus on the JWT approach. It's the most common pattern for API-style services, and it shows off Axum's extractor pattern nicely. If you're building a browser-facing application, start with `tower-sessions` and `axum-login` instead.

## JWT authentication with a custom extractor

JWTs work well for stateless API authentication. The flow is pretty straightforward: the client authenticates (typically with email and password), gets back a JWT, and then includes it in the `Authorization` header on every subsequent request. The server validates the token each time without needing to look anything up in a database.

The key to making this work cleanly in Axum is building a custom extractor that implements `FromRequestParts`. When you put this extractor in a handler's signature, Axum runs the validation logic automatically before the handler body ever executes. Let's look at what that looks like.

```rust
use axum::{
    extract::FromRequestParts,
    http::request::Parts,
};
use axum_extra::{
    headers::{Authorization, authorization::Bearer},
    TypedHeader,
};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};

/// Represents an authenticated user. Including this in a handler's
/// signature automatically requires and validates a JWT.
#[derive(Debug, Clone)]
pub struct AuthUser {
    pub user_id: Uuid,
    pub email: String,
    pub role: Role,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Claims {
    pub sub: Uuid,       // subject (user ID)
    pub email: String,
    pub role: Role,
    pub iss: String,     // issuer
    pub aud: String,     // audience
    pub exp: usize,      // expiration time
    pub iat: usize,      // issued at
}

impl<S> FromRequestParts<S> for AuthUser
where
    AppState: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let app_state = AppState::from_ref(state);

        // Extract the Authorization: Bearer <token> header
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AppError::Unauthorized)?;

        // Decode and validate the JWT with explicit validation rules.
        // Validation::default() uses HS256 and checks exp, but you should
        // be explicit about what you accept in production.
        let mut validation = Validation::new(Algorithm::HS256);
        validation.set_issuer(&["myapp"]);
        validation.set_audience(&["myapp-api"]);
        validation.leeway = 30; // 30 seconds of clock skew tolerance

        let token_data = decode::<Claims>(
            bearer.token(),
            &DecodingKey::from_secret(
                app_state.config.jwt_secret.expose_secret().as_bytes()
            ),
            &validation,
        )
        .map_err(|_| AppError::Unauthorized)?;

        Ok(AuthUser {
            user_id: token_data.claims.sub,
            email: token_data.claims.email,
            role: token_data.claims.role,
        })
    }
}
```

Using it in a handler is as simple as adding the parameter to the function signature:

```rust
async fn get_my_profile(
    user: AuthUser,
    State(state): State<AppState>,
) -> AppResult<Json<ProfileResponse>> {
    let profile = state.user_service
        .get_by_id(UserId::from_uuid(user.user_id))
        .await?
        .ok_or(AppError::NotFound)?;

    Ok(Json(profile.into()))
}
```

If the JWT is missing, expired, or invalid, our extractor returns `AppError::Unauthorized` and the handler body never runs. There's no explicit auth-checking code in the handler itself, which is exactly the kind of separation I want to see.

### Optional authentication with MaybeAuthUser

Some endpoints behave differently depending on whether the caller is logged in, without actually requiring authentication. Think of a feed endpoint that shows personalized results for logged-in users and generic results for anonymous visitors.

You might be tempted to just use `Option<AuthUser>` here, but that loses an important distinction. `None` could mean "no Authorization header was sent" (the user is anonymous) or "an Authorization header was sent but the token was garbage" (which should be a 401, not a silent fallback to anonymous behavior). Those are very different situations.

A dedicated `MaybeAuthUser` extractor lets us tell them apart:

```rust
pub struct MaybeAuthUser(pub Option<AuthUser>);

impl<S> FromRequestParts<S> for MaybeAuthUser
where
    AppState: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        // If there is no Authorization header at all, that is fine.
        let Some(auth_header) = parts.headers.get(header::AUTHORIZATION) else {
            return Ok(Self(None));
        };

        // But if a header IS present, it must be valid.
        // An invalid token is an error, not anonymous access.
        let token = auth_header
            .to_str()
            .ok()
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(AppError::Unauthorized)?;

        let app_state = AppState::from_ref(state);
        let mut validation = Validation::new(Algorithm::HS256);
        validation.set_issuer(&["myapp"]);
        validation.set_audience(&["myapp-api"]);

        let token_data = decode::<Claims>(
            token,
            &DecodingKey::from_secret(
                app_state.config.jwt_secret.expose_secret().as_bytes()
            ),
            &validation,
        )
        .map_err(|_| AppError::Unauthorized)?;

        Ok(Self(Some(AuthUser {
            user_id: token_data.claims.sub,
            email: token_data.claims.email,
            role: token_data.claims.role,
        })))
    }
}
```

I picked up this pattern from the [launchbadge/realworld-axum-sqlx](https://github.com/launchbadge/realworld-axum-sqlx) reference implementation, which documents it as a deliberate design choice. It's a small detail, but it prevents a whole category of subtle auth bugs where a client sends a malformed token and silently gets anonymous behavior instead of a clear error.

## Issuing tokens

Now let's look at the other side of the coin: how we actually create these tokens. The login endpoint validates credentials and issues a JWT:

```rust
use jsonwebtoken::{encode, EncodingKey, Header};

async fn login(
    State(state): State<AppState>,
    Json(credentials): Json<LoginDto>,
) -> AppResult<Json<TokenResponse>> {
    let user = state.user_service
        .authenticate(&credentials.email, &credentials.password)
        .await?
        .ok_or(AppError::Unauthorized)?;

    let now = chrono::Utc::now();
    let claims = Claims {
        sub: *user.id().as_uuid(),
        email: user.email().as_str().to_string(),
        role: user.role().clone(),
        iss: "myapp".to_string(),
        aud: "myapp-api".to_string(),
        iat: now.timestamp() as usize,
        // Short-lived access tokens limit damage if compromised.
        // For longer sessions, implement a refresh token mechanism.
        exp: (now + chrono::TimeDelta::try_minutes(15).expect("valid duration"))
            .timestamp() as usize,
    };

    let token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(
            state.config.jwt_secret.expose_secret().as_bytes()
        ),
    )
    .map_err(|e| AppError::Internal(e.into()))?;

    Ok(Json(TokenResponse { token }))
}
```

There are a few production considerations here that are easy to overlook.

**Token lifetime.** Our example uses 15-minute access tokens. Short lifetimes limit the window of exposure if a token gets stolen. For user-facing applications that need longer sessions, you'll want a separate refresh token flow where the refresh token lives in a secure, HttpOnly cookie and can be revoked server-side.

**Algorithm pinning.** Always specify the expected algorithm explicitly (here, `Algorithm::HS256`) rather than trusting the `alg` header in the token itself. Accepting the token's self-declared algorithm is a well-known attack vector, and I've seen it bite teams who thought they were being flexible.

**Issuer and audience.** Setting `iss` and `aud` claims, and validating them on decode, prevents tokens issued for one service from being accepted by another. This matters as soon as you have more than one service sharing a secret or using the same identity provider.

**Role claims and revocation.** Here's something that catches people off guard: embedding roles in the JWT means that permission changes (revoking admin access, disabling an account) don't take effect until the token expires and a new one is issued. If immediate revocation matters for your application, you'll need either very short token lifetimes, a token version check against the database, or a deny-list.

## Password hashing

You already know not to store passwords in plain text, but the choice of hashing algorithm matters too. For new projects, use Argon2. It won the Password Hashing Competition in 2015 and it's still the best option we have. Don't use SHA-256, MD5, or even bcrypt for new code.

```rust
use argon2::{
    Argon2,
    PasswordHash,
    PasswordHasher,
    PasswordVerifier,
    password_hash::SaltString,
};
use argon2::password_hash::rand_core::OsRng;

pub fn hash_password(password: &str) -> anyhow::Result<String> {
    let salt = SaltString::generate(&mut OsRng);
    // Use Argon2id, which is the variant recommended by OWASP.
    // The default() configuration uses Argon2id with reasonable parameters,
    // but for production you should tune memory cost and iterations based
    // on your hardware and OWASP's current minimum recommendations.
    let hasher = Argon2::default(); // Argon2id with default params
    let hash = hasher
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| anyhow::anyhow!("failed to hash password: {}", e))?
        .to_string();
    Ok(hash)
}

pub fn verify_password(password: &str, hash: &str) -> anyhow::Result<bool> {
    let parsed_hash = PasswordHash::new(hash)
        .map_err(|e| anyhow::anyhow!("failed to parse password hash: {}", e))?;
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}
```

One thing to keep in mind: password hashing is deliberately slow. That's the whole point, making brute-force attacks impractical. But in an async context, this means you should run it on a blocking thread so you don't tie up the async runtime:

```rust
let hash = tokio::task::spawn_blocking(move || hash_password(&password))
    .await
    .context("password hashing task failed")??;
```

## Role-based authorization

Once we have authentication working, authorization is the natural next step. The simplest approach is just checking the user's role inside the handler:

```rust
async fn admin_only_endpoint(
    user: AuthUser,
    State(state): State<AppState>,
) -> AppResult<Json<AdminData>> {
    if user.role != Role::Admin {
        return Err(AppError::Forbidden);
    }
    // ... admin logic
}
```

That works, but if you find yourself repeating the same role check across multiple handlers, we can do better. Let's build an extractor that bakes the role requirement right in:

```rust
pub struct RequireAdmin(pub AuthUser);

impl<S> FromRequestParts<S> for RequireAdmin
where
    AppState: FromRef<S>,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let user = AuthUser::from_request_parts(parts, state).await?;

        if user.role != Role::Admin {
            return Err(AppError::Forbidden);
        }

        Ok(Self(user))
    }
}

// Usage: just include it in the handler signature
async fn admin_endpoint(RequireAdmin(user): RequireAdmin) -> AppResult<Json<AdminData>> {
    // If we get here, the user is authenticated AND has the admin role
    // ...
}
```

## Middleware-based authentication

There's another way to handle this: instead of putting the extractor on each handler, you can apply authentication as middleware to a whole group of routes. This is handy when you have a block of routes that all require authentication and you don't want to repeat `AuthUser` in every handler signature.

```rust
let public_routes = Router::new()
    .route("/health", get(health_check))
    .route("/api/v1/auth/login", post(login))
    .route("/api/v1/auth/register", post(register));

let protected_routes = Router::new()
    .nest("/api/v1/users", user_routes())
    .nest("/api/v1/posts", post_routes())
    .route_layer(middleware::from_fn_with_state(
        state.clone(),
        auth_middleware,
    ));

let app = Router::new()
    .merge(public_routes)
    .merge(protected_routes)
    .with_state(state);
```

Which approach should you pick? In my experience, extractors are more flexible because individual handlers can opt in or out, and the handler gets direct access to the `AuthUser` value. Middleware is more convenient when entire route groups all need the same authentication requirement. You can also combine both, using middleware for the baseline and extractors for finer-grained checks within that group.

## Ecosystem crates

Before we wrap up, it's worth knowing what the ecosystem offers. You don't always have to roll your own:

- **axum-login** provides session-based authentication with pluggable backends
- **axum-gate** combines JWT validation with role-based authorization
- **axum-session** handles database-persisted sessions
- **jwt-authorizer** provides JWT validation with OIDC discovery support
- **axum-csrf-sync-pattern** implements CSRF protection for session-based authentication

For most API-style applications, I think the custom `AuthUser` extractor approach we built above gives you the right balance of simplicity and control. But when you need more complex features like session management, OAuth2 flows, or OIDC integration, these crates can save you a lot of work.
