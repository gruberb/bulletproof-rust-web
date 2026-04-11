# Introduction

If you've done any React work, you might have come across [bulletproof-react](https://github.com/alan2207/bulletproof-react). It's an opinionated guide that gives teams a shared understanding of how to structure applications, which patterns to reach for, and where different kinds of logic belongs. It doesn't invent new ideas so much as curate the ones that have proven themselves in production, then organize them into something coherent.

I wanted the same thing for **Rust and Axum**. So here we are.

Axum is deliberately minimal. You get a router, extractors, and deep integration with Tower, and then it gets out of the way. That restraint is a genuine strength, but it also means two Axum projects built by two different teams can look completely different. One team puts SQL directly in their handlers. Another builds an elaborate hexagonal architecture with five workspace crates. Neither is wrong, exactly, but the inconsistency makes it harder to onboard new people, harder to maintain things over time, and harder to figure out where a particular piece of logic should live. I've seen this play out enough times to know it's a real problem.

What I've tried to do here is provide a curated set of answers to those structural questions. The thinking is informed by hexagonal and clean architecture literature, and by the practical experience I've picked up from dozens of blog posts, conference talks, and open-source projects across the Rust web community.

## What this guide covers

The book is organized into six sections. Let me walk you through them quickly so you know what's where.

**Architecture** lays the foundation. We cover how to organize your files and modules, how to think about layers and dependency direction, and how to model your domain using Rust's type system so that invalid states become unrepresentable. This is the stuff I wish someone had shown me when I started my first serious Axum project.

**Core Patterns** tackles the problems every web application has to solve: error handling, database access, application state, and configuration management. You'll find concrete patterns with real code that you can adapt for your own projects.

**HTTP Layer** is where we get into the Axum-specific parts of the stack: routing, handlers, middleware composition, request validation, authentication, and API design conventions like pagination and versioning.

**Production** covers what you need to actually ship and operate the thing: structured logging and tracing, security hardening, testing strategies, performance considerations, and deployment with Docker and graceful shutdown. Because writing code is only half the job, right?

**Advanced** goes beyond the single-server HTTP model into patterns that show up in production systems with real complexity. We dig into the sharp edges of async Rust (cancellation safety, `select!` semantics), inter-component communication with channels, coordinated shutdown of multi-subsystem applications, background job management, adding a gRPC surface with Tonic, and compile-time state machines using the typestate pattern. Some of this stuff took me a while to figure out on my own, so I'm hoping it saves you some pain.

**Reference** gives you a quick-lookup crate table, a catalog of common anti-patterns to watch out for, and a curated list of books, articles, and example repositories if you want to go deeper.

## Guiding principles

A few values run through everything I recommend in this book. They're not original ideas (none of this is), but they're the ones I keep coming back to.

**Keep handlers thin.** A handler's job is to pull data out of the request, call into a service, and turn the result into a response. That's it. If your handler is doing validation, database queries, and business logic all in one function, it's doing too much. I've seen enough 200-line handlers to know where that road ends. Push that work into the appropriate layer.

**Let the type system do the work.** Rust's type system is unusually powerful for a systems language, and in my experience it really shines in web applications. Newtypes that enforce invariants at construction time, `Result` types that make error paths explicit, trait-based abstractions that decouple your layers from each other. These catch bugs at compile time that would otherwise show up in production at 2am. Worth the upfront effort every single time.

**Separate what changes for different reasons.** Your HTTP routing might change because you're adding a new endpoint. Your database queries might change because you're optimizing a slow path. Your business rules might change because the product requirements evolved. When these concerns live in different modules with clear boundaries, you can change one without worrying about the others. It sounds obvious when you say it out loud, but it's surprisingly easy to let things bleed together.

**Start simple, evolve deliberately.** Not every project needs a hexagonal architecture with five workspace crates. A small API with a handful of endpoints can get by with a flat module structure and direct SQLx calls. What matters is understanding the principles behind the more elaborate architectures so you can move toward them incrementally when the complexity of your project actually demands it, rather than over-engineering from the start or scrambling to refactor when things get messy.

**Optimize for the reader.** Code is read far more often than it's written. We all know this, but it's easy to forget in the moment. The patterns in this guide prioritize clarity and predictability over cleverness. When someone new joins your team and opens the codebase for the first time, they should be able to find what they're looking for and understand why it's structured the way it is.

## How to use this guide

You don't need to read this cover to cover. If you're starting a new project, the Architecture section will help you set up a solid foundation. If you're working on an existing project and want to improve a specific area, just jump straight to the relevant chapter. Each chapter is self-contained enough to be useful on its own, though they reference each other where topics overlap.

I've tried to make the code examples realistic. They use the actual crate APIs you'd use in production, with proper error handling and real type signatures. No `todo!()` placeholders where the hard parts go. Where there are meaningful trade-offs between approaches, I explain both options and give you enough context to make the right call for your situation.

One more thing: this is a living document. The Rust web ecosystem is still evolving fast, and what we consider best practice today will keep getting refined. The principles tend to be more stable than the specific crates or APIs, though, and that's where I think most of the lasting value here lies. Let's get into it.
