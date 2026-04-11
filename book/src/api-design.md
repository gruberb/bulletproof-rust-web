# API Design

If you've ever integrated with an API where every endpoint returned data in a slightly different shape, you know how frustrating that gets. You spend more time reading docs (or guessing) than actually building. I want us to avoid inflicting that on anyone, including our future selves. In this chapter, we'll walk through the REST conventions, response patterns, pagination strategies, and documentation approaches that make our Axum API predictable and pleasant to work with.

## REST conventions

REST isn't a formal specification with strict rules. It's more a set of conventions that most API consumers have come to expect. Let's go through the ones that matter most.

**Use plural nouns for resources.** `/api/v1/users`, not `/api/v1/user`. The resource name represents a collection, and individual items within that collection get accessed by their ID.

**Use HTTP methods to express the operation.** `GET` retrieves data, `POST` creates new resources, `PUT` replaces a resource entirely, `PATCH` applies a partial update, and `DELETE` removes a resource. This might seem obvious, but I've seen plenty of APIs that use `POST` for everything.

**Use appropriate status codes.** Honestly, this is one of the highest-leverage things you can do for API usability. When a client creates a resource, return `201 Created`, not `200 OK`. When a delete succeeds, return `204 No Content`. When the client sends invalid input, return `400 Bad Request` with details about what went wrong. When the requested resource doesn't exist, return `404 Not Found`.

Here are the status codes you'll reach for most often:

| Code | Meaning | When to use |
|------|---------|------------|
| 200 | OK | Successful read or update |
| 201 | Created | A new resource was created |
| 204 | No Content | Successful operation with no response body (delete) |
| 400 | Bad Request | Invalid input or validation failure |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Business rule violation (duplicate email, etc.) |
| 422 | Unprocessable Entity | Semantically invalid request |
| 500 | Internal Server Error | Unexpected server failure |

## API versioning

If you're building a public API or platform service, version from the start. Adding versioning later means either breaking changes or awkward workarounds, and the cost of stamping `/v1` on your routes from day one is basically nothing.

For internal services that ship in lockstep with their consumers, path versioning is less critical. In my experience, disciplined schema evolution (additive changes, deprecation windows, contract tests) often matters more than a version prefix. Don't version just because a guide told you to. Version because your consumers need stability guarantees that you can't provide through coordination alone.

The simplest and most widely used approach is path-based versioning:

```rust
fn api_routes() -> Router<AppState> {
    Router::new()
        .nest("/api/v1", v1_routes())
}

fn v1_routes() -> Router<AppState> {
    Router::new()
        .nest("/users", user_routes())
        .nest("/posts", post_routes())
}
```

When you eventually need a v2 of a particular endpoint, you just add it alongside v1 without disturbing existing consumers:

```rust
fn api_routes() -> Router<AppState> {
    Router::new()
        .nest("/api/v1", v1_routes())
        .nest("/api/v2", v2_routes())
}
```

One thing worth keeping in mind: try to keep your handler implementations decoupled from the version prefix so that v1 and v2 routes can share the same underlying service logic where the behavior hasn't changed.

## Consistent response shapes

You might wonder why we'd bother wrapping every response in a standard type. The reason is simple: when every endpoint returns data in a predictable structure, clients can write generic parsing logic instead of special-casing each endpoint. Let's define a few wrapper types.

```rust
#[derive(Serialize)]
pub struct ApiResponse<T: Serialize> {
    pub data: T,
}

#[derive(Serialize)]
pub struct PaginatedResponse<T: Serialize> {
    pub data: Vec<T>,
    pub meta: PaginationMeta,
}

#[derive(Serialize)]
pub struct PaginationMeta {
    pub page: u32,
    pub per_page: u32,
    pub total: u64,
    pub total_pages: u32,
}
```

For error responses, we'll use the format described in the [Error Handling](./error-handling.md) chapter. That way, clients can always check for an `error` field to determine whether the request succeeded.

## Pagination

Any endpoint that returns a list of resources should support pagination. Without it, you're one large dataset away from timeouts, out-of-memory errors, and unhappy consumers. Trust me, it's much easier to add pagination now than to bolt it on later when your users table has grown to a few hundred thousand rows.

**Offset-based pagination** is the simplest approach and works well when the dataset isn't enormous and records aren't being inserted or deleted frequently during pagination:

```rust
#[derive(Debug, Deserialize)]
pub struct PaginationParams {
    #[serde(default = "default_page")]
    pub page: u32,
    #[serde(default = "default_per_page")]
    pub per_page: u32,
}

fn default_page() -> u32 { 1 }
fn default_per_page() -> u32 { 20 }

async fn list_users(
    State(state): State<AppState>,
    Query(params): Query<PaginationParams>,
) -> AppResult<Json<PaginatedResponse<UserResponse>>> {
    let per_page = params.per_page.clamp(1, 100); // at least 1, at most 100
    let offset = (params.page.saturating_sub(1)) * per_page;

    let (users, total) = state.user_service
        .list(offset, per_page)
        .await?;

    Ok(Json(PaginatedResponse {
        data: users.into_iter().map(Into::into).collect(),
        meta: PaginationMeta {
            page: params.page,
            per_page,
            total,
            total_pages: total.div_ceil(per_page as u64) as u32,
        },
    }))
}
```

**Cursor-based pagination** is better for large or frequently-changing datasets. Instead of an offset, the client passes a cursor (typically the ID or timestamp of the last item they received), and the server returns the next page starting after that cursor. This avoids the "skipping rows" problem, where offset-based pagination can miss or duplicate records when data changes between pages.

If you don't want to implement cursor logic yourself, the `paginator-axum` crate provides cursor-based pagination with metadata including `next_cursor` and `prev_cursor` fields. It's a solid starting point.

## Filtering and sorting

For list endpoints that need filtering, we accept filter parameters as query strings. This is pretty straightforward:

```rust
#[derive(Debug, Deserialize)]
pub struct UserListParams {
    #[serde(default = "default_page")]
    pub page: u32,
    #[serde(default = "default_per_page")]
    pub per_page: u32,
    pub role: Option<Role>,
    pub search: Option<String>,
    #[serde(default = "default_sort")]
    pub sort_by: String,
    #[serde(default = "default_sort_direction")]
    pub sort_direction: SortDirection,
}
```

One thing to watch out for: be mindful about which fields you allow sorting on, and make sure those fields are indexed in your database. Letting users sort on an unindexed column is a recipe for slow queries that'll bite you in production.

## OpenAPI documentation

We've all dealt with API docs that were accurate when someone wrote them six months ago and have slowly drifted out of date since. What I've found works much better is generating documentation directly from the code. The `utoipa` crate does exactly this, producing OpenAPI specifications from your Rust types and handler annotations. Since the docs come from the actual code, they can't go stale.

```rust
use utoipa::{OpenApi, ToSchema};

#[derive(Serialize, ToSchema)]
pub struct UserResponse {
    pub id: Uuid,
    pub name: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
}

#[utoipa::path(
    post,
    path = "/api/v1/users",
    request_body = CreateUserDto,
    responses(
        (status = 201, description = "User created successfully", body = UserResponse),
        (status = 400, description = "Validation error", body = ErrorResponse),
        (status = 409, description = "User with this email already exists", body = ErrorResponse),
    ),
    tag = "users"
)]
async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    // ...
}
```

To serve the Swagger UI alongside our API, we wire it up like this:

```rust
use utoipa::OpenApi;
use utoipa_swagger_ui::SwaggerUi;

#[derive(OpenApi)]
#[openapi(
    paths(create_user, get_user, list_users, update_user, delete_user),
    components(schemas(UserResponse, CreateUserDto, UpdateUserDto, ErrorResponse)),
    tags((name = "users", description = "User management endpoints"))
)]
struct ApiDoc;

let app = Router::new()
    .merge(api_routes())
    .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", ApiDoc::openapi()))
    .with_state(state);
```

Now developers can browse our API documentation at `/swagger-ui` and try out requests directly from the browser. The specification is generated at compile time from the actual types and handlers, so it literally can't fall out of sync with the implementation. That's a nice property to have.

## Health check endpoints

Every production API needs health check endpoints. Your load balancer and container orchestrator need a way to know whether your service is alive and ready to accept traffic. Let's look at how we set these up.

```rust
/// Liveness probe: is the process running and able to handle requests?
async fn health_live() -> StatusCode {
    StatusCode::OK
}

/// Readiness probe: is the application ready to serve traffic?
/// Checks that all dependencies (database, cache, etc.) are reachable.
async fn health_ready(State(state): State<AppState>) -> StatusCode {
    match sqlx::query("SELECT 1").execute(&state.db).await {
        Ok(_) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

We mount these outside our versioned API routes so they remain stable even when the API evolves:

```rust
let app = Router::new()
    .route("/health", get(health_live))
    .route("/health/ready", get(health_ready))
    .merge(api_routes())  // api_routes() already nests under /api/v1
    .with_state(state);
```

The liveness probe should be fast and unconditional. All it tells the orchestrator is "yes, the process is alive." The readiness probe is the one that checks whether our dependencies (database, cache, whatever else) are healthy, so the orchestrator knows it's safe to route traffic to this instance. Getting these two confused is a common mistake, and it can lead to your orchestrator restarting healthy containers just because the database had a brief hiccup.

With our API designed this way, we have a solid foundation: consistent URLs, predictable response shapes, pagination that won't fall over at scale, docs that stay accurate, and health checks that keep our infrastructure informed. Next, let's look at how we handle the errors that inevitably come up when all of this is running in production.
