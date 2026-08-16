# skills

Made by Dylan.

A portable copy of the `/tackle` workflow (ticket → plan → build → review → PR),
generalized out of a specific monorepo so it can be reused on another project.

Uses both Anthropic and OpenAI models: Claude (Fable/Sonnet/Haiku) plans,
implements, and orchestrates the whole run, while OpenAI's Codex (`codex exec`)
runs as an independent second opinion in the local review loop.

## Principles

1. **Human in the loop where it matters, AI for the rest.** The user's judgment
   is required at exactly two points — approving the plan and signing off on
   the QA evidence — because those are the decisions only they can make: what
   the ticket actually means, and whether the result is actually right. Every
   mechanical step in between (fetching the ticket, writing code, running
   tests, opening the PR) runs unattended, so the human's attention goes to
   the two decisions that need it, not to babysitting the rest.

2. **The system compounds — it doesn't just execute.** Every run is supposed
   to leave the codebase a little smarter, not just a little more complete.
   That happens in the "distill lessons" step at the end of the PR phase: a
   fresh, uninvolved agent looks at what went wrong during the run (a gate
   failure, a review finding, a user correction, a plan revision) and, only
   if there's a real, durable, non-obvious lesson, writes it back into
   `docs/architecture/` (how the system is designed) or a paths-scoped rules
   file (how to work in a given area) — updating an existing entry rather than
   piling on duplicates. Those files are exactly what `tackle-planner` reads
   before writing the next plan, so a mistake made once becomes a constraint
   the next ticket's plan is checked against, instead of a mistake made again.

3. **Different models for different jobs, and a second model checking the
   first.** Planning is a small amount of output that has to be right, so it
   goes to the strongest reasoning model (Fable) working alone. Implementation
   is a large amount of output that's cheap to parallelize, so it fans out
   across faster models (Sonnet), one agent per independent slice, with
   mechanical steps routed to the cheapest model that can do them (Haiku).
   Verification and review get the same split: browser QA and design review
   run in parallel against the built artifact, and the code review step
   deliberately hands the diff to a *different vendor's* model — OpenAI's
   Codex, not Claude — because a second model with a different training
   history and different blind spots catches mistakes a model is unlikely to
   catch in its own work. The result is checked by more than one kind of
   intelligence before a human ever sees it.

## How to use it

1. Copy `.claude/` and `docs/` into your project (see [What's here](#whats-here)).
2. Read [Before you use this here](#before-you-use-this-here) and swap the
   project-specific pieces (ticket source, worktree setup, QA scripts, design
   skill) for your own.
3. Fill in `docs/architecture/principles.md` with your project's real
   principles — `tackle-planner` checks every plan against it.
4. Have `gh` and Codex CLI (`codex`) installed and authenticated — Phase 5's
   review loop shells out to `codex exec`.
5. Run `/tackle <ticket URL or key>` (or a free-text title). It runs end to
   end and only stops twice: to get your plan approved, and to hand you the
   QA evidence before opening a PR.

## Diagram

```mermaid
flowchart LR
    Setup["Fetch ticket &<br/>worktree setup<br/><i>Haiku</i>"] --> Plan["Plan<br/><i>tackle-planner · Fable</i>"]
    Plan --> G1{{"PAUSE<br/>approve plan"}}
    G1 --> Build["Build<br/><i>tackle-implementer · Sonnet</i><br/>parallel, one agent per slice"]
    Build --> Verify["Verify<br/>tests · browser QA<br/>bug repro · design review"]
    Verify --> Review["Review<br/><i>codex exec</i><br/>correctness · simplification · efficiency<br/>parallel, ≤3 rounds"]
    Review --> G2{{"PAUSE<br/>review QA"}}
    G2 --> PR["PR & merge handoff"]

    style G1 fill:#f9e79f,stroke:#333
    style G2 fill:#f9e79f,stroke:#333
```

## What's here

```
.claude/
  commands/tackle.md       the /tackle slash command
  agents/
    tackle-planner.md      plans a ticket before any code is written
    tackle-implementer.md  implements one slice of an approved plan
docs/
  architecture/principles.md   stub — fill in this project's real principles
  adr/template.md               stub — copy this to write a new ADR
```

## Before you use this here

`tackle.md` still has some pieces from its original project that won't work
as-is in a new one:

- Ticket source is hardcoded to Linear
- Worktree setup calls a monorepo-specific `setup.sh` script
- QA/review steps call `scripts/qa/*` scripts that don't exist here
- Design review loads a `project-frontend-design` skill that doesn't exist here

Swap those out for whatever this project actually uses (its own ticket
tracker, its own test/lint commands, its own design rules, etc.) before
running `/tackle` for real.

## Everything else is generic

The two-pause structure (approve the plan, review the result), the
planner/implementer agent split, "don't stop between phases," and the overall
review/verify/PR loop shape all carry over as-is.

## Phases

Never stops between phases except at the two pauses.

| Phase | What happens | Model | Parallel? |
|-------|--------------|-------|-----------|
| Setup | Fetch the ticket, create the worktree | Haiku | — |
| Plan | `tackle-planner` explores the code, writes the plan | **Fable** | — |
| *pause* | user approves the plan | — | — |
| Build | `tackle-implementer` builds each slice | **Sonnet** | ✅ one agent per slice |
| Verify | type-check/lint/tests, browser QA with before/after bug repro, design self-review | orchestrator | — |
| Review | local review against the diff, fix, re-review (≤3 rounds) | `codex exec` | ✅ multiple sessions |
| *pause* | user reviews QA evidence | — | — |
| PR | push, open PR, attach QA evidence, merge handoff | orchestrator | — |

The command names *agent types* (`tackle-planner`, `tackle-implementer`),
never models — each agent's own definition pins its model, so routing stays
correct regardless of what model is running the orchestrator itself.

**What each review agent looks for:**

- **`tackle-planner` (Fable)** — checks the plan against architecture principles
  and ADRs, reuse of existing components, and correctness of the QA plan itself.
  It never touches code, only reads it.
- **`tackle-implementer` (Sonnet)** — self-reviews its own slice before
  reporting back: design-system values only (no improvised colors/spacing),
  scope creep, and whether every screen state (empty/loading/error) is handled.
- **Design self-review (Verify phase, Claude)** — looks at rendered screenshots,
  not source: visual restraint (no redundant text, information revealed only
  when needed), correct component choice, spacing/type from the design scale.
- **`codex exec` (Review phase, OpenAI)** — an independent second opinion on the
  committed diff, focused on **correctness bugs** and **reuse/simplification/
  efficiency** cleanups — deliberately not the same lens as Claude's own review,
  so it catches what Claude is more likely to miss in its own work.

