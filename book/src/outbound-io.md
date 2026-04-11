# Outbound HTTP and Service Resilience

So far we've been looking at our service as a server, receiving requests and sending responses. But in practice, most services are also clients. Our web server calls payment providers, sends webhooks, queries internal APIs, fetches data from third parties, and pushes messages to queues. I've seen outages that had nothing to do with how we handled inbound traffic. The problem was how we called out. If a downstream dependency gets slow and you don't have the right guardrails, it'll take your whole service down with it.

In this chapter, we'll walk through the patterns that keep outbound HTTP calls from ruining your day: timeouts, retries, backoff, idempotency, circuit breaking, and trace context propagation.

## reqwest as the HTTP client

The `reqwest` crate is the go-to HTTP client in the Rust ecosystem. It's async-native, handles TLS, supports connection pooling, and plays nicely with the Tokio runtime. One thing that trips people up early on: you want to create a single `Client` instance at startup and share it across your application. The `Client` manages an internal connection pool, so if you're creating a new one for every request, you're throwing away all those pooling benefits.

```rust
use reqwest::Client;
use std::time::Duration;

pub fn create_http_client() -> Client {
    Client::builder()
        .timeout(Duration::from_secs(10))
        .connect_timeout(Duration::from_secs(3))
        .pool_max_idle_per_host(10)
        .build()
        .expect("failed to create HTTP client")
}
```

We'll store the client in our `AppState` and extract it in handlers or services that need to make outbound calls. The same client can serve multiple concurrent requests safely, which is exactly what we want.

## Timeouts at every level

If I had to pick one resilience mechanism for outbound calls, it would be timeouts. Without them, a slow or unresponsive downstream service can hold your connections open indefinitely. Eventually you'll exhaust your connection pool, your file descriptors, or your request-handling capacity. I've watched this happen in production, and it's not fun.

There are three levels of timeout you should think about:

**Connect timeout** controls how long we wait to establish a TCP connection. If the remote server is unreachable or the network is congested, you want to fail fast rather than waiting for the OS-level default (which can be 120 seconds on Linux). Three to five seconds is a reasonable starting point.

**Per-request timeout** caps the total time for a single HTTP request, including connection, TLS handshake, sending the request, and receiving the full response. Ten to thirty seconds is typical, but it really depends on how fast you expect the downstream to respond.

**Overall operation timeout** is the deadline for the entire operation, including retries. If your business logic says "fetch the payment status within 30 seconds," that 30-second budget covers all retry attempts, not just one. Use `tokio::time::timeout` around the entire retry loop, not around individual attempts.

```rust
use tokio::time::timeout;

async fn fetch_with_deadline(
    client: &Client,
    url: &str,
    deadline: Duration,
) -> anyhow::Result<String> {
    timeout(deadline, async {
        // Individual attempts have their own timeout (via client config).
        // The outer timeout caps the total time including retries.
        let response = client.get(url).send().await?.error_for_status()?;
        Ok(response.text().await?)
    })
    .await
    .map_err(|_| anyhow::anyhow!("operation timed out after {:?}", deadline))?
}
```

## Retries and backoff

Transient failures are just a fact of life in distributed systems. Network blips, brief service restarts, load-induced timeouts, all of these produce errors that would succeed if you tried again a moment later. So a well-behaved outbound client should retry these failures, with increasing delays between attempts to avoid piling on.

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn fetch_with_retries(
    client: &Client,
    url: &str,
    max_retries: u32,
) -> anyhow::Result<reqwest::Response> {
    let mut last_error = None;

    for attempt in 0..=max_retries {
        if attempt > 0 {
            // Exponential backoff with jitter to prevent thundering herd.
            let base = Duration::from_millis(100 * 2u64.pow(attempt - 1));
            let jitter = Duration::from_millis(rand::random::<u64>() % 50);
            sleep(base + jitter).await;
        }

        match client.get(url).send().await {
            Ok(resp) if resp.status().is_server_error() => {
                // 5xx errors are retryable.
                last_error = Some(anyhow::anyhow!(
                    "server error: {}", resp.status()
                ));
            }
            Ok(resp) => return Ok(resp),
            Err(e) if e.is_timeout() || e.is_connect() => {
                // Timeouts and connection errors are retryable.
                last_error = Some(e.into());
            }
            Err(e) => return Err(e.into()), // Non-retryable error.
        }
    }

    Err(last_error.unwrap_or_else(|| anyhow::anyhow!("exhausted retries")))
}
```

When you're building a retry strategy, there are a few key decisions to make:

**Which errors to retry.** Retry connection errors, timeouts, and 5xx responses. Don't retry 4xx responses, because the request was wrong, and sending it again won't help. Same goes for errors that indicate a permanent problem, like DNS resolution failure for a domain that doesn't exist.

**Backoff strategy.** Exponential backoff (doubling the delay each attempt) with jitter (adding a small random component) is the standard approach. The jitter is important because it prevents the thundering herd problem, where many clients retry at exactly the same time after an outage and overwhelm the recovering service.

**Maximum attempts.** Three to five retries is typical. More than that usually means the downstream has a sustained outage, and continuing to retry just adds load without improving anything.

## Idempotency of retried operations

Retrying a GET request is safe because GET is idempotent by definition. But retrying a POST that creates a resource? That's a different story. Unless the downstream API supports idempotency keys, a retried POST can create duplicate records, which is probably the last thing you want.

If you're calling an API that supports idempotency keys (Stripe, for example, accepts an `Idempotency-Key` header), the approach is straightforward: generate a UUID for the operation and include it on every attempt:

```rust
let idempotency_key = Uuid::new_v4().to_string();

for attempt in 0..=max_retries {
    let result = client
        .post(url)
        .header("Idempotency-Key", &idempotency_key)
        .json(&payload)
        .send()
        .await;

    // Same retry logic as above.
}
```

What if the downstream API doesn't support idempotency keys? You have two options: design the operation to be naturally idempotent (use PUT with a known ID rather than POST to a collection), or don't retry write operations at all and surface the failure for manual resolution. Neither is perfect, but both are better than silently creating duplicate charges.

## Circuit breaking

You might wonder: if a downstream service is consistently failing, why keep sending requests to it? You're wasting your own resources and adding load to a system that's already struggling. This is where circuit breakers come in. A circuit breaker tracks failure rates and, when they cross a threshold, short-circuits requests for a cool-down period before trying again.

The `tower` ecosystem provides `tower-resilience` with circuit breaker, bulkhead, and retry middleware that you can layer onto a reqwest client or any Tower service. For simpler needs, you can implement a basic circuit breaker with an `AtomicU32` failure counter and a `tokio::time::Instant` for the last-check timestamp.

The three states are:

- **Closed** (normal): requests flow through. Failures increment a counter.
- **Open** (tripped): requests are rejected immediately without calling the downstream. After a timeout, the breaker moves to half-open.
- **Half-open** (probing): a single request is allowed through. If it succeeds, the breaker closes. If it fails, it reopens.

## Propagating trace context

When our service calls another service, we want the trace context (trace ID, span ID, and sampling decision) to travel with the request. This is what lets you correlate logs and spans across service boundaries, and it's what makes distributed tracing actually useful rather than just decoration.

If both services use OpenTelemetry, the trace context is propagated via the `traceparent` HTTP header (W3C Trace Context format). The `reqwest-tracing` or `reqwest-middleware` crates can inject this header automatically, which is nice. If you're using a simpler setup, at minimum propagate your request ID as a custom header so you can correlate logs manually. You'll thank yourself at 2am when you're debugging a production issue.

```rust
let request_id = "req-abc-123"; // from the inbound request's span

let response = client
    .get(url)
    .header("x-request-id", request_id)
    .send()
    .await?;
```

## When not to retry

Not every failure should be retried. This is one of those things that seems obvious when you read it, but in my experience, it's easy to get wrong. Retrying aggressively against a service that's already overloaded just makes the overload worse. Here are the situations where retrying is the wrong call:

- **4xx client errors** (except 429 Too Many Requests, which is explicitly asking you to back off and try later)
- **Errors that indicate a permanent problem** (invalid credentials, resource deleted, schema mismatch)
- **When the operation deadline has expired** (there's no point starting a new attempt if the caller has already given up)
- **When the circuit breaker is open** (the downstream is known to be unhealthy)

The general rule I follow: retry transient failures with backoff, respect rate limits, and fail fast when the problem is structural. Let's look at how we wrap all of this up into something clean.

## Wrapping outbound clients

For downstream services that your application calls frequently, I'd recommend wrapping the raw `reqwest::Client` in a dedicated client struct. This struct encapsulates the base URL, authentication, timeout policy, retry logic, and error mapping all in one place. Think of it as the outbound equivalent of the repository pattern we use for database access.

```rust
pub struct PaymentClient {
    http: Client,
    base_url: String,
    api_key: String,
}

impl PaymentClient {
    pub async fn charge(
        &self,
        amount_cents: i64,
        currency: &str,
        idempotency_key: &str,
    ) -> Result<PaymentResult, PaymentError> {
        let url = format!("{}/v1/charges", self.base_url);

        let response = self.http
            .post(&url)
            .bearer_auth(&self.api_key)
            .header("Idempotency-Key", idempotency_key)
            .json(&serde_json::json!({
                "amount": amount_cents,
                "currency": currency,
            }))
            .send()
            .await
            .map_err(PaymentError::Network)?;

        match response.status() {
            s if s.is_success() => {
                let result = response.json().await.map_err(PaymentError::Parse)?;
                Ok(result)
            }
            StatusCode::UNPROCESSABLE_ENTITY => {
                let body = response.text().await.unwrap_or_default();
                Err(PaymentError::Validation(body))
            }
            s => Err(PaymentError::Upstream(s)),
        }
    }
}
```

This gives our domain services a clean, typed interface for the downstream dependency. It hides the HTTP details, and it gives us a natural place to layer in retries, circuit breaking, and metrics later without touching the calling code. What I've found is that this pattern pays for itself quickly, especially as the number of downstream dependencies grows.
