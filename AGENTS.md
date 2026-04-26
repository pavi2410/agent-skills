# Working on this repository

This file is the contributor guide for AI agents (and humans) working on the skills
themselves — not for the skills' end users. End-user documentation lives in the
top-level `README.md` and in each skill's `SKILL.md`.

## Repository philosophy

These skills are codebase-hygiene disciplines, not technology-specific cookbooks.
A skill earns its place here when:

- It addresses a problem that recurs across languages and stacks.
- The discipline is non-obvious enough that a competent agent without the skill
  loaded would produce noticeably worse output.
- The discipline can be tested — there should be a realistic scenario where you
  can compare with-skill and without-skill outputs and see a meaningful difference.

Skills that fail any of those tests don't belong here. Narrow, technology-specific
guidance is better as project documentation or a dedicated skill in a separate
repository.

## Adding a new skill

Before drafting:

1. **Justify the skill on its own merits.** Don't add a skill because the collection
   "feels incomplete" — that's the same anti-pattern the `intent-encoding` skill
   warns against in code.
2. **Confirm scope doesn't overlap an existing skill.** If a new skill would partially
   duplicate `intent-encoding` or `architecture-decisions`, consider extending one of
   those instead. A monorepo with two well-tested skills is more valuable than one
   with five overlapping ones.
3. **Sketch the failure mode.** Write down what a session *without* the skill would
   produce on a realistic task, and what the skill would change. If you can't name
   a clear difference, the skill isn't ready.

When drafting:

- Follow the existing skill format. YAML frontmatter with `name` and `description`,
  Markdown body, optional `references/` for content loaded on demand.
- The description is the trigger surface. Make it specific about when to trigger
  and explicit about when *not* to trigger.
- Bias against prescription. A skill that says "always do X" is brittle; one that
  says "here's how to think about X, here's when X applies, here's the failure
  mode if you skip X" survives.
- Keep `SKILL.md` under ~500 lines. Push deeper material into `references/`.
- Crosslink to other skills in this collection when scopes meet.

After drafting:

- Test it on a realistic scenario. Write the with-skill and without-skill outputs
  for at least one task; verify the difference is real.
- Update the top-level `README.md` with the new skill's section.

## Modifying an existing skill

Apply the skills' own disciplines to the skills themselves:

- **From `intent-encoding`:** if you're tempted to add a section explaining
  *why* the skill is structured a certain way, first see if the structure can be
  made self-evident. Reorganize before annotating.
- **From `architecture-decisions`:** if you're substantially changing a skill's
  approach (not a typo or clarification), consider whether the change deserves
  a record. Skills don't have ADRs of their own, but a commit message should
  carry the rationale at minimum.

When in doubt, the skill body is the user-facing spec. Edit it as carefully as
you'd edit a user-facing API.

## What does NOT belong here

- **Project-specific conventions.** "Our team uses tabs not spaces" or "all our
  React components use this naming pattern" belongs in the project's `CLAUDE.md`,
  not in a skill.
- **Tooling instructions.** "How to run our test suite" is project documentation.
- **Tutorials.** Skills are reference material loaded into context during work,
  not learning material.
- **Speculative skills.** If a skill exists only because "we might want this
  someday," delete it. A skill that doesn't trigger doesn't help anyone.

## Testing a skill

The minimum viable test is a side-by-side comparison: pick a realistic task, run
it once without the skill loaded and once with the skill loaded, and check that
the outputs differ in the way the skill predicts.

For skills with verifiable outputs, follow the broader test methodology in
Anthropic's `skill-creator` skill (`evals/`, grading, benchmarking). For
discipline-style skills like the ones in this collection, qualitative comparison
is usually enough — but it should be written down, not just felt.
