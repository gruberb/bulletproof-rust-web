# Resources

I wanted to put together the books, articles, repos, and community resources that shaped how I think about the topics in this guide. If you're looking to go deeper on anything we covered, this is where I'd start.

## Books

**[Design Patterns and Best Practices in Rust](https://www.amazon.com/Design-Patterns-Best-Practices-Rust/dp/1836209479)** by Evan Williams (2025) is a solid walkthrough of idiomatic Rust patterns. It covers GoF patterns adapted for Rust, functional patterns, and architectural patterns you'll actually use in real projects.

**[The Rust Spellbook](https://bitfieldconsulting.com/posts/best-rust-books)** (2026) is packed with hundreds of Rust tips and techniques. It's great for discovering lesser-known features of the language, the standard library, and the toolchain that you might not stumble on otherwise.

**[Rust Web Development](https://www.manning.com/books/rust-web-development)** by yours truly (Manning). This is my earlier book on building web services in Rust, covering Warp and the broader ecosystem. If you want more context on how I think about these problems, that's a good place to look.

## Architecture and design articles

**[Master Hexagonal Architecture in Rust](https://www.howtocodeit.com/guides/master-hexagonal-architecture-in-rust)** is, in my experience, the most thorough guide to implementing hexagonal architecture (ports and adapters) in Rust. It walks through domain modeling, trait-based ports, adapter implementations, dependency injection, testing strategies, and the trade-offs you'll hit along the way. I leaned on this heavily when writing the architecture chapters in this guide.

**[A Rustacean Clean Architecture Approach to Web Development](https://kigawas.me/posts/rustacean-clean-architecture-approach/)** describes a four-layer architecture (models, persistence, routers, documentation) with a pragmatic take on framework coupling. I think it's a nice complement to the hexagonal architecture guide, because it shows a simpler approach that works well when your app is mostly CRUD.

**[Rust, Axum, and Onion Architecture](https://medium.com/@jonathan.el.baz/rust-axum-and-onion-architecture-escaping-the-tech-debt-spiral-14df5db946df)** walks through implementing onion architecture with Axum. It covers the presentation, application, domain, and infrastructure layers with concrete code examples you can follow along with.

**[Building a Clean Rust Backend with Axum, Diesel, PostgreSQL and DDD](https://medium.com/@qkpiot/building-a-robust-rust-backend-with-axum-diesel-postgresql-and-ddd-from-concept-to-deployment-b25cf5c65bc8)** shows domain-driven design patterns with Axum and Diesel. You'll find the repository pattern, layered error handling, and custom extractors all in one place.

**[The Best Way to Structure Rust Web Services](https://blog.logrocket.com/best-way-structure-rust-web-services/)** covers project organization patterns, from flat modules to workspace-based clean architecture. What I like about it is the practical advice on when to pick each approach.

## Axum-specific guides

**[The Ultimate Guide to Axum (Shuttle)](https://www.shuttle.dev/blog/2023/12/06/using-axum-rust)** covers a lot of ground: handlers, routing, state management, custom extractors, middleware, testing with `oneshot`, and deployment. If you want one article that touches all the Axum basics, this is a good pick.

**[How to Build Production-Ready REST APIs in Rust with Axum](https://oneuptime.com/blog/post/2026-01-07-rust-axum-rest-api/view)** takes you through the complete lifecycle of a production API: project setup, error handling, validation, authentication, middleware composition, graceful shutdown, and Docker deployment. It's a good end-to-end reference.

**[Building High-Performance APIs with Axum and Rust](https://dasroot.net/posts/2026/04/building-high-performance-apis-axum-rust/)** focuses on performance, which we didn't dig into much in this guide. It covers benchmarking and optimization strategies specific to Axum.

**[Error Handling in Axum (LogRocket)](https://blog.logrocket.com/rust-axum-error-handling/)** goes through the various approaches to error handling in Axum, from simple status code returns to custom error types with `IntoResponse`. If our error handling chapter left you wanting more options, start here.

**[Elegant Error Handling with IntoResponse (Leapcell)](https://leapcell.io/blog/elegant-error-handling-in-axum-actix-web-with-intoresponse)** shows the `thiserror` plus `IntoResponse` pattern for centralized error handling. It's a clean approach that I've found works well in practice.

## Domain modeling and type-driven design

**[Using Types to Guarantee Domain Invariants](https://lpalmieri.com/posts/2020-12-11-zero-to-production-6-domain-modelling/)** by Luca Palmieri is one of my favorite articles on the newtype pattern and "parse, don't validate" applied to Rust web applications. If those ideas clicked for you in our earlier chapters, this goes deeper.

**[Type-Driven Design in Rust and TypeScript](https://www.luiscardoso.dev/blog/parsing-data-with-rust-typescript)** compares type-driven approaches across both languages. If you're coming from TypeScript (and many of us are), this helps bridge the mental models.

**[Rust Design Patterns](https://rust-unofficial.github.io/patterns/)** is a community-maintained catalog of Rust design patterns, anti-patterns, and idioms. It includes the newtype pattern and plenty of other patterns that come up in web development. I find myself coming back to this one regularly.

## Specific topics

**[Secure Configuration and Secrets Management in Rust](https://leapcell.io/blog/secure-configuration-and-secrets-management-in-rust-with-secrecy-and-environment-variables)** covers the `secrecy` crate, environment variable management, and in-memory secret protection. If you want to go further than what we covered in our configuration chapter, this is a good next step.

**[How to Add Structured Logging to Rust HTTP APIs (techbuddies)](https://www.techbuddies.io/2026/04/04/how-to-add-structured-logging-to-rust-http-apis-with-axum-middleware/)** walks through building a complete structured logging pipeline with correlation IDs and JSON output. Very practical, and you can follow along step by step.

**[Instrument Rust Axum with OpenTelemetry](https://oneuptime.com/blog/post/2026-02-06-instrument-rust-axum-opentelemetry/view)** covers the full OpenTelemetry setup for Axum, including trace propagation and exporter configuration. Getting OTel right can be fiddly, so having a complete reference helps.

**[JWT Authentication in Rust (Shuttle)](https://www.shuttle.dev/blog/2024/02/21/using-jwt-auth-rust)** walks you through implementing JWT authentication with custom extractors, step by step.

**[Graceful Shutdown for Axum Servers](https://medium.com/@wedevare/rust-async-graceful-shutdown-for-axum-servers-signals-draining-cleanup-done-right-3b52375412ec)** covers signal handling, connection draining, and cleanup patterns. Graceful shutdown is one of those things you don't think about until you need it, and then you really need it.

**[Health Checks and Readiness Probes in Rust for Kubernetes](https://oneuptime.com/blog/post/2026-01-07-rust-kubernetes-health-checks/view)** covers implementing liveness and readiness probes for container orchestrators. If you're deploying to Kubernetes, you'll want these.

**[Axum Backend Series: Models, Migrations, DTOs and Repository Pattern](https://blog.0xshadow.dev/posts/backend-engineering-with-axum/axum-model-setup/)** is a practical walkthrough of setting up the database layer with SQLx. It's hands-on and gets you to working code quickly.

**[An Ergonomic Pattern for SQLx Queries in Axum](https://www.joshka.net/axum-sqlx-queries-pattern/)** describes the `FromRef` pattern for clean database access in handlers. Short read, but the pattern it shows is something I use all the time.

## Example repositories

**[rust10x/rust-web-app](https://github.com/rust10x/rust-web-app)** is a production blueprint for Axum web applications. It's maintained as a reference implementation and includes things like multi-tenancy support that you won't find in most example repos.

**[launchbadge/realworld-axum-sqlx](https://github.com/launchbadge/realworld-axum-sqlx)** implements the RealWorld spec (a Medium clone API) using Axum and SQLx. I like it because it's a complete, working example of a non-trivial API, which is hard to find.

**[JoeyMckenzie/realworld-rust-axum-sqlx](https://github.com/JoeyMckenzie/realworld-rust-axum-sqlx)** is another RealWorld implementation, but with a different architectural approach. Comparing the two side by side is a great way to see how different design decisions play out in practice.

**[jeremychone-channel/rust-axum-course](https://github.com/jeremychone-channel/rust-axum-course)** has the code from a thorough Axum video course. It covers everything from basics to production patterns, and the commit history is nicely structured if you want to follow along.

**[tokio-rs/axum/examples](https://github.com/tokio-rs/axum/tree/main/examples)** has over 30 official examples maintained by the Axum team. When I'm unsure how something is supposed to work, this is usually the first place I check. It covers everything from basic routing to WebSockets, testing, and graceful shutdown.

## Ecosystem references

**[Axum ECOSYSTEM.md](https://github.com/tokio-rs/axum/blob/main/ECOSYSTEM.md)** is the official catalog of over 70 community-maintained crates for Axum. Before you build something yourself, check here first. It covers authentication, middleware, validation, observability, and all sorts of specialized features.

**[Idiomatic Rust (mre)](https://github.com/mre/idiomatic-rust)** is a peer-reviewed collection of articles, talks, and repositories that teach concise, idiomatic Rust. It's a rabbit hole, but a worthwhile one.

**[Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)** has an extensive list of recommendations for designing idiomatic Rust APIs. It's useful for both your internal module APIs and your public HTTP API, and it's where I'd point anyone who asks "how should I design this interface?"
