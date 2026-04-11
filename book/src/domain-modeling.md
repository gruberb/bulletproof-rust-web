# Domain Modeling

If you've worked on a codebase of any real size, you've probably run into this: a function takes three `String` parameters, someone swaps two of them, and you don't find out until a user gets a very confusing email. I've seen this bug in production more than once, and it's the kind of thing that makes you wish the compiler could just catch it for you.

Good news: in Rust, it can. Rust's type system is expressive enough that we can encode our domain rules directly into types, so the compiler rejects bad data before our code ever runs. This chapter walks through the patterns that make that work.

The core idea comes from the functional programming world: **parse, don't validate**. Instead of accepting raw strings and then sprinkling boolean checks all over the place, we define types that can only be constructed from valid data. Once you have a value of that type, you know it's good. No re-checking needed.

## The newtype pattern

A newtype is just a struct that wraps a single value. At runtime there's zero overhead, the compiler treats it the same as the inner value. But at compile time it's a completely different type, which means the compiler will reject any attempt to use one where the other is expected. That's a pretty great deal.

Let me show you what I mean. Say your application works with email addresses. You could represent them as `String` everywhere, but that tells the compiler (and the reader) nothing about what kind of string it is. A function that accepts `(String, String)` for name and email can be called with the arguments swapped, and nobody will notice until a user gets an email addressed to "alice@example.com" with the subject line "Dear Alice Johnson". Ask me how I know.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);

impl Email {
    /// WARNING: This is a simplified validator for illustration only.
    /// In production, use a proper email validation library or at minimum
    /// a well-tested regex. The `validator` crate's `#[validate(email)]`
    /// attribute handles this at the DTO layer; this constructor is for
    /// the domain layer where you want the type itself to guarantee validity.
    pub fn parse(raw: &str) -> Result<Self, DomainError> {
        let trimmed = raw.trim().to_lowercase();
        if trimmed.contains('@') && trimmed.len() >= 3 {
            Ok(Self(trimmed))
        } else {
            Err(DomainError::InvalidEmail(raw.to_string()))
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl AsRef<str> for Email {
    fn as_ref(&self) -> &str {
        &self.0
    }
}
```

A few things worth pointing out here.

The inner `String` field is **private**. There's no way to construct an `Email` except through `parse`, which enforces our invariant. And there's no way to get a mutable reference to the inner string, so the invariant can't be violated after construction. That's the whole trick.

The `parse` method returns a `Result`, and this is the "parse" in "parse, don't validate". It either gives you back a valid `Email` or an error explaining why the input was rejected. Once you have an `Email`, you know it's good. You never need to re-check it when you pass it to another function, which is a really nice property to have.

The `AsRef<str>` implementation makes the type easy to use in contexts that accept string references, like formatting or database queries, without exposing the ability to modify the inner value.

## Building a vocabulary of domain types

Once you get the hang of the newtype pattern, I'd encourage you to apply it to every meaningful concept in your domain. It might feel like a lot of ceremony at first, but it pays off fast. Here's what a set of domain types might look like for a user management system:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(Uuid);

impl UserId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }

    pub fn from_uuid(id: Uuid) -> Self {
        Self(id)
    }

    pub fn as_uuid(&self) -> &Uuid {
        &self.0
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct UserName(String);

impl UserName {
    pub fn parse(raw: &str) -> Result<Self, DomainError> {
        let trimmed = raw.trim();
        if trimmed.is_empty() {
            return Err(DomainError::EmptyName);
        }
        if trimmed.len() > 100 {
            return Err(DomainError::NameTooLong);
        }
        // Reject characters that could cause problems in downstream systems
        let forbidden = ['/', '(', ')', '"', '<', '>', '\\', '{', '}'];
        if trimmed.chars().any(|c| forbidden.contains(&c)) {
            return Err(DomainError::NameContainsForbiddenCharacters);
        }
        Ok(Self(trimmed.to_string()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}
```

Now look at what happens to our function signatures:

```rust
// Before: what is the first String? The second? Who knows.
fn create_user(name: String, email: String) -> Result<User, Error> { ... }

// After: the types make it self-documenting and impossible to mix up.
fn create_user(name: UserName, email: Email) -> Result<User, CreateUserError> { ... }
```

The second version isn't just clearer to read. It's physically impossible to call it with the arguments in the wrong order. The compiler will reject it. I love this about Rust: you can make whole categories of bugs unrepresentable. Not "caught by a test," not "caught in code review," but literally impossible to write.

## Domain entities

An entity is a domain object with an identity (usually an ID) and a lifecycle. Think of the things that persist in your system: a user, an order, a blog post. These are your entities, and they tend to be the nouns your product team already talks about.

```rust
#[derive(Debug, Clone)]
pub struct User {
    id: UserId,
    name: UserName,
    email: Email,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

impl User {
    /// Used by the repository layer when hydrating from the database.
    pub fn hydrate(
        id: UserId,
        name: UserName,
        email: Email,
        created_at: DateTime<Utc>,
        updated_at: DateTime<Utc>,
    ) -> Self {
        Self { id, name, email, created_at, updated_at }
    }

    pub fn id(&self) -> &UserId { &self.id }
    pub fn name(&self) -> &UserName { &self.name }
    pub fn email(&self) -> &Email { &self.email }
    pub fn created_at(&self) -> DateTime<Utc> { self.created_at }
    pub fn updated_at(&self) -> DateTime<Utc> { self.updated_at }
}
```

Notice that the fields are private and the struct only exposes getter methods that return references. This is intentional. If we need to update a user's name, that should go through a method on the entity (or a service) that can enforce whatever rules apply to name changes. We don't want random code reaching in and mutating fields directly. I've been bitten by that enough times in other languages to be a little paranoid about it.

## Request types vs. entity types

Here's a pattern that might seem obvious once you see it, but I've watched teams skip it and regret it later. The data you need to **create** something is not the same as the thing itself. A `CreateUserRequest` contains a name, an email, and a password. A `User` has an ID, timestamps, and no password (it stores a hash instead). These are different concepts, so they should be different types.

```rust
/// The data needed to register a new user.
/// All fields have already been parsed into domain types.
pub struct CreateUserRequest {
    pub name: UserName,
    pub email: Email,
    pub password_hash: String,
}
```

This type lives in the domain layer. It's separate from the HTTP request DTO (which lives in the API layer) and separate from the database row (which lives in the infrastructure layer). Each layer gets its own types, and the explicit conversions between them keep the boundaries clean. Yes, it's more types to maintain. But you'll thank yourself the first time you need to change the API response without touching the database schema.

## Conversions between layers

So we have different types in each layer, which means we need to convert between them at the boundaries. The `From` and `TryFrom` traits are the idiomatic way to do this in Rust, and they work really well for this.

```rust
// Database row type (lives in infra layer)
#[derive(sqlx::FromRow)]
pub struct UserRow {
    pub id: Uuid,
    pub name: String,
    pub email: String,
    pub password_hash: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Convert from database row to domain entity.
// Using TryFrom rather than From because database values may not satisfy
// current domain invariants: legacy rows, manual SQL fixes, schema migrations,
// and previous bugs can all produce data that would fail validation today.
impl TryFrom<UserRow> for User {
    type Error = anyhow::Error;

    fn try_from(row: UserRow) -> Result<Self, Self::Error> {
        Ok(User::hydrate(
            UserId::from_uuid(row.id),
            UserName::parse(&row.name)
                .map_err(|e| anyhow::anyhow!("corrupt user name in row {}: {}", row.id, e))?,
            Email::parse(&row.email)
                .map_err(|e| anyhow::anyhow!("corrupt email in row {}: {}", row.id, e))?,
            row.created_at,
            row.updated_at,
        ))
    }
}

// API response type (lives in api layer)
#[derive(Serialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub name: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Convert from domain entity to API response
impl From<User> for UserResponse {
    fn from(user: User) -> Self {
        Self {
            id: *user.id().as_uuid(),
            name: user.name().as_str().to_string(),
            email: user.email().as_str().to_string(),
            created_at: user.created_at(),
            updated_at: user.updated_at(),
        }
    }
}
```

Notice that the response type **excludes the password hash**. This is one of my favorite things about having separate types for each boundary: you literally cannot leak sensitive data into the API response because the response type doesn't have that field. No code review required to catch it, no linter rule to forget about. The struct just doesn't have it.

## Domain errors

Our domain errors should describe business-level failures, not infrastructure-level ones. The domain knows about "user already exists" and "invalid email format." It doesn't know (and shouldn't know) about "SQL unique constraint violation" or "HTTP 409 Conflict." If you find yourself importing `sqlx` or `axum` types in your domain error enum, something has gone sideways.

```rust
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("invalid email address: {0}")]
    InvalidEmail(String),

    #[error("name cannot be empty")]
    EmptyName,

    #[error("name exceeds maximum length")]
    NameTooLong,

    #[error("name contains forbidden characters")]
    NameContainsForbiddenCharacters,
}

#[derive(Debug, thiserror::Error)]
pub enum CreateUserError {
    #[error("a user with email {email} already exists")]
    Duplicate { email: Email },

    #[error(transparent)]
    Unknown(#[from] anyhow::Error),
}
```

The `CreateUserError::Unknown` variant uses `anyhow::Error` as a catch-all for unexpected failures. This lets the infrastructure layer convert arbitrary errors (like database connection timeouts) into domain errors without the domain having to know about every possible failure mode. The API layer then maps `Unknown` to a generic 500 response while logging the full details server-side. You might wonder if this is too loose. In my experience, it strikes a good balance: your expected errors are typed, and the unexpected ones still get handled gracefully.

## Putting it all together

Let's trace the complete flow from HTTP request to database and back, so you can see how the types change at each boundary:

```
HTTP POST /api/users
  { "name": "Alice", "email": "alice@example.com", "password": "hunter2" }

  ↓ Axum deserializes into CreateUserDto (api/dtos)
  ↓ Handler parses fields into domain types: UserName, Email
  ↓ Handler calls user_service.register(name, email, &password)

  ↓ Service hashes password, builds CreateUserRequest (domain)
  ↓ Service calls repo.create(&request)

  ↓ Repository converts to SQL, executes INSERT
  ↓ Repository gets UserRow back from the database
  ↓ Repository converts UserRow into User (domain entity)

  ↑ Service returns User to handler
  ↑ Handler converts User into UserResponse (api/dtos)
  ↑ Handler returns (StatusCode::CREATED, Json(response))

HTTP 201 Created
  { "id": "...", "name": "Alice", "email": "alice@example.com", ... }
```

At every boundary, the data gets converted into the type that belongs to that layer. The API layer works with DTOs. The domain layer works with domain types. The infrastructure layer works with database row types. The conversions between them are explicit, auditable, and enforced by the compiler.

I know this might feel like a lot of ceremony compared to just passing a single struct all the way through. I felt the same way when I first started doing this. But here's what I've found in practice: when you need to add a field to the API response, you change the DTO. When you need to add a column to the database, you change the row type. When you need to add a business rule, you change the domain type. Each change stays in the layer it belongs to, and the `From` implementations tell you exactly how the layers connect. Once you've worked this way on a project that's grown past a few thousand lines, it's hard to go back.
