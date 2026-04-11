# Configuration

I've seen enough configuration bugs to last a lifetime. A database URL that works on your laptop but not in staging. A JWT secret that ends up in a git commit. An environment variable that's missing, but you only find out ten minutes after deployment when the first request hits a code path that needs it. These are all avoidable, and the strategy we'll use here prevents every single one of them.

The idea is straightforward: pull configuration from the environment, validate it eagerly at startup, and keep sensitive values protected in memory. Let's look at how that works in practice.

## A strongly-typed config struct

We start with a Rust struct that represents all the configuration our application needs. This is one of those places where Rust's type system is a natural fit: you get compile-time guarantees about which fields exist and what types they have. And you can validate values when the struct is constructed, instead of hoping they're correct when something finally tries to use them.

```rust
use secrecy::{SecretString, ExposeSecret};
use serde::Deserialize;

#[derive(Debug, Clone, Deserialize)]
pub struct Config {
    #[serde(default = "default_port")]
    pub port: u16,

    #[serde(default = "default_environment")]
    pub environment: Environment,

    pub database_url: SecretString,

    pub jwt_secret: SecretString,

    #[serde(default = "default_log_level")]
    pub log_level: String,

    #[serde(default)]
    pub cors_origins: Vec<String>,
}

#[derive(Debug, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum Environment {
    Development,
    Staging,
    Production,
}

fn default_port() -> u16 { 3000 }
fn default_environment() -> Environment { Environment::Development }
fn default_log_level() -> String { "info".to_string() }
```

There are a few deliberate choices here that are worth walking through.

Notice that `database_url` and `jwt_secret` use `SecretString` from the `secrecy` crate instead of plain `String`. This does two things for us: the values get automatically redacted when you print the struct with `Debug` (you'll see `[REDACTED]` instead of the actual secret), and the memory is zeroed when the value is dropped. I've seen production secrets end up in log files more times than I'd like to admit, so this kind of protection is really helpful.

The `Environment` enum is a proper type rather than a string. You can match on it exhaustively, and the compiler will tell you if you forget to handle a variant. It also catches typos like "prodduction" at parse time instead of letting them slip through silently.

We provide default values for fields that have sensible defaults. That way you can start the application with minimal configuration during development, while still requiring explicit values for things like database credentials.

## Loading from environment variables

The `config` crate gives us a flexible system for loading configuration from multiple sources and merging them together. For most applications, loading from environment variables is all you need:

```rust
impl Config {
    pub fn from_env() -> anyhow::Result<Self> {
        // In development, load from a .env file if present.
        // The .ok() is intentional: in production there is no .env file,
        // and that is fine.
        dotenvy::dotenv().ok();

        let config = config::Config::builder()
            .add_source(
                config::Environment::default()
                    .separator("__")
            )
            .build()
            .context("failed to build configuration")?;

        let parsed: Config = config
            .try_deserialize()
            .context("failed to deserialize configuration")?;

        parsed.validate()?;

        Ok(parsed)
    }

    fn validate(&self) -> anyhow::Result<()> {
        if self.jwt_secret.expose_secret().len() < 48 {
            anyhow::bail!(
                "JWT_SECRET must be at least 48 characters for adequate security"
            );
        }
        Ok(())
    }
}
```

The `__` separator means you can set nested config values using double-underscore notation in environment variables. So if you had a nested `database.max_connections` field, you'd set it with `DATABASE__MAX_CONNECTIONS`.

The `validate` method runs checks that we can't express through types alone. Here it makes sure the JWT secret is long enough to resist brute-force attacks. If validation fails, the application exits immediately with a clear error message. That's way better than starting up fine and then crashing when someone tries to authenticate and the code discovers it has a three-character JWT secret.

## The .env file for development

During local development, it's convenient to store configuration in a `.env` file so you don't have to export environment variables manually every time you start the application:

```
DATABASE_URL=postgres://localhost:5432/myapp_dev
JWT_SECRET=this-is-a-local-dev-secret-that-is-long-enough-for-validation
LOG_LEVEL=debug
ENVIRONMENT=development
```

This file must be in your `.gitignore`. Don't commit it to version control, even if it only contains development secrets. In my experience, the habit of committing `.env` files always leads to someone accidentally pushing production secrets eventually. Instead, provide a `.env.example` file that documents which variables are expected:

```
# Copy this file to .env and fill in the values
DATABASE_URL=postgres://localhost:5432/myapp_dev
JWT_SECRET=<generate a random string of at least 48 characters>
LOG_LEVEL=info
ENVIRONMENT=development
```

## Using secrets safely

The `secrecy` crate is small, but it pulls a lot of weight. When you wrap a value in `SecretString`, you get three protections:

1. **Debug redaction.** Any code that prints the config struct, whether on purpose or by accident, will see `[REDACTED]` instead of the actual secret. This matters more than you might think. Structured logging and error reporting can easily serialize the entire config if you're not careful.

2. **Secure memory zeroing.** When the `SecretString` is dropped, its memory is overwritten with zeros before being deallocated. This shrinks the window during which the secret is sitting around in a memory dump.

3. **Intentional exposure.** To actually use the secret value, you have to call `.expose_secret()`, which returns a reference to the inner string. This makes every secret access explicit and easy to grep for during code review.

```rust
// When you need to use the secret, you explicitly expose it.
// Keep the exposed value's lifetime as short as possible.
let decoding_key = DecodingKey::from_secret(
    config.jwt_secret.expose_secret().as_bytes()
);
```

The rule of thumb: call `expose_secret()` at the point of use, not earlier. Don't store the exposed value in a variable that hangs around longer than it needs to, and never log it.

## Production deployment

In production, your configuration should come from the deployment platform's secrets management, not from files. That could be:

- Kubernetes secrets mounted as environment variables
- Docker environment blocks in your compose file or orchestrator config
- Cloud provider secret managers (AWS Secrets Manager, GCP Secret Manager, etc.)

The nice thing about our approach is that the application code doesn't need to change between environments. It always reads from environment variables. The only difference is how those variables get set: a `.env` file in development, platform-managed secrets in production.

## Fail fast

If there's one rule I'd drill into every team, it's this: fail immediately and loudly at startup if any required configuration is missing or invalid. A crash at startup with a clear error message ("JWT_SECRET environment variable is not set") is so much better than a crash ten minutes later when the first authentication request comes in and you discover there's no JWT secret.

This is exactly why our `Config::from_env()` function deserializes and validates eagerly. By the time the application starts accepting requests, we know that all configuration is present, correctly typed, and passes validation. There are no latent configuration bugs waiting to surprise you at 2am on a Saturday. With that foundation in place, let's look at how we handle errors across the rest of our application.
