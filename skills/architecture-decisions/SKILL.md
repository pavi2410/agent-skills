---
name: architecture-decisions
description: >
  Record, evolve, and discover architectural decisions in a codebase using ADRs
  (Architecture Decision Records). Use this skill whenever a decision is being
  made or revisited that affects multiple files, has meaningful alternatives
  considered, or carries rationale that future maintainers (human or LLM) would
  need to find without reading source. Trigger when the user asks to "write an
  ADR", "document this decision", "record the rationale", "record a tradeoff",
  "supersede a decision", "deprecate this approach", "change a previous decision",
  or mentions "ADR", "architecture decision", "design decision", "decision record",
  "RFC accepted", or similar. Also trigger when reasoning about an existing
  codebase's decisions — when reading or applying past decisions, this skill's
  index discipline is what prevents acting on a superseded one. Do NOT trigger for
  inline code rationale (that belongs in the `intent-encoding` skill) or for
  general documentation tasks (READMEs, guides, API docs).
---

# Architecture Decision Records

A discipline for recording the structural decisions in a codebase — the ones that
affect multiple files, have meaningful alternatives considered, or carry rationale
that doesn't fit inline. Pairs with the `intent-encoding` skill, which handles
local, in-source rationale; this skill handles decisions that are too cross-cutting
or too long-form to inline.

## The first rule: read the index before reading any ADR

When entering a codebase that uses ADRs, **read `docs/decisions/README.md` (the
index) first.** Per-ADR files preserve historical reasoning and may be superseded
or deprecated; only the index reflects current state. Treating an arbitrary ADR as
authoritative is the most common failure mode of this system — especially for LLMs,
which can grab a file by filename match without realizing it's been superseded.

If a codebase has ADRs but no index, that's a gap worth flagging to the user before
proceeding. Without an index, every ADR has to be opened to determine whether it's
still in force.

## When to write an ADR

Use an ADR when at least one of these applies:

- The decision affects more than ~3 files. There's no single inline location that
  would dominate.
- The rationale is long enough (more than ~10 lines) that inlining would harm code
  readability.
- The decision has a meaningful **alternatives-considered** section that would bloat
  inline comments.
- Discoverability matters — someone joining the team should be able to find this
  decision without reading the relevant source.

If none of the above applies, the decision belongs inline. The overhead of a
separate file (separate review, easier to forget, requires index update) only pays
off when the alternatives are worse.

## Format

Use a flat directory: `docs/decisions/NNNN-short-slug.md`. Numeric prefix for
ordering, short slug for grep-ability. Don't nest by topic — flat lists scale fine
to hundreds of entries and avoid bikeshedding about taxonomy.

A minimal ADR has six sections, no more. Anything more is decoration.

```markdown
# 0007 — Ring buffer for audio thread queue

**Status:** Accepted, 2026-04-15
**Affects:** audio/engine.rs, audio/queue.rs, ui/audio_bridge.rs

## Context

The audio callback runs on a real-time thread with a ~5ms deadline. The previous
implementation used a `Vec` for the inter-thread queue, which allocated on growth
and caused intermittent xruns under load.

## Decision

Use a fixed-size lock-free SPSC ring buffer (crossbeam's `ArrayQueue`) for messages
from the UI thread to the audio thread.

## Alternatives considered

- **`Vec` with pre-reserved capacity.** Rejected: still requires lock for thread
  safety; capacity overflow falls back to allocation.
- **`crossbeam::channel::bounded`.** Rejected: internal mutex makes worst-case
  latency unpredictable.
- **Custom lock-free queue.** Rejected: not worth the maintenance cost over an
  audited library implementation.

## Consequences

- The queue has a fixed capacity (currently 1024). Overflow drops messages and
  surfaces a counter to the UI; this is preferable to blocking the audio thread.
- The choice is benchmarked in `benches/audio_queue.rs`. Revisit if the benchmark
  regresses or if message rates exceed ~10kHz.

## References

- Profiling data: docs/perf/2026-04-12-audio-xruns.md
- Crossbeam ArrayQueue docs: https://docs.rs/crossbeam-queue
```

## Linking from code

Every file the ADR names under "Affects" should have one inline reference back:

```rust
// See: docs/decisions/0007-ring-buffer-audio-thread.md
let queue = ArrayQueue::new(1024);
```

One link is enough. Don't sprinkle the same link across every line — put it at the
construction site or the type definition, wherever a reader would land first.

An ADR without inline references back to the code it explains is a dead document.
The link is the load-bearing part.

## Evolving decisions: supersede, don't edit

When a decision changes, **create a new ADR that supersedes the old one. Mark the
old one's status as superseded, with a link to the new one. Do not edit the old
one's body except to update its status field.**

The reasoning matters because the alternatives all sound reasonable:

- **Editing the old ADR with the new decision** loses the historical reasoning
  entirely. The whole point of an ADR is to record *why this was decided this way
  at that time*. Editing destroys that.
- **Appending history to the existing ADR** turns it into a stratigraphy of
  decisions, each section partially contradicting the previous. The Status field
  can only describe one state. ADRs are decisions, not changelogs.
- **Linking to git history via commit hash** technically preserves information but
  fails the discoverability test. ADRs exist precisely so readers don't have to
  run `git log --follow` to understand decisions.

Only "new ADR, supersede the old" preserves all three things you need: the
*original decision in its original context*, the *new decision in its current
context*, and a *visible link between them*.

The mechanics are minimal. Update the old ADR:

```markdown
# 0007 — Ring buffer for audio thread queue

**Status:** Superseded by 0023, 2026-08-02
**Affects (historical):** audio/engine.rs, audio/queue.rs, ui/audio_bridge.rs
```

Add a banner (see next section). Otherwise leave the body alone.

Write the new ADR with a `Supersedes:` field:

```markdown
# 0023 — Replace ring buffer with channel-based queue

**Status:** Accepted, 2026-08-02
**Supersedes:** 0007
**Affects:** audio/engine.rs, audio/queue.rs, ui/audio_bridge.rs

## Context

ADR-0007 chose a fixed-size SPSC ring buffer to avoid allocations on the audio
thread. Since then, [what changed].
...
```

Update the index (see "The decisions index" below). Update inline `// See:`
references in code to point at the new ADR.

## Marking superseded ADRs prominently

The `Status` field alone is easy to miss — especially for LLMs that may skim past
the metadata block. Add a prominent banner as the first content in the body:

```markdown
# 0007 — Ring buffer for audio thread queue

> ⚠️ **SUPERSEDED.** This decision was replaced by [ADR-0023](0023-channel-queue.md)
> on 2026-08-02. Do not apply this decision to current code. Read 0023 for the
> current approach. This document is preserved for historical reference only.

**Status:** Superseded by 0023, 2026-08-02
**Affects (historical):** audio/engine.rs, audio/queue.rs, ui/audio_bridge.rs

## Context
[unchanged from original]
...
```

The banner is intended to survive aggressive skimming — even a reader who only
ingests the first 200 tokens of the file gets the supersession warning before any
of the original Context or Decision content.

## Deprecation vs. supersession

Use `Status: Deprecated` instead of `Superseded by` when a decision becomes
obsolete because the thing it was deciding about no longer exists — not because a
new decision replaced it. Example: an ADR about a Web Audio backend choice when
the project has since switched to a Rust audio engine. There's no replacement
decision; the original is just no longer relevant.

```markdown
> ⚠️ **DEPRECATED.** This decision no longer applies. The Web Audio backend was
> removed in [link to commit/PR]; this document is preserved for historical
> reference only.

**Status:** Deprecated, 2026-09-01
```

## The decisions index

`docs/decisions/README.md` is the single entry point and the file readers
(including LLMs) consult first. It exists to answer "what's currently in force"
without requiring a read of every per-ADR file.

```markdown
# Architecture Decision Records

This index is the authoritative entry point. Read this before reading any
individual ADR. Per-ADR files preserve historical reasoning and may be
superseded — only this index reflects current state.

## Active decisions

| #    | Title                                  | Status   | Affects                |
|------|----------------------------------------|----------|------------------------|
| 0001 | Use Postgres as primary datastore      | Accepted | persistence/, api/     |
| 0023 | Channel-based queue for audio thread   | Accepted | audio/                 |
| 0024 | TypeScript strict mode                 | Accepted | (repo-wide)            |

## Superseded / deprecated

| #    | Title                                  | Status                  | Replaced by |
|------|----------------------------------------|-------------------------|-------------|
| 0007 | Ring buffer for audio thread queue     | Superseded 2026-08-02   | 0023        |
| 0014 | Custom JSON parser                     | Deprecated 2025-11-10   | —           |

## Decision lineage

- Audio thread queue: 0007 → 0023
```

Three sections, three jobs. Active answers "what's in force right now?" Superseded
preserves the trail without polluting the active list. Lineage shows multi-step
chains; for most projects this section is empty or short, but when it fills in
it's the most useful artifact in the directory.

**The index is the file most likely to drift.** A new ADR's `Supersedes:` field
gets written as part of the template, but the index is a separate file that
someone has to remember to update. Mitigations:

- Treat index updates as part of the same commit as a new or superseded ADR.
- A small CI check verifying every ADR appears in the index exactly once and that
  supersession claims are bidirectional is worth ~30 lines of script and prevents
  most rot. Mention this option to the user; don't add it without permission.
- Don't auto-generate the index from frontmatter. The lineage section needs human
  curation, and partial auto-generation invites edits to the auto-parts that get
  clobbered.

## What ADRs are not

- Not API documentation. That belongs with the API.
- Not tutorials. That belongs in `docs/guides/` or similar.
- Not a changelog. That belongs in CHANGELOG.md or release notes.
- Not a brain dump. The format is forced concision; if a decision needs more than
  one page, it usually needs to be split into smaller decisions.
- Not for proposals or speculation. ADRs record *accepted* decisions. Use a
  design doc or RFC for the proposal phase; promote to an ADR when decided.

A useful test: if the ADR wouldn't help someone six months from now decide
whether to keep, change, or undo the decision, it shouldn't exist.

## When editing existing ADRs

The cases where editing an existing ADR (rather than superseding) is acceptable:

- **Correcting a typo or formatting error** in the original text.
- **Updating the status field** to mark superseded or deprecated.
- **Adding a banner** to a superseded or deprecated ADR.
- **Adding a backward link** ("Superseded by NNNN") when a successor is written.

Anything else — changing the Decision, Context, Alternatives, or Consequences
sections — destroys historical record. If the change is substantive, write a new
ADR instead.

## When the user is starting from scratch

If a project has no `docs/decisions/` directory yet, the first ADR work creates it:

1. Create `docs/decisions/`.
2. Create `docs/decisions/README.md` with the active/superseded/lineage scaffold.
3. Write the first ADR as `0001-<slug>.md`.
4. Add the inline `// See:` reference at the relevant code site.

Don't backfill ADRs for past decisions retroactively. The exercise tends to
produce thin or speculative records (memory has faded, alternatives were never
written down) that dilute the signal of real ADRs. ADRs are written as decisions
are made, not as history is reconstructed.
