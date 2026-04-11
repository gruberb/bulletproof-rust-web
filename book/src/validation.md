# Request Validation

If you've ever shipped a form without server-side validation because the frontend "already checks it," you know how that story ends. Client-side validation is a nice UX touch, but it's not a security boundary. Someone with curl doesn't care about your JavaScript checks. We always validate on the server.

In my experience, validation in a well-structured app naturally splits into two levels. At the API layer, we check the shape and format of what comes in: are the required fields present? Does the email field actually look like an email? Is the password long enough? Then at the domain layer, we enforce business invariants through the type system, which I covered in the [Domain Modeling](./domain-modeling.md) chapter.

This chapter focuses on that first level, the API-layer validation. We'll use the `validator` crate and build a custom Axum extractor to make it seamless.

## Validation with the validator crate

The `validator` crate gives us derive macros to annotate struct fields with validation rules. You call `.validate()` on an instance, it checks every rule, and hands back a structured error if anything fails. Let's look at what that looks like in practice.

```rust
use serde::Deserialize;
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserDto {
    #[validate(length(min = 2, max = 50, message = "name must be between 2 and 50 characters"))]
    pub name: String,

    #[validate(email(message = "must be a valid email address"))]
    pub email: String,

    #[validate(length(min = 8, message = "password must be at least 8 characters"))]
    pub password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateUserDto {
    #[validate(length(min = 2, max = 50, message = "name must be between 2 and 50 characters"))]
    pub name: Option<String>,

    #[validate(length(max = 500, message = "bio cannot exceed 500 characters"))]
    pub bio: Option<String>,

    #[validate(url(message = "must be a valid URL"))]
    pub avatar_url: Option<String>,
}
```

The `validator` crate comes with a solid set of built-in validations: `email`, `url`, `length`, `range`, `contains`, `must_match` (handy for password confirmation), and more. If you need something that doesn't fit any of the built-in validators, you can write your own custom validation functions too.

## A custom ValidatedJson extractor

Axum's built-in `Json` extractor handles deserialization for us, but it doesn't run any validation. You could call `.validate()` by hand at the start of every handler, but honestly, that gets old fast and it's easy to forget in one place. What I've found works much better is building a custom extractor that combines deserialization and validation into a single step.

```rust
use axum::{
    extract::{FromRequest, Request},
    Json,
};
use validator::Validate;

pub struct ValidatedJson<T>(pub T);

impl<T, S> FromRequest<S> for ValidatedJson<T>
where
    T: serde::de::DeserializeOwned + Validate,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|e| AppError::Validation(format!("invalid JSON: {e}")))?;

        value.validate().map_err(|e| {
            AppError::ValidationFields(format_validation_errors(&e))
        })?;

        Ok(Self(value))
    }
}

fn format_validation_errors(
    errors: &validator::ValidationErrors,
) -> HashMap<String, Vec<String>> {
    errors
        .field_errors()
        .iter()
        .map(|(field, errs)| {
            let messages = errs
                .iter()
                .map(|e| {
                    e.message
                        .as_ref()
                        .map(|m| m.to_string())
                        .unwrap_or_else(|| format!("{} is invalid", field))
                })
                .collect();
            (field.to_string(), messages)
        })
        .collect()
}
```

Now our handlers can accept `ValidatedJson<CreateUserDto>` instead of `Json<CreateUserDto>`, and validation just happens before our handler body ever runs. If validation fails, the client gets a 400 response with a clear error message telling them exactly which fields failed and why. No extra work on our part.

```rust
async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    // By this point, we know:
    // - The JSON was well-formed
    // - name is between 2 and 50 characters
    // - email is a valid email format
    // - password is at least 8 characters
    //
    // The handler can focus on business logic.
    let user = state.user_service.register(payload).await?;
    Ok((StatusCode::CREATED, Json(user.into())))
}
```

## Validating query parameters

We can use the same approach for query parameters. Let's build a `ValidatedQuery` extractor that combines `Query` extraction with validation:

```rust
use axum::extract::Query;

pub struct ValidatedQuery<T>(pub T);

impl<T, S> FromRequestParts<S> for ValidatedQuery<T>
where
    T: serde::de::DeserializeOwned + Validate,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut http::request::Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection> {
        let Query(value) = Query::<T>::from_request_parts(parts, state)
            .await
            .map_err(|e| AppError::Validation(format!("invalid query parameters: {e}")))?;

        value.validate().map_err(|e| {
            AppError::ValidationFields(format_validation_errors(&e))
        })?;

        Ok(Self(value))
    }
}
```

## Structured error responses

If other developers are consuming your API (and they probably are), you'll want to return validation errors in a structured format they can actually parse, not just a single concatenated string. I like returning errors as a JSON object with per-field error lists:

```json
{
    "error": {
        "type": "validation_error",
        "message": "request validation failed",
        "fields": {
            "email": ["must be a valid email address"],
            "password": ["password must be at least 8 characters"]
        }
    }
}
```

To make this work, we need to extend our `AppError` enum to carry structured field errors and adjust the `IntoResponse` implementation to format them. The exact shape depends on whatever conventions your API follows, but the idea is the same: give the client enough information to fix everything in one shot. Nobody wants to fix one field, resubmit, and then discover the next failure.

## Pre-built validation extractors

If writing your own extractor feels like more ceremony than you want, the ecosystem has you covered:

- **axum-valid** integrates with `validator`, `garde`, and `validify`, providing `Valid<Json<T>>`, `Valid<Query<T>>`, and `Valid<Form<T>>` extractors
- **axum-validated-extractors** provides `ValidatedJson`, `ValidatedQuery`, and `ValidatedForm` out of the box

These crates save you the boilerplate we wrote above. That said, I tend to prefer rolling my own because it gives me full control over the error format and behavior, which matters once you have opinions about your API's error responses (and you will).

## Validation at two levels

Before we move on, I want to make the distinction between API-layer and domain-layer validation really clear, because mixing them up leads to either duplicated logic or gaps in your protection. I've seen both, and neither is fun to untangle.

**API-layer validation** (what we covered in this chapter) checks the shape of the input: are required fields present? Are strings the right length? Does the email field actually look like an email? This is about the format of data as it arrives over the wire.

**Domain-layer validation** (which we cover in the [Domain Modeling](./domain-modeling.md) chapter) enforces business invariants: a username doesn't contain forbidden characters, an email gets normalized to lowercase, an order amount is positive. This validation happens when you construct domain types like `UserName::parse()` or `Email::parse()`.

These two levels work together nicely. Our API layer catches obviously malformed input early and returns a helpful error message. Our domain layer makes sure business rules are enforced consistently, whether the data comes from an HTTP request, a message queue, a CSV import, or a test harness. With both in place, we have a solid foundation for the error handling patterns we'll look at next.
