# Bulletproof Rust Web

An opinionated guide to building production-grade web applications with Rust and Axum.

**[Read it here](https://gruberb.github.io/bulletproof-rust-web/)**

## What is this?

The React ecosystem has [bulletproof-react](https://github.com/alan2207/bulletproof-react), a well-known reference for structuring React applications in a way that scales with team size and codebase complexity. This project attempts the same thing for Rust web backends.

Axum is deliberately minimal. It gives you a router, extractors, and Tower middleware integration, then gets out of your way. That restraint is a strength, but it means two Axum projects built by two different teams can look completely different. One puts SQL in handlers. Another builds five crates of hexagonal architecture. Neither approach is inherently wrong, but the inconsistency makes it harder to onboard people, harder to maintain over time, and harder to know where things should go.

This guide provides a curated set of answers to those structural questions.

## What it covers

The book is organized into six sections:

**Architecture** covers project structure (single crate and workspace), the layered dependency model, trait-based abstractions, and domain modeling with Rust's type system.

**Core Patterns** addresses error handling (thiserror + anyhow + IntoResponse), the database/repository layer with SQLx, application state management, and configuration with secrets protection.

**HTTP Layer** focuses on Axum specifics: routing, thin handlers, Tower middleware composition, request validation, JWT authentication, and API design (versioning, pagination, OpenAPI).

**Production** covers structured logging with tracing, security hardening (CORS, CSRF, rate limiting, security headers), testing strategies (unit and integration with oneshot), performance, and Docker deployment with graceful shutdown.

**Advanced** covers async pitfalls and cancellation safety, channel-based message passing, multi-subsystem graceful shutdown, background jobs, gRPC with Tonic, outbound HTTP resilience (timeouts, retries, circuit breaking), and typestate patterns.

**Reference** provides a crate lookup table, a catalog of common anti-patterns, a curated reading list, and an AI-assisted development workflow with ready-to-use repo instruction files.

## AI Agent Templates

The `templates/` directory contains ready-to-use instruction files for AI coding agents (Claude Code, Codex, Cursor). Copy them into your Rust project to give your AI agent the guide's rules as persistent, structured context:

```
templates/
  AGENTS.md                              # Root project rules
  src/http/AGENTS.md                     # HTTP layer rules
  src/domain/AGENTS.md                   # Domain layer rules
  src/infra/db/AGENTS.md                 # Database layer rules
  .agents/skills/add-endpoint/SKILL.md   # Step-by-step endpoint creation
  .agents/skills/add-migration/SKILL.md  # Migration workflow
  .agents/skills/review-pr/SKILL.md      # PR review checklist
  docs/checklists/feature.md             # Feature implementation checklist
```

## Building locally

```sh
cargo install mdbook
mdbook serve book
```

Then open http://localhost:3000.

## Contributing

Found something wrong or missing? Open an issue or submit a PR. The source files are in `book/src/`.
