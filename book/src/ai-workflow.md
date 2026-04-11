# AI-Assisted Development Workflow

If you've been building with AI coding agents, you've probably noticed that they're great at cranking out code but terrible at knowing *your* project's rules. This chapter is about bridging that gap. We'll look at how to take the principles from this guide and turn them into repo-level instructions that AI agents (Claude Code, Codex, Cursor, Aider) can consume effectively, so they stop guessing and start following your architecture.

## The problem with pasting the whole guide

I get it, the temptation is real. You copy the whole guide, paste it into a chat, and ask the agent to "build a Rust app." I've done it. The results are... not great. The model can't tell what's a hard rule versus a nice-to-have suggestion, so it cherry-picks whatever seems easiest and ignores the rest.

What I've found works much better is distilling your non-negotiable rules into persistent repo files that the agent loads automatically at the start of every session. Then you provide procedural knowledge (how to add an endpoint, how to write a migration) as discrete skills that the agent invokes when it needs them. Let's walk through how to set this up.

## Repo instruction structure

Most AI coding tools these days support persistent, file-based instructions that get loaded automatically when the agent starts a session:

- **Claude Code** reads `CLAUDE.md` at the repo root, plus `.claude/rules/` for scoped rules
- **Codex** reads `AGENTS.md` at the repo root, plus nested `AGENTS.md` files in subdirectories
- **Cursor** reads `.cursor/rules/` or `AGENTS.md`
- **Aider** uses a repo map to understand structure and can be given conventions files

The file names differ, but the idea is the same: short, specific, durable instructions that live in the repo and get loaded at the start of every session. That's a much better home for your rules than a chat message you'll forget to paste next time.

This repo includes a `templates/` directory with ready-to-use instruction files. You can copy them into your project and adapt them to your codebase.

## Root instructions

Your root `AGENTS.md` (or `CLAUDE.md`) should contain only the rules that apply to every session and every file. I'd keep it under 40 lines. Think of it as the stuff you'd tell a new team member on day one:

- The project's layer structure and dependency rule
- Hard code style rules (no unwrap in handlers, newtypes for domain values, etc.)
- The verification command that defines "done"
- Pointers to nested instruction files for layer-specific rules

```
templates/
  AGENTS.md                          # Root rules (copy to your repo root)
```

Keep this file intentionally small. Everything that only applies to a specific layer goes into nested files, which we'll look at next.

## Nested layer instructions

Each architectural layer gets its own instruction file with rules specific to that layer. This way, the agent only loads the relevant guidance when it's editing files in that directory. No need to flood it with database rules when it's working on a handler.

```
templates/
  src/http/AGENTS.md                 # Handler, DTO, middleware, routing rules
  src/domain/AGENTS.md               # Model, service, port, error rules
  src/infra/db/AGENTS.md             # Repository, query, migration rules
```

The HTTP layer instructions cover handler length limits, extractor conventions, middleware ordering, and DTO rules. The domain layer instructions enforce our zero-framework-dependency rule and the newtype/constructor patterns we covered earlier. The database layer instructions cover query patterns, row-to-domain mapping, and migration conventions.

## Skills for procedural workflows

Rules tell the agent what to do. Skills tell it *how*. You might wonder why the distinction matters. In my experience, agents are surprisingly good at following step-by-step procedures but terrible at inventing them from scratch. A skill is basically a recipe for a common task that the agent can follow when it needs to.

```
templates/
  .agents/skills/add-endpoint/SKILL.md    # Domain -> DB -> Handler -> Test flow
  .agents/skills/add-migration/SKILL.md   # Migration creation and verification
  .agents/skills/review-pr/SKILL.md       # Architecture and security review checklist
```

The add-endpoint skill walks through our full vertical slice: define domain types, create the migration, implement the repository method, add the service logic, write the DTO and handler, register the route, write tests, add tracing, and run verification. This is the procedure we want the agent to follow every time it adds a new API endpoint, and it mirrors exactly how we'd do it ourselves.

The review-pr skill is a checklist that covers architecture (dependency direction, handler thickness), correctness (no unwrap, proper error mapping), database (parameterized queries, TryFrom conversions), security (input validation, no secrets in logs), testing (happy path and error path), and operations (tracing, no blocking, outbound timeouts).

## Checklists for human review

Even when an AI agent writes the code, you still need a human reviewing it. Don't skip this part. The feature checklist gives you a structured path through that review:

```
templates/
  docs/checklists/feature.md              # End-to-end feature implementation checklist
```

## The three-phase chat loop

Here's the workflow I've settled on after a lot of trial and error. It's a three-phase loop, and it works because it forces the agent to think in the same order we would: scope first, then implementation, then review.

**Phase 1: Plan.** Ask the agent to read the instruction files, summarize the architecture constraints that apply, propose the smallest vertical slice to implement, and list the files it plans to edit. Don't let it write code yet. This might seem tedious, but it catches bad assumptions before they turn into bad code.

**Phase 2: Implement.** The agent implements one slice at a time. It stops after the code compiles and the tests pass, then shows you what changed and what's left.

**Phase 3: Review.** The agent reviews its own diff against the review checklist. It flags architecture drift, leaked types, missing tests, blocking calls, and security issues. It suggests minimal fixes.

You'll be surprised how much better the output gets compared to a single open-ended prompt. The constraint of working in phases keeps the agent from going off the rails and building something that compiles but violates half your architecture rules.

## Promote corrections into repo memory

When the agent makes the same mistake twice, and it will (leaking a database type into the API, putting SQL in a handler, forgetting tracing instrumentation), don't just correct it in chat. Go update the root or nested instruction file. The next session starts smarter because the correction is now part of the project's persistent context.

This is where the real compounding value comes from. Over time, your instruction files evolve into a precise, battle-tested encoding of your team's standards. What I've found is that after a few weeks, both humans and agents end up following the same rules consistently, which is kind of the whole point.

## Getting started

1. Copy the `templates/` directory contents into your Rust project
2. Rename `AGENTS.md` to `CLAUDE.md` if you use Claude Code (or keep both, they won't conflict)
3. Adapt the rules to your specific project, adjusting layer paths and adding any project-specific conventions
4. Move the nested instruction files to match your actual `src/` directory structure
5. Start a chat session and ask the agent to read its instruction files before doing any work

That last step is important. If you don't ask the agent to read its instructions first, it'll happily ignore them and do whatever it wants. Once you've got this workflow running, you'll find that the agents become genuinely useful collaborators rather than code generators you have to babysit.
