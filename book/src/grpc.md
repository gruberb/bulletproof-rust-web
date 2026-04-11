# gRPC with Tonic

If you've been building an Axum service following the patterns in earlier chapters, at some point you'll run into a familiar question: how do we talk to other internal services? An HTTP/JSON API is great for browser clients and third-party consumers, but for service-to-service communication, you often want something with stronger schema guarantees, built-in streaming, and automatic code generation. That's where gRPC comes in. Tonic is the go-to gRPC framework in the Rust ecosystem, and since it builds on Tokio just like Axum does, the two play nicely together.

In this chapter, we'll walk through adding a gRPC surface to our existing Rust service, running it alongside our Axum HTTP server, and applying the same architectural principles we've been using all along (thin handlers, domain separation, centralized error handling) to the gRPC side of things.

## When to use gRPC

gRPC is a natural fit when your consumers are other services that you control. Both sides benefit from the shared schema defined in `.proto` files, the protobuf wire format is more compact than JSON, and you get type safety across language boundaries for free. Streaming RPCs (server-streaming, client-streaming, bidirectional) are first-class concepts rather than something you bolt on after the fact.

Where gRPC is less of a fit is browser-based clients (though gRPC-Web bridges that gap), public APIs where curl-ability matters, or simple CRUD services where the protobuf machinery adds more overhead than it's worth.

In my experience, most production systems end up using both: gRPC for internal communication between services, and HTTP/JSON for the public-facing API. That's exactly the scenario we'll tackle here.

## Proto file organization

Protocol buffer definitions live in `.proto` files, typically in a `proto/` directory at the root of your project or workspace:

```
my-service/
├── proto/
│   └── myservice/
│       └── v1/
│           ├── users.proto
│           └── health.proto
├── build.rs
├── Cargo.toml
└── src/
    └── ...
```

You might wonder why we version the proto package (e.g., `myservice.v1`) from the start. It's actually even more important here than it is with REST, because proto schemas get compiled into client code. Once consumers depend on those generated types, changing the schema is a breaking change for every one of them. Versioning from day one saves you from painful migrations later.

A typical proto file looks like this:

```protobuf
syntax = "proto3";

package myservice.v1;

service UserService {
    rpc GetUser (GetUserRequest) returns (GetUserResponse);
    rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
    rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
    string id = 1;
}

message GetUserResponse {
    string id = 1;
    string name = 2;
    string email = 3;
}

// ... other messages
```

## Build-time code generation

Tonic uses a build script to compile `.proto` files into Rust code at build time. Let's add the dependencies to our `Cargo.toml`:

```toml
[dependencies]
tonic = "0.14"
tonic-prost = "0.14"
prost = "0.14"

[build-dependencies]
tonic-prost-build = "0.14"
```

One thing that might trip you up: as of tonic 0.14, protobuf code generation uses `tonic-prost-build`, not `tonic-build` directly. The `tonic-build` crate still exists as the underlying codegen infrastructure, but `tonic-prost-build` is what you actually depend on for protobuf compilation. You'll also need `tonic-prost` as a runtime dependency alongside `tonic` itself.

Now let's create our `build.rs`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_prost_build::configure()
        .build_server(true)
        .build_client(true)
        .compile_protos(
            &["proto/myservice/v1/users.proto"],
            &["proto/"],
        )?;
    Ok(())
}
```

The generated code ends up in your `target/` directory, and you pull it in with `tonic::include_proto!("myservice.v1")`. If you need the generated types to play nicely with your domain types, you can configure `tonic_build` to add custom derives (like `Eq`, `Hash`, or `serde::Serialize`).

## Implementing a gRPC service

The generated code gives you a trait to implement for your service. Each RPC method becomes a trait method that receives a `Request<T>` and returns a `Result<Response<U>, Status>`. If you've been writing Axum handlers, this will feel very familiar.

```rust
use tonic::{Request, Response, Status};

// Include the generated code.
pub mod pb {
    tonic::include_proto!("myservice.v1");
}

use pb::user_service_server::{UserService, UserServiceServer};

pub struct MyUserService {
    // Same domain services and repositories you use in Axum handlers.
    inner: domain::services::UserService<PostgresUserRepo>,
}

impl UserService for MyUserService {
    async fn get_user(
        &self,
        request: Request<pb::GetUserRequest>,
    ) -> Result<Response<pb::GetUserResponse>, Status> {
        let req = request.into_inner();
        let id = Uuid::parse_str(&req.id)
            .map_err(|_| Status::invalid_argument("invalid user ID"))?;

        let user = self.inner
            .get_by_id(&UserId::from_uuid(id))
            .await
            .map_err(|e| {
                tracing::error!(error = ?e, "failed to fetch user");
                Status::internal("internal error")
            })?
            .ok_or_else(|| Status::not_found("user not found"))?;

        Ok(Response::new(pb::GetUserResponse {
            id: user.id().as_uuid().to_string(),
            name: user.name().as_str().to_string(),
            email: user.email().as_str().to_string(),
        }))
    }

    // ... other methods
}
```

Notice how this follows the exact same pattern as our Axum handlers: extract input, call the domain service, map the result to the transport format. The differences are small. Errors map to `tonic::Status` codes rather than HTTP status codes, and request/response types come from protobuf instead of JSON deserialization. But the shape of the code is the same, which is the whole point of our layered architecture.

## Error handling in gRPC

`tonic::Status` is the gRPC equivalent of HTTP status codes, but it also carries an error message. The standard codes include `NotFound`, `InvalidArgument`, `PermissionDenied`, `Internal`, `Unavailable`, and others that map fairly directly to HTTP semantics.

If you already have an `AppError` enum for your Axum handlers (as described in the [Error Handling](./error-handling.md) chapter), you can write a conversion from `AppError` to `Status` and reuse all that work:

```rust
impl From<AppError> for Status {
    fn from(err: AppError) -> Self {
        match err {
            AppError::NotFound => Status::not_found("resource not found"),
            AppError::Validation(msg) => Status::invalid_argument(msg),
            AppError::Unauthorized => Status::unauthenticated("authentication required"),
            AppError::Forbidden => Status::permission_denied("insufficient permissions"),
            AppError::Conflict(msg) => Status::already_exists(msg),
            AppError::Internal(e) => {
                tracing::error!(error = ?e, "internal error in gRPC handler");
                Status::internal("internal error")
            }
            _ => Status::internal("internal error"),
        }
    }
}
```

With this in place, your gRPC handlers can use the same `?` propagation pattern as your Axum handlers. Domain errors automatically map to the right gRPC status codes, and you don't have to repeat mapping logic everywhere.

## Running gRPC alongside Axum

There are two common approaches for running both HTTP and gRPC in the same process. Let's look at each.

**Separate ports** is the simpler option, and it's what I'd recommend starting with. Our Axum server binds to port 3000, our Tonic server binds to port 50051. Each runs in its own `tokio::spawn`, and they share the same `AppState`, database pool, and domain services.

```rust
// Spawn the HTTP server.
let http_handle = tokio::spawn(async move {
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, http_router)
        .with_graceful_shutdown(http_token.cancelled_owned())
        .await
        .unwrap();
});

// Spawn the gRPC server.
let grpc_handle = tokio::spawn(async move {
    tonic::transport::Server::builder()
        .add_service(UserServiceServer::new(grpc_user_service))
        .serve_with_shutdown(
            "0.0.0.0:50051".parse().unwrap(),
            grpc_token.cancelled_owned(),
        )
        .await
        .unwrap();
});
```

**Same port with protocol detection** is more complex but reduces your operational surface. Tonic's `Server` can be composed with an Axum router so that HTTP/2 gRPC requests and HTTP/1.1 REST requests are handled by the same listener. This requires careful configuration though, and in my experience it's usually only worth it when you have a strong reason to avoid multiple ports, like a restrictive network policy.

For most applications, go with separate ports. It's simpler, easier to monitor independently, and you won't run into protocol-detection edge cases.

## Middleware and interceptors

Tonic supports interceptors, which are the gRPC equivalent of middleware. You can add authentication, logging, and other cross-cutting concerns just like you would in Axum:

```rust
use tonic::service::interceptor;

fn auth_interceptor(req: Request<()>) -> Result<Request<()>, Status> {
    let token = req.metadata()
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or_else(|| Status::unauthenticated("missing token"))?;

    // Validate token...

    Ok(req)
}

// Apply the interceptor to a service.
let user_service = UserServiceServer::with_interceptor(
    MyUserService::new(state),
    auth_interceptor,
);
```

For more complex middleware (tracing, metrics, timeouts), Tonic integrates with Tower layers, just like Axum does. You can apply the same `TraceLayer`, `TimeoutLayer`, and other Tower middleware to your gRPC server, which means the skills you've built up in earlier chapters transfer directly.

## gRPC reflection

Adding server reflection lets clients like `grpcurl` and `grpc-ui` discover your service's API without needing the proto files locally. Think of it as the gRPC equivalent of Swagger UI. It's incredibly useful during development, so I'd recommend setting it up early.

```rust
use tonic_reflection::server::Builder;

let reflection_service = Builder::configure()
    .register_encoded_file_descriptor_set(pb::FILE_DESCRIPTOR_SET)
    .build_v1()?;

tonic::transport::Server::builder()
    .add_service(reflection_service)
    .add_service(UserServiceServer::new(user_service))
    .serve(addr)
    .await?;
```

To generate the file descriptor set, add this to your `build.rs`:

```rust
tonic_prost_build::configure()
    .file_descriptor_set_path(
        std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap())
            .join("myservice_descriptor.bin")
    )
    .compile_protos(&["proto/myservice/v1/users.proto"], &["proto/"])?;
```

## Testing gRPC services

You can test gRPC services by spinning up a real server in the test and connecting to it with a generated client:

```rust
#[tokio::test]
async fn test_get_user() {
    let addr = start_test_grpc_server().await;
    let mut client = UserServiceClient::connect(
        format!("http://{}", addr)
    ).await.unwrap();

    let response = client
        .get_user(pb::GetUserRequest {
            id: "some-uuid".to_string(),
        })
        .await
        .unwrap();

    assert_eq!(response.into_inner().name, "Alice");
}
```

If the network overhead bothers you for fast unit tests, you can also use `tonic::transport::Channel` with an in-process transport. It's similar to Axum's `oneshot` testing pattern, and it keeps your tests quick without sacrificing coverage.
