# Async Pitfalls and Cancellation Safety

If you've written async Rust for any length of time, you've probably hit a bug that made no sense at first. Data disappeared. A stream got corrupted. Everything looked correct, but something silently dropped work on the floor. In my experience, the culprit is almost always cancellation: when a future gets dropped before it finishes, the work it was doing just stops, and your program can end up in a state you never planned for.

We'll walk through the patterns that trip people up here, even experienced Rust developers. We're going to focus on `tokio::select!`, cancellation safety, and the structured concurrency primitives that help you manage concurrent work without losing your mind.

## How `tokio::select!` works

`tokio::select!` is our main tool for waiting on multiple async operations at once. It polls all of its branches, and when the first one completes, it drops the rest and runs the matching handler.

```rust
tokio::select! {
    msg = rx.recv() => {
        // A message arrived from the channel.
        handle_message(msg).await;
    }
    _ = shutdown.cancelled() => {
        // A shutdown signal was received.
        tracing::info!("shutting down");
        return;
    }
}
```

The important thing here is what happens to the branch that loses. When `shutdown.cancelled()` completes first, the `rx.recv()` future gets dropped. For `recv()`, that's totally fine: the channel still holds whatever messages were in its buffer, and the next call to `recv()` picks up where we left off. That property is what we call cancellation safety.

## What "cancellation safe" means

A future is cancellation safe if dropping it partway through doesn't lose data or leave shared state in a broken condition. The Tokio docs explicitly mark each method with whether it's cancellation safe, and I'd strongly recommend checking before you put any operation inside a `select!` branch.

Operations that are cancellation safe include `channel.recv()`, `TcpListener::accept()`, `sleep()`, and `CancellationToken::cancelled()`. Dropping these just means "I stopped waiting," and nothing is lost.

Operations that are **not** cancellation safe include `read_exact()`, `write_all()`, and `read_to_string()`. Think about what happens if `write_all` has written 50 bytes of a 100-byte message when `select!` drops it. Those 50 bytes are already on the wire, and you have no way to know how many were sent. The next attempt to write starts the full message again, corrupting the stream.

Let's look at a concrete example of this problem:

```rust
// BROKEN: buf is not cancel-safe inside select!
let mut buf = vec![0u8; 1024];
loop {
    tokio::select! {
        // If this branch loses, we lose track of how many bytes
        // were read. The next iteration starts over with an empty buf.
        result = socket.read_exact(&mut buf) => {
            process(&buf).await;
        }
        _ = shutdown.cancelled() => {
            return;
        }
    }
}
```

The fix is to move the non-cancel-safe operation out of `select!` entirely, so it always runs to completion. We keep only cancel-safe operations (like channel receives) inside our `select!` branches:

```rust
loop {
    tokio::select! {
        result = rx.recv() => {
            // Only cancel-safe operations in select! branches.
            if let Some(data) = result {
                process(&data).await;
            }
        }
        _ = shutdown.cancelled() => {
            return;
        }
    }
}
```

## The actor model as a cancellation-resilient pattern

What I've found to be the most reliable way to avoid cancellation bugs is to structure your concurrent components as actors. Each actor runs a loop that receives a message, processes it to completion, and then goes back to waiting for the next one. Because `select!` only runs between loop iterations (when the actor is waiting for input, not in the middle of doing work), there's nothing to cancel partway through.

```rust
async fn run_worker(
    mut commands: mpsc::Receiver<Command>,
    shutdown: CancellationToken,
) {
    loop {
        let command = tokio::select! {
            cmd = commands.recv() => {
                match cmd {
                    Some(c) => c,
                    None => return, // Channel closed, all senders dropped.
                }
            }
            _ = shutdown.cancelled() => {
                tracing::info!("worker shutting down");
                return;
            }
        };

        // This runs to completion before the next select! iteration.
        // No cancellation risk here.
        process_command(command).await;
    }
}
```

This is the pattern we used extensively in [topos-protocol/topos](https://github.com/topos-protocol/topos), where each subsystem (broadcast, synchronizer, API) runs an event loop that multiplexes commands, shutdown signals, and timers through `select!`, but processes each event to completion before going back to the top of the loop. It might seem like extra structure at first, but it pays for itself immediately in bugs you never have to chase.

## Structured concurrency with CancellationToken

Once your application has multiple concurrent subsystems (an HTTP server, a background worker, a metrics reporter), you need a way to tell all of them to shut down and then wait for them to actually finish. This is where `tokio_util::sync::CancellationToken` comes in.

```rust
use tokio_util::sync::CancellationToken;

let root_token = CancellationToken::new();

// Each subsystem gets a child token. Cancelling the root
// cancels all children, but cancelling a child does not
// affect siblings or the parent.
let http_token = root_token.child_token();
let worker_token = root_token.child_token();
let metrics_token = root_token.child_token();

// Spawn subsystems with their tokens.
let http_handle = tokio::spawn(run_http_server(http_token));
let worker_handle = tokio::spawn(run_background_worker(worker_token));
let metrics_handle = tokio::spawn(run_metrics_reporter(metrics_token));

// When a shutdown signal arrives, cancel the root.
wait_for_signal().await;
root_token.cancel();

// Wait for all subsystems to finish their cleanup.
let _ = tokio::join!(http_handle, worker_handle, metrics_handle);
tracing::info!("all subsystems shut down cleanly");
```

The parent-child relationship ensures that shutdown propagates downward. Each subsystem checks `token.cancelled()` in its event loop (via `select!`) and starts its cleanup when it fires. The parent waits for all handles to resolve, so we know no work gets abandoned.

We'll dig into this pattern much more in the [Graceful Shutdown](./graceful-shutdown.md) chapter, where we'll build a full shutdown sequence for a real application.

## Common mistakes

**Using `select!` when you mean `join!`.** You might wonder why your second operation never finishes. If you want to run two operations concurrently and wait for both, use `tokio::join!` or `tokio::try_join!`. `select!` returns when the first one completes and drops the other. I've seen this mix-up cause lost work more times than I can count.

**Forgetting that `select!` in a loop needs all branches to be cancel-safe.** Each time the loop iterates, the futures from the previous iteration are dropped. If one of those futures wasn't cancel-safe, data can silently disappear between iterations. The actor pattern we looked at above avoids this by keeping all `select!` branches limited to cancel-safe operations (channel receives, timer ticks, cancellation signals).

**Holding a MutexGuard across an await point.** With `std::sync::Mutex`, holding the guard across an `.await` can block the entire runtime thread. With `tokio::sync::Mutex`, it's safe but can lead to long hold times if the future inside the critical section takes a while. In both cases, I'd recommend the actor/channel pattern instead: send a message to a task that owns the state, rather than locking shared state from multiple tasks.

**Not handling the `else` branch in `select!`.** If all channels in a `select!` are closed (every `recv()` returns `None`), and there's no `else` branch, the `select!` will panic. In production code, always either handle the `None` case explicitly or make sure at least one branch (like a cancellation token) can never close.

## `tokio::pin!` and when you need it

Some futures need to be pinned before you can use them in `select!`, because `select!` requires that its branch futures implement `Unpin` or are pinned. You'll most often run into this when you store a future in a variable and want to reuse it across loop iterations:

```rust
let sleep_future = tokio::time::sleep(Duration::from_secs(30));
tokio::pin!(sleep_future);

loop {
    tokio::select! {
        _ = &mut sleep_future => {
            tracing::info!("timeout reached");
            break;
        }
        msg = rx.recv() => {
            // Process message. The sleep future continues
            // from where it left off on the next iteration.
        }
    }
}
```

Without `tokio::pin!`, the compiler will reject this with an error about `Unpin` bounds. If you haven't seen that error yet, don't worry; you will soon. The pin ensures the future's memory location stays stable across await points, which is a requirement of the `select!` macro's polling mechanism. It's one of those things that feels confusing the first time but becomes second nature once you've seen the pattern a few times.
