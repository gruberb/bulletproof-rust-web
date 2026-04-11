# Background Jobs and Task Management

If you've built any web application beyond a toy project, you've run into this: some work just doesn't belong in the request-response cycle. Sending confirmation emails, processing uploaded files, generating reports, retrying failed webhook deliveries, cleaning up expired sessions. All of these would slow down our HTTP responses if we tried to do them inline. In this chapter, we'll walk through the patterns for managing background work in a Tokio-based application, starting with simple fire-and-forget tasks and working our way up to persistent, retry-aware job queues.

## In-process task spawning

The simplest way to run background work is `tokio::spawn`. It creates a new async task that runs independently of whatever spawned it. The spawning code doesn't wait for the spawned task to finish unless you explicitly `.await` the returned `JoinHandle`.

```rust
async fn create_user(
    State(state): State<AppState>,
    ValidatedJson(payload): ValidatedJson<CreateUserDto>,
) -> AppResult<(StatusCode, Json<UserResponse>)> {
    let user = state.user_service.register(payload).await?;

    // Send the welcome email in the background.
    // The HTTP response does not wait for this to finish.
    let email_client = state.email_client.clone();
    let email = user.email().clone();
    tokio::spawn(async move {
        if let Err(e) = email_client.send_welcome(&email).await {
            tracing::error!(error = ?e, "failed to send welcome email");
        }
    });

    Ok((StatusCode::CREATED, Json(user.into())))
}
```

This works well for non-critical side effects where the occasional failure is fine (our user still gets created even if the email fails). But there are some important caveats you should know about.

If the spawned task panics, that panic gets captured in the `JoinHandle`. If you don't await the handle, the panic is silently swallowed, which can be really confusing when you're debugging in production. You should either await the handle or use a `JoinSet` that reports errors back to you.

Here's the bigger problem, though: if your process restarts (deployment, crash, OOM kill), any in-progress spawned tasks are just gone. There's no retry, no persistence, no guarantee the work was completed. For operations that *must* eventually succeed, you'll need a persistent job queue, which we'll get to later in this chapter.

## JoinSet for managing groups of tasks

When you have a known set of tasks that all need to complete, `tokio::task::JoinSet` gives us a structured way to spawn them and collect their results:

```rust
use tokio::task::JoinSet;

async fn process_batch(items: Vec<Item>, pool: PgPool) -> anyhow::Result<()> {
    let mut set = JoinSet::new();

    for item in items {
        let pool = pool.clone();
        set.spawn(async move {
            process_item(&pool, &item).await
        });
    }

    // Wait for all tasks to complete.
    while let Some(result) = set.join_next().await {
        match result {
            Ok(Ok(())) => {}
            Ok(Err(e)) => tracing::error!(error = ?e, "item processing failed"),
            Err(e) => tracing::error!(error = ?e, "task panicked"),
        }
    }

    Ok(())
}
```

`JoinSet` is also handy for limiting concurrency. You can check `set.len()` before spawning new tasks and wait for one to complete if you're at capacity. And when the `JoinSet` is dropped, all tasks in it are aborted, which makes it a natural fit for shutdown scenarios where you want to cancel whatever work is left.

## Periodic tasks

A lot of applications need recurring background work: cache cleanup every 5 minutes, metrics reporting every 30 seconds, health pings every 10 seconds. You get the idea. Tokio's `interval` is what we reach for here:

```rust
use tokio::time::{interval, Duration};

async fn run_cache_cleanup(
    pool: PgPool,
    shutdown: CancellationToken,
) {
    let mut ticker = interval(Duration::from_secs(300));

    loop {
        tokio::select! {
            _ = ticker.tick() => {
                if let Err(e) = cleanup_expired_sessions(&pool).await {
                    tracing::error!(error = ?e, "session cleanup failed");
                }
            }
            _ = shutdown.cancelled() => {
                tracing::info!("cache cleanup shutting down");
                return;
            }
        }
    }
}
```

One thing that tripped me up early on: `interval` has a `MissedTickBehavior` setting that controls what happens when your task takes longer than the interval period. The default is `Burst`, which fires missed ticks immediately to catch up. For most background maintenance tasks, `MissedTickBehavior::Skip` is what you actually want, because you'd rather skip the ticks that were missed while the previous run was still going than pile them all up at once.

```rust
let mut ticker = interval(Duration::from_secs(300));
ticker.set_missed_tick_behavior(tokio::time::MissedTickBehavior::Skip);
```

## Integrating background workers with Axum

In practice, you'll start your background workers alongside the HTTP server, sharing the database pool and configuration through the same `AppState` or by cloning the pool directly. The [Graceful Shutdown](./graceful-shutdown.md) chapter shows the full pattern for spawning multiple subsystems and coordinating their lifecycle.

Let me walk through the key integration points:

**Shared state.** Our background worker needs the same database pool, configuration, and possibly the same service instances as the HTTP handlers. Clone these from the same source at startup.

**Coordinated shutdown.** The worker needs a `CancellationToken` so it can stop gracefully when the application shuts down. You want the worker to finish its current job before exiting, rather than abandoning work mid-flight.

**Error isolation.** A panicking background task shouldn't bring down the HTTP server. By running the worker in a separate `tokio::spawn`, its failures stay isolated. Log the error and, depending on your requirements, either restart the worker or let it stay down.

## Persistent job queues

For work that must survive process restarts, you need a persistent queue backed by a database or a message broker. The good news is that several mature Rust crates already solve this problem for us:

**[apalis](https://github.com/geofmureithi/apalis)** is a type-safe, extensible background processing library that supports multiple storage backends (PostgreSQL, Redis, SQLite) and includes built-in monitoring, metrics, and graceful shutdown. It works with both Tokio and other async runtimes.

**[fang](https://github.com/ayrat555/fang)** runs each worker in a separate Tokio task with automatic restart on panic. It supports scheduled jobs via cron expressions, custom retry backoff modes, and unique job deduplication.

**[backie](https://github.com/rafaelcaricio/backie)** provides async persistent task processing backed by PostgreSQL, designed for horizontal scaling where workers don't need to be in the same process as the queue producer.

The general pattern across all of these is the same, and once you see it, it's pretty intuitive: your HTTP handler (or service) enqueues a job by inserting a row into a jobs table or publishing to a message broker. A separate worker process (or a background task in the same process) polls for pending jobs, processes them, and marks them as completed or failed. Failed jobs get retried with configurable backoff, and after a maximum number of retries they move to a dead-letter state where someone can inspect them manually.

## Error handling in background tasks

This is where things get interesting, because unlike HTTP handlers, background tasks have no client to return errors to. When a background task fails, you need a strategy for making that failure visible and actionable. Here's what I've found works well:

**Log the error with context.** Use `tracing::error!` with structured fields so the failure shows up in your log aggregation system and can be correlated with the original request that enqueued the work. Without this, you'll be flying blind when something goes wrong at 2 AM.

**Retry with backoff.** For transient failures (network timeouts, temporary database unavailability), retry the operation after an increasing delay. Most persistent job queue crates handle this automatically, so you don't need to build it yourself.

**Dead-letter after exhausting retries.** If a job fails repeatedly, move it to a dead-letter table or queue where it can be inspected and either fixed or discarded manually. Don't retry indefinitely, because a permanently failing job will consume worker capacity and delay everything else in the queue.

**Emit metrics.** Track job success rates, processing duration, queue depth, and retry counts. These metrics are your early warning system for problems that are building up before they become visible to users. In my experience, queue depth trending upward is almost always the first sign that something is going sideways.
