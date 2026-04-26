---
name: intent-encoding
description: >
  Encode programmer intent into source code so the *why* survives alongside the *what*.
  Use this skill whenever authoring or substantially editing code in a general-purpose
  language (Rust, TypeScript, Python, Kotlin, Go, Java, C++, etc.) and the code carries
  non-obvious specification, design, constraint, or causal intent that future readers
  (human or LLM) would need to understand or preserve. Trigger when the user asks to
  "write", "implement", "refactor", "document the why", "add comments that won't rot",
  "make this code self-explaining", or mentions "intent", "design rationale", "invariants",
  "assumptions", "why is this code like this", or similar. Also trigger
  proactively during authoring tasks when a function involves non-trivial invariants,
  surprising tradeoffs, ordering constraints, performance-driven shape, or assumptions
  about callers/inputs/the world that aren't visible from names and types alone. Do NOT
  trigger for trivial, self-descriptive code — this skill is explicitly biased against
  annotation theater. If names, types, and structure already convey the intent, the skill
  says: don't annotate.
---

# Intent Encoding for General-Purpose Code

Source code reliably encodes *what* the machine does. It encodes *why* the programmer
wrote it that way only weakly, and that weakness is where bugs, regressions, and bad
refactors come from. This skill is a discipline for closing that gap when — and only
when — closing it pays off.

## Core stance

**Self-descriptive code first. Annotation last.**

Before adding any intent annotation, try to make the code itself convey the intent.
Rename a variable. Extract a function. Replace a magic number with a named constant.
Tighten a type. Split a branch. Almost every annotation you're tempted to write is a
signal that the code could be clearer.

Reach for explicit intent encoding only when the intent is **genuinely not derivable
from the code** — invariants the compiler can't see, constraints from the world outside
the program, decisions where the alternative was non-obvious, or rationale that explains
*absence* (why something is *not* there).

## The four kinds of intent

Different intents live in different places and rot in different ways. Treating them
as one thing ("comments") is why comments don't work. Separate them.

### 1. Specification intent — *what should this do?*

The contract: inputs, outputs, postconditions, error behavior. **Prefer code-level
encoding.** Use the strongest mechanism the language offers, in this order:

1. Types (sum types, refinement types, newtype wrappers, branded types)
2. Assertions, `debug_assert!`, `invariant()`
3. Property-based or example-based tests adjacent to the code
4. Doc comments — only for what the above genuinely can't capture

If you reach step 4, the doc comment should describe behavior that is *deliberately
unverified* (e.g. "best-effort retry; may drop duplicates under partition") and explain
why it's unverified.

### 2. Design intent — *why is it shaped this way?*

The structural decision: why this abstraction boundary, this data layout, this
concurrency model, this dependency direction. Encode it **closest to the boundary it's
defending** — the module top, the trait declaration, the construction site. For
decisions that span multiple files or carry meaningful alternatives-considered
rationale, use an architecture decision record — see the `architecture-decisions` skill.

Strong design-intent annotations explain the *alternatives that were rejected* and why.
"We use a ring buffer here" is weak. "We use a ring buffer because the allocator showed
up in profiling and a Vec resize stalled the audio thread" is strong — it tells the
next person what would have to change for a different choice to win.

### 3. Constraint intent — *what must hold?*

Invariants, preconditions, assumptions about callers, environment, or data. The
highest-leverage layer, because constraint violations cause the most expensive bugs
and constraints are exactly what tend to be unwritten.

**Prefer mechanically-checkable constraints:** type, assertion, debug check, test.
Fall through to a comment only when the constraint is genuinely external (a hardware
quirk, an upstream API contract, a UI thread requirement).

When a comment is the only option, mark it visibly:

```
INVARIANT: <what must hold>
ASSUMES:   <what the caller / world must guarantee>
```

These tags are deliberately ugly — they're meant to catch the eye during refactors,
which is exactly when invariants get broken silently.

### 4. Causal intent — *why does this line exist?*

The historical reason: this exists because of bug X, regulation Y, customer Z's data.
The most fragile kind, because it's easiest to lose in a refactor. Git blame is the
degenerate form and rots the moment anyone touches the line.

Encode causal intent only when it's **load-bearing for future change.** "Fixes #1234"
on a workaround for a third-party bug is load-bearing — if the third party fixes their
bug, the line should be removed. "Refactored on 2024-03" is not load-bearing; delete it.

```
WHY: <the cause, with link if applicable>
```

If the cause is gone (bug fixed upstream, regulation repealed), **delete the workaround
and the comment together.** Stale causal intent is worse than none.

## Decision flow when authoring

For each non-trivial block of code, ask in order:

1. **Can the code itself carry this?** Better name, better type, better structure.
   If yes, do that and stop.
2. **Is there a constraint the compiler/runtime can check?** Assertion, type, test.
   If yes, encode it and stop.
3. **Is there intent that's genuinely not in the code and that a future maintainer
   would lose without help?** If no, stop. If yes, pick the right kind (spec / design
   / constraint / causal), use the tag if it's a comment, and keep it short.
4. **Does this intent span multiple files or carry alternatives-considered rationale?**
   If yes, it likely belongs in an ADR — see the `architecture-decisions` skill.

The "stop" answers are the most important. The default is to write nothing.

## What this skill is against

Read this section every time you're about to add an annotation.

- **Restating the code.** `// increment counter` above `counter += 1`, or `// the
  user's id` above `user_id: UserId`. The line is the comment.
- **Apology comments.** `// TODO: this is hacky` without saying what better looks like
  or what's blocking it. Either fix it or write a real `WHY:` tag.
- **Decorative banners and author stamps.** `// ===== HELPERS =====`, `// Added by
  Alice, 2024-03-12`. Version control owns the second; the first means split the file.
- **Speculative future intent.** `// might want to make this async later`. Either do
  it or don't write it down.
- **Annotations on trivial code.** A short pure function with descriptive name and
  types needs no further annotation. Adding one dilutes the signal of annotations
  that *do* matter.

When in doubt, leave it out. A codebase with a few high-signal intent annotations is
more valuable than one with annotations everywhere.

## Heuristics for "is this worth annotating?"

If the answer to all of these is no, don't annotate.

- **Surprise test:** would a competent reader looking at this in six months be
  surprised by the choice, the constraint, or the reason this exists?
- **Refactor test:** if someone refactored this without reading the comment, would
  they break something?
- **Question test:** has someone actually asked "why is this like this?" about this
  code or near-identical code? If yes, annotate the answer.
- **Link test:** is there an external thing (bug, ticket, paper, RFC, benchmark) that
  the reader needs to know exists? The comment exists to carry the link.

## Examples

The principle is language-independent. The examples below show the *kind* of judgment,
not language-specific syntax.

### Example 1 — Don't annotate; rename instead

**Tempted to write (Python):**

```python
# Returns the user if found, else None. Called from the auth middleware.
def get(uid):
    ...
```

**Better — the code carries the intent:**

```python
def find_user_by_id(user_id: UserId) -> User | None:
    ...
```

The name and signature now tell the reader everything the comment said. The "called from
auth middleware" part is either irrelevant (it's just one of many callers) or it's a
real constraint — in which case encode it as a constraint, not as trivia.

### Example 2 — Constraint that genuinely can't be a type (TypeScript)

```typescript
// INVARIANT: callers hold the document lock for the duration of this call.
// Violating this races with the autosave worker (see RaceTest.documentMutation).
function applyEdit(doc: Document, edit: Edit): void {
  ...
}
```

The lock isn't visible in the type system here. The annotation is load-bearing because
the next person to call this function from a new site needs to know. The pointer to
the race test makes the constraint *checkable* indirectly — if you doubt the comment,
you can read the test.

A stronger version would be a `LockedDocument` newtype that the function takes instead
of `Document`, making the constraint a type. Reach for that first. The comment is the
fallback when the refactor is out of scope.

### Example 3 — Design intent worth recording (Rust)

```rust
// We deliberately do NOT use a HashMap here. Profiling showed the hash function
// dominated for the typical case (n < 8). A linear scan over a Vec wins until ~32
// entries; revisit if entry counts grow. Bench: benches/lookup.rs.
struct SmallTable { entries: Vec<(Key, Value)> }
```

This is design intent. It explains a choice that looks naive without context, names
the alternative that was rejected, gives the threshold for revisiting, and points to
the benchmark that justifies it. A future reader who wants to "improve" this to a
HashMap now has to engage with the actual reason — and if `n` has grown past 32, they
have a green light to make the change.

### Example 4 — Causal intent, narrowly scoped

```rust
// WHY: Some Android OEMs return a bogus content-length of -1 for chunked responses.
// Fall back to read-to-end in that case. Tracked upstream in OEM-bug-7421.
if content_length < 0 {
    return read_to_end(stream);
}
```

If `OEM-bug-7421` ever resolves and the affected devices age out, this branch and its
comment should be deleted together.

### Example 5 — Triviality test

```kotlin
fun isEven(n: Int): Boolean = n % 2 == 0
```

No annotation needed. Adding one would make the code worse, not better.

## When editing existing code

Treat annotations as code: keep them in sync with the lines they describe, or delete
them — stale is worse than absent. If you remove a workaround, search for its `WHY:`
tag and delete it in the same change. If an existing annotation is wrong, fix it as a
clearly-labeled commit so history shows the correction.
