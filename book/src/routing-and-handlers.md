# Routing and Handlers

Every HTTP request that hits your application needs a place to land, and in Axum that place is a handler. A handler is just an async function that takes zero or more extractors as arguments and returns something that implements `IntoResponse`. Axum handles the plumbing for you: it deserializes request data into the extractor types and serializes your return value back into an HTTP response.

The thing I want you to take away from this chapter early on is that handlers should be thin. Their job is to pull data out of the request, hand it off to a service or repository, and turn the result into a response. If you find yourself writing business logic, database queries, or complex branching inside a handler, that code belongs somewhere else. We'll talk about where exactly in the domain and infrastructure chapters, but for now just remember: thin handlers, always.

## Defining routes

Axum's router uses method chaining, which feels pretty natural once you've seen it a couple of times. You define individual routes with `.route()`, group related routes under a common prefix with `.nest()`, and combine separate route groups with `.merge()`.

```rust
use axum::{routing::{get, post, put, delete}, Router};

pub fn router(state: AppState) -> Router {
    Router::new()
        .nest("/api/v1", api_routes())
        .route("/health", get(health_check))
        .route("/health/ready", get(readiness_check))
        .with_state(state)
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .nest("/users", user_routes())
        .nest("/posts", post_routes())
}

fn user_routes() -> Router<AppState> {
    Router::new()
        .route("/", get(list_users).post(create_user))
        .route("/{id}", get(get_user).patch(update_user).delete(delete_user))
}

fn post_routes() -> Router<AppState> {
    Router::new()
        .route("/", get(list_posts).post(create_post))
        .route("/{id}", get(get_post).patch(update_post).delete(delete_post))
}
```

I like splitting routes into separate functions (or even separate files) because it keeps the router definition readable as your application grows. Each function returns a `Router<AppState>`, and the main router composes them together. This is also where you'd apply route-specific middleware, which we'll cover in the [Middleware](./middleware.md) chapter.

## Writing handlers

So what does a well-structured handler actually look like? In my experience, they all follow the same rhythm: extract, delegate, respond.

```rust
use axum::{extract::{State, Path}, http::StatusCode, Json};

async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    let name = UserName::parse(&payload.name)
        .map_err(|e| AppError::Validation(e.to_string()))?;
    let email = Email::parse(&payload.email)
        .map_err(|e| AppError::Validation(e.to_string()))?;

    let user = state.user_service
        .register(name, email, &payload.password)
        .await?;

    Ok((StatusCode::CREATED, Json(user.into())))
}

async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> AppResult<Json<UserResponse>> {
    let user = state.user_service
        .get_by_id(UserId::from_uuid(id))
        .await?
        .ok_or(AppError::NotFound)?;

    Ok(Json(user.into()))
}

async fn list_users(
    State(state): State<AppState>,
    Query(pagination): Query<PaginationParams>,
) -> AppResult<Json<PaginatedResponse<UserResponse>>> {
    let (users, total) = state.user_service
        .list(pagination.page, pagination.per_page)
        .await?;

    let response = PaginatedResponse {
        data: users.into_iter().map(Into::into).collect(),
        meta: PaginationMeta {
            page: pagination.page,
            per_page: pagination.per_page,
            total,
            total_pages: (total as f64 / pagination.per_page as f64).ceil() as u32,
        },
    };

    Ok(Json(response))
}

async fn delete_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> AppResult<StatusCode> {
    state.user_service
        .delete(UserId::from_uuid(id))
        .await?;

    Ok(StatusCode::NO_CONTENT)
}
```

Take a look at the conventions here. When we successfully create something, we return `StatusCode::CREATED` (201) with the created resource in the body. A successful deletion returns `StatusCode::NO_CONTENT` (204) with no body, because there's nothing left to show. And when a lookup finds nothing, we return `AppError::NotFound`, which our `IntoResponse` implementation on `AppError` turns into a proper 404 response. You might notice how little work each handler actually does. That's the goal.

## Extractors

Extractors are how Axum pulls data out of an incoming request for you. Under the hood, they implement either `FromRequest` (if they need to consume the request body) or `FromRequestParts` (if they only need headers, query parameters, path segments, or other metadata without touching the body).

The built-in extractors cover most of what you'll need day to day:

```rust
use axum::extract::{State, Path, Query, Json};

// Path parameters: /users/{id}
async fn get_user(Path(id): Path<Uuid>) -> ... { }

// Query parameters: /users?page=2&per_page=20
async fn list_users(Query(params): Query<PaginationParams>) -> ... { }

// JSON body
async fn create_user(Json(body): Json<CreateUserDto>) -> ... { }

// Application state
async fn handler(State(state): State<AppState>) -> ... { }
```

You can absolutely use multiple extractors in the same handler. The one constraint to keep in mind is that at most one extractor can consume the request body (like `Json`), and it has to come last in the argument list. If you mix up the order, the compiler will let you know.

## The `#[debug_handler]` macro

If you've ever had a handler fail to compile because of trait bound errors, you know the pain. The error messages from the Rust compiler can be really opaque in this context, and I've lost more time than I'd like to admit staring at them. Axum provides a `#[debug_handler]` macro that makes things much clearer:

```rust
#[axum::debug_handler]
async fn my_handler(
    State(state): State<AppState>,
    Json(body): Json<CreateUserDto>,
) -> AppResult<Json<UserResponse>> {
    // ...
}
```

What it does is add extra type checking that produces human-readable errors like "argument #2 must implement FromRequest" instead of a wall of trait bound failures. It has no runtime cost, so you can leave it in place in production if you want. Some teams prefer to remove it once the handler compiles correctly, but honestly I don't bother. It's one of those small things that saves you real time when you come back to change a handler six months later.

## Response types

Axum is pretty flexible about what your handlers can return. Anything that implements `IntoResponse` works, and there are quite a few built-in implementations. Here are the patterns I use most often:

```rust
// Just a status code
async fn health_check() -> StatusCode {
    StatusCode::OK
}

// A tuple of status code and body
async fn create_resource() -> (StatusCode, Json<Resource>) {
    (StatusCode::CREATED, Json(resource))
}

// A Result for fallible operations
async fn get_resource() -> Result<Json<Resource>, AppError> {
    Ok(Json(resource))
}

// Headers and body together
async fn with_headers() -> (StatusCode, [(HeaderName, &'static str); 1], Json<Data>) {
    (
        StatusCode::OK,
        [(header::CACHE_CONTROL, "max-age=3600")],
        Json(data),
    )
}
```

In practice, you'll use the `Result` return type most of the time, because most handlers can fail in one way or another. Using `Result<T, AppError>` gives you the clean error mapping we'll build in the [Error Handling](./error-handling.md) chapter, and it keeps your handler code focused on the happy path.

## Keeping handlers thin

I know I already mentioned this, but it's worth repeating because I've seen this go wrong so many times across different languages and frameworks. When a handler starts growing beyond 15 or 20 lines, that's usually a sign that logic has crept into the wrong place.

What should live in a handler:
- Extracting data from the request (via Axum extractors)
- Parsing raw input into domain types (calling `UserName::parse()`, `Email::parse()`, etc.)
- Calling a single service method
- Converting the result to an HTTP response

What should not live in a handler:
- Database queries
- Business rule enforcement
- Calls to multiple services that need to be coordinated
- Complex conditional logic
- Sending emails, publishing events, or other side effects

If your handler needs to do several things in coordination, that coordination logic belongs in a service method. We'll look at how to structure those services in the next chapter.
