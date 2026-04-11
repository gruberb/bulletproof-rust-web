# Performance

One of the nice things about choosing Rust and Axum is that you get a really solid performance baseline without doing anything special. Zero-cost abstractions, no garbage collector, and Tokio's async runtime working together means our straightforward Axum application can handle thousands of concurrent connections on modest hardware. That said, it's still possible to leave performance on the table if we're not thoughtful about how we use the tools.

In this chapter, we'll walk through the practices that keep our application fast, and the pitfalls I've seen trip people up.

## Do not block the async runtime

If there's one performance rule you take away from this entire book, let it be this one. Tokio's runtime uses a thread pool to execute async tasks cooperatively. If one task blocks a thread (doing CPU-intensive work, synchronous I/O, or calling a blocking library function), that thread isn't available for other tasks until the blocking work finishes. Since Tokio defaults to one worker thread per CPU core, blocking even a single thread can visibly hurt throughput.

The tricky part is that the symptoms are subtle. Our application handles moderate load just fine, but under higher concurrency we start seeing latency spikes and throughput drops. What's actually happening is that async tasks are queued up, waiting for a thread that's stuck doing something synchronous.

The fix is to move blocking work onto a dedicated thread pool:

```rust
// CPU-intensive work (password hashing, image processing, etc.)
let hash = tokio::task::spawn_blocking(move || {
    hash_password(&password)
})
.await
.context("password hashing task panicked")??;
```

`spawn_blocking` runs the closure on Tokio's blocking thread pool, which is separate from the async worker threads. While it waits for the result, our async task yields, freeing the worker thread to handle other requests.

One thing to keep in mind: `spawn_blocking` is an escape hatch, not a general compute scheduler. Tokio's blocking pool defaults to a maximum of 512 threads. If we spawn a blocking task per incoming request during a traffic spike, we can saturate the pool and every subsequent `spawn_blocking` call just queues up. For CPU-intensive work at high concurrency, I'd recommend bounding the number of concurrent blocking tasks with a `tokio::sync::Semaphore`, or moving the work to a dedicated thread pool like `rayon`. Also worth noting that once a blocking task starts, it can't be aborted. If our application is shutting down, it'll wait for all blocking tasks to complete before exiting.

Common sources of accidental blocking include:

- **Password hashing** with Argon2 or bcrypt (CPU-intensive by design)
- **Synchronous file I/O** (use `tokio::fs` instead of `std::fs`)
- **DNS resolution** in some HTTP clients (use async-native clients like `reqwest`)
- **Calling `.lock()` on a `std::sync::Mutex`** (use `tokio::sync::Mutex` if the lock is held across await points)

## Connection pooling

Creating a new database connection for each request is expensive. We're talking TCP handshake, TLS negotiation, and authentication, which can easily add up to tens of milliseconds. Connection pooling amortizes that cost by maintaining a set of pre-established connections that our requests borrow and return.

SQLx's `PgPool` handles this transparently for us. The important thing is to tune the pool size appropriately:

```rust
PgPoolOptions::new()
    .max_connections(10)
    .min_connections(2)
    .acquire_timeout(Duration::from_secs(3))
```

If the pool is too small, requests will queue up waiting for a connection. Too large, and we waste memory on the database server while risking PostgreSQL's connection limit. A good starting point is 2 to 4 connections per CPU core on the application server, and then we adjust based on how much time our requests actually spend in the database versus doing other work.

The `acquire_timeout` is our safety net. If all connections are in use and a new request can't acquire one within the timeout, it fails immediately rather than hanging indefinitely. That's much better than having the request sit in a queue for 30 seconds before timing out at the HTTP level.

## Response compression

Compressing response bodies with gzip or brotli can significantly reduce the amount of data we send over the network. This is especially true for JSON APIs, where the response body is highly compressible text.

```rust
use tower_http::compression::CompressionLayer;

let app = Router::new()
    .merge(routes)
    .layer(CompressionLayer::new())
    .with_state(state);
```

`CompressionLayer` automatically negotiates the compression algorithm with the client based on the `Accept-Encoding` header. For most JSON APIs, gzip compression reduces response sizes by 70 to 90 percent, which translates to noticeably faster responses for clients, especially on slower networks.

## Efficient data handling

**Avoid unnecessary allocations.** In hot paths, prefer borrowing (`&str`) over owned types (`String`) where we can. Use iterators instead of collecting into intermediate vectors. These are small optimizations individually, but they compound when we're handling a lot of traffic.

**Use `Arc` for shared immutable data.** Configuration, static assets, and other data that doesn't change after startup should be wrapped in `Arc`. That way, cloning our application state is cheap (just incrementing a reference count) rather than expensive (deep-copying the data).

**Stream large responses.** For endpoints that return large amounts of data, let's stream the response instead of buffering the entire thing in memory. The `axum-streams` crate supports streaming JSON arrays, CSV files, and other formats.

## Database query optimization

In my experience, the database is almost always the bottleneck in a web application. A few practices make a real difference here:

**Add indexes for columns you filter on.** If we're querying users by email, we need to make sure the `email` column has an index. Without one, the database has to scan every row in the table for each query.

**Use `EXPLAIN ANALYZE` to understand query plans.** When a query is slow, run it with `EXPLAIN ANALYZE` in your database client to see how the database is actually executing it. Look for sequential scans on large tables, which usually point to a missing index.

**Fetch only what you need.** If a handler only needs the user's name and email, don't `SELECT *` and pull back every column. Selecting specific columns reduces the amount of data transferred and processed.

**Use pagination for list endpoints.** As we covered in the [API Design](./api-design.md) chapter, always paginate list queries. An unbounded `SELECT * FROM users` on a table with a million rows will eventually bring our application to its knees.

## Benchmarking and profiling

When we need to optimize, let's measure first. Guessing at performance bottlenecks is unreliable, because the actual bottleneck is often not where you'd expect.

**Criterion** is the standard benchmarking framework for Rust. It gives us statistically rigorous benchmarks with confidence intervals, so we can tell whether a change actually improved performance or just happened to produce faster results on one run.

**Flamegraph** visualizes where our application spends its CPU time. I've found it really useful for identifying hot spots in production-like workloads.

**DHAT** profiles memory allocations, which helps us find code that allocates more than it needs to.

**tokio-console** is a diagnostic tool built specifically for async Rust applications. It shows us which tasks are running, which are blocked, and how the runtime is scheduling work. This is particularly useful for diagnosing the "blocked runtime" issues we talked about at the beginning of this chapter.

## What not to optimize

It's worth remembering that premature optimization is a real cost. Micro-optimizing a handler that takes 2 milliseconds when the database query inside it takes 50 milliseconds? That's a waste of effort. Let's focus our optimization work on the actual bottlenecks, which in a web application are almost always database queries, network I/O, and accidental blocking of the async runtime.

Write clear, idiomatic code first. Profile under realistic load. Then optimize the specific things that the profiler tells us are slow. In my experience, this approach reliably produces applications that are both fast and maintainable. And that's what we'll need when we look at how to keep our application observable in the next chapter.
