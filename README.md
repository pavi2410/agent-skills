# Agent Skills

A collection of skills for AI coding agents, focused on **codebase hygiene and
maintainability disciplines** — the cross-cutting practices that keep a codebase
understandable, maintainable, and resilient to refactor as it grows.

These skills are deliberately not technology-specific. They apply across general-purpose
languages (Rust, TypeScript, Python, Kotlin, Go, Java, C++, etc.) and across stack
choices.

## Available skills

### intent-encoding

Encode programmer intent into source code so the *why* survives alongside the *what*.
Distinguishes specification, design, constraint, and causal intent — each encoded with
the strongest mechanism the language offers (types > assertions > tests > comments).
Strongly biased against annotation theater.

**Use when:**

- Writing or substantially editing code with non-obvious invariants, tradeoffs, or
  assumptions
- Adding comments that "won't rot"
- Refactoring legacy code that has accumulated unclear rationale
- Asking "why is this code like this?" and wanting the answer to be discoverable later

**Skips:**

- Trivial, self-descriptive code (annotation would dilute signal)
- Decisions that span multiple files or carry alternatives-considered rationale —
  those belong in `architecture-decisions`

### architecture-decisions

Record, evolve, and discover architectural decisions using ADRs (Architecture Decision
Records). Covers ADR format, supersession protocol, prominent superseded-banner pattern,
and the index/lineage discipline that keeps the system from rotting as decisions evolve.

**Use when:**

- Documenting a structural decision (database, framework, threading model, API shape)
- Changing or reversing a previous decision
- Reading an existing codebase's decisions and needing to know what's currently in force
- Setting up `docs/decisions/` for the first time

**Skips:**

- Inline code rationale (use `intent-encoding` instead)
- General documentation tasks (READMEs, guides, API docs)
- Speculative proposals not yet decided (use a design doc or RFC)

### ask-for-help

Recognize when the agent is stuck — looping, blocked by missing access, or working
on genuinely ambiguous intent — and ask the human a structured, useful question
instead of grinding. Distinguishes real stuck-ness (multiple concrete signals) from
first-contact friction. Strongly biased against premature escalation as well as
against silent looping.

**Use when:**

- The same action has failed two or more times with the same or near-identical error
- An interactive terminal, TUI, REPL, or browser flow is unresponsive to the inputs
  you can send
- The next step requires authentication, credentials, or external resources you
  don't have
- The user's intent is genuinely ambiguous in a way that changes the approach
  materially
- You notice yourself about to retry-with-a-tweak for the third time

**Skips:**

- First failures (diagnose and retry once before escalating)
- Stylistic decisions the user has implicitly delegated
- Long-running tasks that are still making progress
- Confirmation of routine actions already authorized in session or `CLAUDE.md`

## Relationship between the skills

The two skills are complementary, not overlapping:

```
                    Intent / rationale to record
                              │
                ┌─────────────┴──────────────┐
                ▼                            ▼
     Local to one site/file?            Spans multiple files
     Short rationale?                   Has alternatives considered?
     Constraint enforceable             Needs to be discoverable
     in code?                           without reading source?
                │                            │
                ▼                            ▼
        intent-encoding             architecture-decisions
        (in-source annotation)      (ADR + index + supersession)
```

Each skill points to the other when its scope ends. Code-resident annotations link to
ADRs; ADRs reference the code they affect.

`ask-for-help` is orthogonal to the two skills above. It governs *when the agent
stops and consults the human* — independent of which artifact-type skill (if any)
is in use during the work.

## Skill structure

Each skill follows the standard Anthropic SKILL.md format:

```
skills/<skill-name>/
├── SKILL.md       # YAML frontmatter + instructions
├── references/    # Optional: deeper docs loaded on demand
└── scripts/       # Optional: helper scripts
```

## Contributing

See `AGENTS.md` for guidance on adding or modifying skills in this collection.

## License

MIT — see [LICENSE](LICENSE).
