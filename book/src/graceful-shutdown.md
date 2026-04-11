# Graceful Shutdown and Lifecycle Management

In the [Deployment](./deployment.md) chapter, we looked at how to shut down a single Axum server cleanly when a SIGTERM arrives. That covers the simplest case, but production applications are rarely just an HTTP server. You've got background workers processing jobs, open WebSocket connections draining messages, metrics reporters flushing data, and database pools that need to close cleanly. If any of these subsystems is still doing work at the moment the process exits, you end up with lost data, broken connections, or inconsistent state.

In this chapter, we'll coordinate shutdown across multiple concurrent subsystems so that each one finishes its in-flight work in the right order before the process exits.

## CancellationToken as the coordination primitive

If you've tried to coordinate shutdown before, you might have reached for oneshot channels or `Arc<AtomicBool>` flags. Those work, but they get messy fast once you have more than two subsystems. The `tokio_util::sync::CancellationToken` is a much better fit here. It gives you a `cancelled()` future that resolves when the token is cancelled, and it supports parent-child relationships where cancelling a parent automatically cancels all children.

```rust
use tokio_util::sync::CancellationToken;

// Create a root token for the entire application.
let root_token = CancellationToken::new();

// Each subsystem gets a child token.
let http_token = root_token.child_token();
let worker_token = root_token.child_token();
let telemetry_token = root_token.child_token();
```

When a shutdown signal arrives, we cancel the root token, and every subsystem sees it through its child token. The token is cancellation-aware and integrates naturally with `tokio::select!`, which makes the code much easier to follow than the alternatives.

## A multi-subsystem application

Let's look at a realistic `main` function that starts three subsystems and coordinates their shutdown:

```rust
use tokio_util::sync::CancellationToken;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    init_tracing(&config.log_level);

    let pool = create_pool(config.database_url.expose_secret()).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;

    let root_token = CancellationToken::new();

    // Spawn the HTTP server.
    let http_token = root_token.child_token();
    let http_handle = tokio::spawn({
        let pool = pool.clone();
        let config = config.clone();
        async move {
            let app = build_router(config, pool);
            let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
                .await
                .expect("failed to bind");
            axum::serve(listener, app)
                .with_graceful_shutdown(http_token.cancelled_owned())
                .await
                .expect("server error");
        }
    });

    // Spawn a background job worker.
    let worker_token = root_token.child_token();
    let worker_handle = tokio::spawn({
        let pool = pool.clone();
        async move {
            run_job_worker(pool, worker_token).await;
        }
    });

    // Spawn a metrics reporter.
    let metrics_token = root_token.child_token();
    let metrics_handle = tokio::spawn(async move {
        run_metrics_reporter(metrics_token).await;
    });

    // Wait for the shutdown signal, then cancel everything.
    wait_for_signal().await;
    tracing::info!("shutdown signal received");
    root_token.cancel();

    // Wait for all subsystems to finish.
    let _ = tokio::join!(http_handle, worker_handle, metrics_handle);

    // Close the database pool after all subsystems have stopped.
    pool.close().await;

    tracing::info!("shutdown complete");
    Ok(())
}
```

The key thing to notice here is that the database pool is closed **after** all subsystems have finished. Why? Because those subsystems may still be executing queries during their drain phase. If you close the pool first, those queries will fail. This is what I mean by ordered teardown, and it's worth looking at more closely.

## Ordered teardown

The general rule is: release resources in the reverse order of their dependency chain. If our HTTP handlers depend on the database pool, and the metrics reporter depends on the HTTP server being active, our teardown order should be:

1. Stop accepting new HTTP connections (Axum handles this when the shutdown future resolves)
2. Drain in-flight HTTP requests to completion
3. Stop the background worker (it finishes its current job, then exits)
4. Flush the metrics reporter (it sends any buffered data, then exits)
5. Close the database pool

In practice, steps 2 through 4 happen concurrently (via `tokio::join!`) because each subsystem manages its own drain independently. All we need to make sure of is that shared resources like the pool aren't closed until every consumer has stopped.

## Managing dynamic task pools with FuturesUnordered

Some subsystems don't have a fixed number of tasks. You might have one task per active WebSocket connection, one per in-flight background job, one per connected gRPC stream. The count changes constantly. For these situations, `FuturesUnordered` is a great tool because it gives you a stream of task completions that you can drain during shutdown.

```rust
use futures::stream::FuturesUnordered;
use futures::StreamExt;

async fn run_connection_manager(
    mut new_connections: mpsc::Receiver<TcpStream>,
    shutdown: CancellationToken,
) {
    let mut active_tasks = FuturesUnordered::new();

    loop {
        tokio::select! {
            // Accept new connections while running.
            conn = new_connections.recv() => {
                if let Some(stream) = conn {
                    let token = shutdown.child_token();
                    active_tasks.push(tokio::spawn(
                        handle_connection(stream, token)
                    ));
                }
            }

            // Reap completed tasks to free resources.
            Some(result) = active_tasks.next() => {
                if let Err(e) = result {
                    tracing::error!(error = ?e, "connection task panicked");
                }
            }

            // On shutdown, stop accepting new connections
            // and drain the remaining tasks.
            _ = shutdown.cancelled() => {
                tracing::info!(
                    active = active_tasks.len(),
                    "shutting down, draining active connections"
                );
                break;
            }
        }
    }

    // Drain: wait for all active connections to finish.
    while let Some(result) = active_tasks.next().await {
        if let Err(e) = result {
            tracing::error!(error = ?e, "connection task panicked during drain");
        }
    }

    tracing::info!("all connections drained");
}
```

I want to highlight the two-phase structure here, because it's a pattern you'll use a lot. During normal operation, the `select!` loop accepts new connections, reaps completed ones, and watches for the shutdown signal. Once shutdown fires, the loop breaks and we enter the drain phase, which just waits for all remaining tasks to complete. Each connection task receives a child cancellation token so it can clean up its own resources (flushing buffers, sending close frames) before returning.

## Connection cleanup

For long-lived connections like WebSockets, the cleanup sequence inside each connection task matters more than you might expect. The [mozilla-services/autopush-rs](https://github.com/mozilla-services/autopush-rs) project is a good example of doing this thoroughly: when a WebSocket connection shuts down, it disconnects from the client registry, drains any remaining server notifications through `on_server_notif_shutdown()`, calls `client.shutdown()` with the error details, and then closes the session with the appropriate WebSocket close reason. This reverse-order cleanup makes sure no notifications are lost between the shutdown decision and the actual connection close.

The principle I'd encourage you to follow: when a connection or task shuts down, finish all pending outbound work (flush write buffers, send remaining notifications, acknowledge pending messages) before closing the underlying transport. It might seem tedious to think through all these steps, but in my experience, the bugs you get from skipping cleanup are much harder to debug than the cleanup code itself.

## Testing shutdown behavior

You might wonder how to test any of this. Shutdown logic is tricky because it involves timing, concurrency, and edge cases that only surface under specific orderings. What I've found works well is using `tokio::time::timeout` to make sure your shutdown sequence completes within a reasonable duration:

```rust
#[tokio::test]
async fn shutdown_completes_within_timeout() {
    let token = CancellationToken::new();
    let handle = tokio::spawn(run_my_subsystem(token.clone()));

    // Give it a moment to start up.
    tokio::time::sleep(Duration::from_millis(100)).await;

    // Trigger shutdown.
    token.cancel();

    // It should finish within 5 seconds.
    let result = tokio::time::timeout(
        Duration::from_secs(5),
        handle,
    ).await;

    assert!(result.is_ok(), "shutdown did not complete within timeout");
}
```

If this test hangs, you know your shutdown path has a bug. Something is waiting on a channel that will never send, or a task isn't checking its cancellation token. It's a simple test, but it catches real problems. And once you have it in place, you'll be much more confident making changes to your shutdown logic later on.
