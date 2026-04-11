# Message Passing and Channel Patterns

At some point, your async tasks need to talk to each other. When that moment arrives, you basically have two options: shared state protected by a lock, or message passing through channels. Both have their place. But in my experience, channels are the better default for most web application work because they naturally enforce sequential processing, make ownership transfer explicit, and play nicely with `tokio::select!`.

In this chapter, we'll walk through the channel primitives Tokio gives us, the patterns they enable, and how to combine them into an actor/service-worker model that I've found underpins most well-structured concurrent Rust applications.

## The three channel types

Tokio gives us three channel types, and each one is designed for a different communication shape. Let's look at what they do and when you'd reach for each one.

### mpsc: many producers, single consumer

`mpsc` is the workhorse. Multiple tasks can send messages into it, and one task receives and processes them in order. If you're building command queues, work dispatching, or really any coordination pattern, this is probably where you'll start.

```rust
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::channel::<Command>(32); // bounded, capacity 32

// Senders can be cloned and shared across tasks.
let tx2 = tx.clone();

// The receiver processes commands one at a time.
while let Some(cmd) = rx.recv().await {
    process(cmd).await;
}
```

One thing that really matters in production is whether you pick a bounded or unbounded channel. A **bounded channel** (created with `mpsc::channel(capacity)`) gives you natural backpressure: when the channel is full, `send().await` blocks the sender until the receiver catches up. This prevents a fast producer from overwhelming a slow consumer with an ever-growing queue. An **unbounded channel** (created with `mpsc::unbounded_channel()`) never blocks the sender, but it can eat unbounded memory if the receiver falls behind. I'd recommend bounded channels by default, and only reaching for unbounded when you have a specific reason.

### oneshot: single-shot request-response

You'll reach for `oneshot` when you need exactly one response to a request. The pattern looks like this: you create a oneshot channel, send the `Sender` half alongside your command, and then await the `Receiver` to get the result back.

```rust
use tokio::sync::oneshot;

enum Command {
    Get {
        key: String,
        reply: oneshot::Sender<Option<String>>,
    },
    Set {
        key: String,
        value: String,
    },
}

// Caller side: send a command and wait for the response.
async fn get_value(tx: &mpsc::Sender<Command>, key: String) -> Option<String> {
    let (reply_tx, reply_rx) = oneshot::channel();
    tx.send(Command::Get { key, reply: reply_tx }).await.ok()?;
    reply_rx.await.ok()?
}
```

This is how you build request-response communication over an mpsc command channel. The caller creates a oneshot pair, bundles the `Sender` into the command, and awaits the `Receiver`. On the other end, the processing task receives the command, does its work, and sends the result back through the oneshot. It might seem like a lot of ceremony at first, but you'll find it becomes second nature quickly.

### broadcast: one-to-many fan-out

`broadcast` channels let one sender deliver every message to all active receivers. Each receiver gets its own copy of every message sent after it subscribed.

```rust
use tokio::sync::broadcast;

let (tx, _rx) = broadcast::channel::<Event>(100);

// Each subscriber gets a receiver by calling subscribe().
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// Both rx1 and rx2 will receive this event.
tx.send(Event::CertificateReady { id: cert_id })?;
```

Broadcast channels are great for distributing events, invalidation signals, or configuration changes to multiple consumers. One thing to watch out for: if a receiver falls behind, it'll see a `RecvError::Lagged(n)` error telling you how many messages it missed. You'll want to make sure your consumers handle that case gracefully.

## The actor/service-worker pattern

Now we get to the pattern I'm most excited about. The actor (or service-worker) model is the most important thing channels enable, and once you see it, you'll start using it everywhere. The idea is simple: an actor is a task that owns some state, receives commands through a channel, and processes them one at a time. External code interacts with the actor through a client struct that wraps the channel sender and gives you a nice typed async API.

Let's look at a complete example of a simple cache actor:

```rust
use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

// The commands the actor understands.
enum CacheCommand {
    Get {
        key: String,
        reply: oneshot::Sender<Option<String>>,
    },
    Set {
        key: String,
        value: String,
    },
    Invalidate {
        key: String,
    },
}

// The client that external code uses. It hides the channel details.
#[derive(Clone)]
pub struct CacheClient {
    sender: mpsc::Sender<CacheCommand>,
}

impl CacheClient {
    pub async fn get(&self, key: &str) -> Option<String> {
        let (tx, rx) = oneshot::channel();
        self.sender
            .send(CacheCommand::Get {
                key: key.to_string(),
                reply: tx,
            })
            .await
            .ok()?;
        rx.await.ok()?
    }

    pub async fn set(&self, key: String, value: String) {
        let _ = self
            .sender
            .send(CacheCommand::Set { key, value })
            .await;
    }

    pub async fn invalidate(&self, key: &str) {
        let _ = self
            .sender
            .send(CacheCommand::Invalidate {
                key: key.to_string(),
            })
            .await;
    }
}

// Spawn the actor and return the client.
pub fn spawn_cache(shutdown: CancellationToken) -> CacheClient {
    let (tx, rx) = mpsc::channel(64);
    tokio::spawn(run_cache_actor(rx, shutdown));
    CacheClient { sender: tx }
}

async fn run_cache_actor(
    mut commands: mpsc::Receiver<CacheCommand>,
    shutdown: CancellationToken,
) {
    let mut store: HashMap<String, String> = HashMap::new();

    loop {
        let cmd = tokio::select! {
            cmd = commands.recv() => match cmd {
                Some(c) => c,
                None => return, // All clients dropped.
            },
            _ = shutdown.cancelled() => return,
        };

        match cmd {
            CacheCommand::Get { key, reply } => {
                let _ = reply.send(store.get(&key).cloned());
            }
            CacheCommand::Set { key, value } => {
                store.insert(key, value);
            }
            CacheCommand::Invalidate { key } => {
                store.remove(&key);
            }
        }
    }
}
```

So why is this pattern such a natural fit for production code? Let me walk through what makes it work so well.

**No locking.** The HashMap is owned exclusively by the actor task. No `Mutex`, no `RwLock`, no contention. The mpsc channel serializes access for us naturally.

**Cancellation resilient.** Each command is processed to completion before the next `select!` iteration. There's no risk of dropping a future in the middle of a state mutation, which is a subtle bug that can haunt you with other approaches.

**Testable.** You can test the actor by creating a channel, sending commands, and asserting on the responses. No need to reason about concurrent timing, which is a huge win.

**Composable.** The `CacheClient` can live in your `AppState` and get extracted in Axum handlers just like any other shared dependency. It's `Clone` and `Send`, which is all Axum needs.

## When to use channels vs. shared state

You might wonder when to use channels versus just slapping a `Mutex` on your state. Here's how I think about it.

Channels and the actor pattern are the right tool when:

- The state needs sequential processing (commands must not interleave)
- You want to encapsulate state behind an async API
- The access pattern involves both reads and writes that need to be coordinated
- You're already using `tokio::select!` to multiplex multiple event sources

Shared state (`Arc<RwLock<T>>`, `Arc<DashMap<K, V>>`) is the right tool when:

- The access pattern is overwhelmingly reads with rare writes
- You need many tasks to read concurrently without serialization
- The state is simple enough that a lock doesn't introduce significant complexity
- You don't need to coordinate the state with other event sources in a `select!` loop

One more thing before we move on: for application configuration, a `watch` channel is also worth knowing about. It holds a single value that any number of receivers can observe, and receivers always see the most recent value. This is really useful for runtime-reloadable configuration or health status that many tasks need to check. We'll see more of this kind of coordination in the chapters ahead.
