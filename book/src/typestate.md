# Typestate and Advanced Type System Patterns

In the [Domain Modeling](./domain-modeling.md) chapter, we used newtypes to enforce invariants on individual values: an `Email` can only be constructed from a valid email string, and once you have one, its validity is guaranteed. Now we're going to take that idea further. Instead of encoding what a value **is**, we'll encode what state an entity is **in**, and use the type system to control which operations are available in each state.

The payoff? Invalid state transitions become compile errors rather than runtime bugs.

## The typestate pattern

The core idea is straightforward: we represent an entity's state as a type parameter, and then define methods that are only available when the entity is in the right state. When you transition to a new state, the old value gets consumed and you get back a new one with a different type parameter. Let's look at what this looks like in practice.

```rust
use std::marker::PhantomData;

// States are zero-sized types. They exist only at compile time.
pub struct Created;
pub struct Paid;
pub struct Shipped;

pub struct Order<State> {
    id: Uuid,
    customer_id: Uuid,
    total_cents: i64,
    _state: PhantomData<State>,
}

impl Order<Created> {
    pub fn new(customer_id: Uuid, total_cents: i64) -> Self {
        Self {
            id: Uuid::new_v4(),
            customer_id,
            total_cents,
            _state: PhantomData,
        }
    }

    /// Pay for the order. Consumes the Created order
    /// and returns a Paid order.
    pub fn pay(self, payment_id: String) -> Order<Paid> {
        tracing::info!(order_id = %self.id, "order paid");
        Order {
            id: self.id,
            customer_id: self.customer_id,
            total_cents: self.total_cents,
            _state: PhantomData,
        }
    }
}

impl Order<Paid> {
    /// Ship the order. Only available on paid orders.
    pub fn ship(self, tracking_number: String) -> Order<Shipped> {
        tracing::info!(order_id = %self.id, "order shipped");
        Order {
            id: self.id,
            customer_id: self.customer_id,
            total_cents: self.total_cents,
            _state: PhantomData,
        }
    }

    /// Refund the order. Only available on paid (not yet shipped) orders.
    pub fn refund(self) -> Order<Created> {
        tracing::info!(order_id = %self.id, "order refunded");
        Order {
            id: self.id,
            customer_id: self.customer_id,
            total_cents: self.total_cents,
            _state: PhantomData,
        }
    }
}

impl Order<Shipped> {
    pub fn tracking_number(&self) -> &str {
        // In a real implementation, this would be stored in the struct.
        "tracking info"
    }
}
```

With this design, the happy path compiles just fine:

```rust
let order = Order::new(customer_id, 4999);
let paid_order = order.pay("pay_123".into());
let shipped_order = paid_order.ship("TRACK456".into());
```

But try to skip a step, and you won't get past the compiler:

```rust
let order = Order::new(customer_id, 4999);
// Error: no method named `ship` found for `Order<Created>`
let shipped = order.ship("TRACK456".into());
```

The compiler catches the invalid transition because `ship()` is only defined on `Order<Paid>`, not on `Order<Created>`. You can't ship an unpaid order, and the type system enforces this without any runtime checks. No `if` statements, no panics, just a compile error that tells you exactly what went wrong.

## Real-world application: connection lifecycle

Where this pattern really shines is protocol implementations where a connection moves through distinct phases with different capabilities. I worked on the [mozilla-services/autopush-rs](https://github.com/mozilla-services/autopush-rs) project, which uses this approach for WebSocket connections: an `UnidentifiedClient` handles the initial handshake, and when a valid Hello message arrives, it transitions to a `WebPushClient` that has access to the user's subscription data and notification channels.

Here's a simplified version of that pattern:

```rust
pub struct Unauthenticated;
pub struct Authenticated;

pub struct Connection<State> {
    stream: WebSocketStream,
    remote_addr: SocketAddr,
    _state: PhantomData<State>,
}

impl Connection<Unauthenticated> {
    pub fn new(stream: WebSocketStream, remote_addr: SocketAddr) -> Self {
        Self {
            stream,
            remote_addr,
            _state: PhantomData,
        }
    }

    /// Perform the authentication handshake.
    /// Consumes the unauthenticated connection and returns
    /// an authenticated one, or an error if auth fails.
    pub async fn authenticate(
        mut self,
        timeout: Duration,
    ) -> Result<(Connection<Authenticated>, UserId), AuthError> {
        let hello = tokio::time::timeout(timeout, self.read_hello())
            .await
            .map_err(|_| AuthError::Timeout)?
            .map_err(AuthError::Protocol)?;

        let user_id = validate_credentials(&hello)?;

        Ok((
            Connection {
                stream: self.stream,
                remote_addr: self.remote_addr,
                _state: PhantomData,
            },
            user_id,
        ))
    }
}

impl Connection<Authenticated> {
    /// Send a push notification. Only available on authenticated connections.
    pub async fn send_notification(&mut self, notif: &Notification) -> Result<(), SendError> {
        self.stream.send(notif.to_message()).await?;
        Ok(())
    }

    /// Graceful close with notification draining.
    pub async fn shutdown(mut self, reason: CloseReason) {
        // Drain any remaining notifications before closing.
        self.drain_pending_notifications().await;
        self.stream.close(reason.into()).await.ok();
    }
}
```

The key insight from autopush-rs is that error handling differs between the two phases. During the unauthenticated phase, a protocol error just disconnects the client (there's nothing to clean up). But during the authenticated phase, a protocol error needs to trigger a graceful shutdown that drains pending notifications before closing. The typestate pattern makes this distinction structural: the `shutdown` method with notification draining only exists on `Connection<Authenticated>`. You literally can't call it in the wrong phase.

## The builder pattern as a degenerate typestate

You might already be using a simplified version of typestate without realizing it: the builder pattern with mandatory fields. By using different type parameters for each stage of the builder, we can ensure that required fields are set before `build()` becomes available.

```rust
pub struct NoAddr;
pub struct HasAddr;

pub struct ServerBuilder<AddrState> {
    addr: Option<SocketAddr>,
    max_connections: usize,
    _state: PhantomData<AddrState>,
}

impl ServerBuilder<NoAddr> {
    pub fn new() -> Self {
        Self {
            addr: None,
            max_connections: 100,
            _state: PhantomData,
        }
    }

    pub fn bind(self, addr: SocketAddr) -> ServerBuilder<HasAddr> {
        ServerBuilder {
            addr: Some(addr),
            max_connections: self.max_connections,
            _state: PhantomData,
        }
    }
}

impl ServerBuilder<HasAddr> {
    /// build() is only available after bind() has been called.
    pub fn build(self) -> Server {
        Server {
            addr: self.addr.unwrap(), // Safe: guaranteed by typestate.
            max_connections: self.max_connections,
        }
    }
}

// Optional settings are available in any state.
impl<S> ServerBuilder<S> {
    pub fn max_connections(mut self, n: usize) -> Self {
        self.max_connections = n;
        self
    }
}
```

This guarantees at compile time that you can't call `build()` without first calling `bind()`. If you try, the compiler error is clear: "no method named `build` found for `ServerBuilder<NoAddr>`." I love how Rust turns what would be a runtime panic in most languages into something the compiler catches for you.

## When typestate is worth the complexity

Now, I don't want to leave you with the impression that you should reach for typestate everywhere. It adds complexity to your type signatures and makes some common operations harder (like storing a heterogeneous collection of entities in different states). In my experience, it's worth it when:

- Invalid transitions would cause security vulnerabilities or data corruption
- There are relatively few states with clear, linear transitions
- The code that manages transitions is performance-sensitive enough that you want zero runtime overhead
- The types are used within a single module or subsystem, so the generic parameters don't propagate widely

You'll want to prefer enum-based state machines when:

- You need to store entities in different states together (a `Vec<Order>` where some are paid and some are shipped)
- The number of states is large or the transition graph is complex
- The state needs to be serialized to or deserialized from a database or API response
- The state is determined at runtime by external data that you can't know at compile time

What I've found in practice is that many systems use both: typestate for the core internal logic where transitions are safety-critical, and an enum wrapper for persistence and external interfaces. The typestate enforces correctness in the code that manages transitions, while the enum gives you the flexibility you need for storage and serialization. We'll see more of this when we get to the persistence layer.

## Connecting back to domain modeling

If you're seeing a theme here, you're right. The typestate pattern is a natural extension of the "parse, don't validate" philosophy from the [Domain Modeling](./domain-modeling.md) chapter. Where newtypes make invalid values unrepresentable (`Email` can only hold valid emails), typestate makes invalid transitions unrepresentable (`Order<Created>` has no `ship()` method). Both patterns use Rust's type system to move correctness guarantees from runtime checks to compile-time enforcement, and both pay off most in code where getting it wrong would hurt. In the next chapter, we'll look at how to structure our error types so that when things do go wrong at runtime, we handle it cleanly.
