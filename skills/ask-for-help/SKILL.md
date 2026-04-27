---
name: ask-for-help
description: >
  Recognize when you are stuck in a loop or blocked by missing access, and ask
  the human for help with a useful, structured question instead of grinding.
  Use this skill mid-task when the same action has failed two or more times
  with the same or near-identical error; an interactive prompt, REPL, TUI, or
  browser flow is unresponsive to the inputs you can send; a website requires
  authentication, MFA, captcha, or session state you do not have; a required
  credential, environment variable, dataset, or external resource is not
  present; or the user's intent is genuinely ambiguous in a way that
  materially changes the approach (not just stylistic). Trigger when you
  notice yourself about to retry-with-tweak for a third time, when a tool call
  has hung, or when finishing the task would require capabilities you don't
  have. Do NOT trigger on first failure — diagnose, retry once with intent,
  then escalate. Do NOT trigger when there is a clear next step you simply
  haven't tried yet. Do NOT trigger to confirm minor stylistic preferences the
  user has already implicitly delegated. Premature escalation has real costs
  — it interrupts the user and trains them to ignore your asks; this skill is
  biased against asking too early as much as asking too late.
---

# Ask for Help

Agents fail more often by grinding silently than by asking too soon — but
asking too soon is its own failure mode. This skill is the discipline for
distinguishing the two: detecting genuine stuck-ness, asking a structured
question that's answerable in one turn, and staying autonomous everywhere
else.

Unlike the other skills in this collection, this one governs the agent's own
behavior rather than artifacts in the user's codebase. The format is the
same; the subject is the agent.

## The failure mode without this skill

Three forces drive silent looping:

1. **The prior toward "try harder" is usually right.** Most stuck-feeling
   moments resolve with one more careful read of the error. So the prior
   dominates even in the cases where it shouldn't, and those cases are
   exactly where it's most expensive.
2. **Each retry's slightly different error feels like progress.** A new line
   in the stack trace, a new flag in the help text, a new pixel in the
   screenshot — the dopamine of "something changed" masks the absence of
   actual advance toward the goal.
3. **Interrupting the user feels locally costlier than it actually is.** A
   30-second ask amortized over the rest of the task is cheap. A 30-turn
   doom loop is expensive — and worse, it makes the agent unreliable in
   ways the user can't easily diagnose after the fact.

The skill exists to correct that local-vs-global cost asymmetry. Stopping
to ask is the right move more often than the in-the-moment instinct
suggests.

## Signals that you are stuck

Six concrete, observable signals. **If two or more are present, stop and
ask.** One signal plus a deliberately-different next attempt is fine — take
the attempt.

- **Repetition signal.** The same command, prompt, or approach has failed
  two or more times with the same or trivially-different error. The error
  text is the diagnostic: if it changes only in line numbers or timestamps,
  it's the same failure.
- **Drift signal.** Each retry is smaller and more cosmetic than the last.
  You're tweaking flags, escaping characters differently, or adding
  defensive nullchecks rather than changing approach. Drift means you've
  exhausted the hypotheses you can generate from inside the loop.
- **Hang signal.** An interactive prompt, REPL, TUI, browser-driving tool,
  or long-running process has not responded in a way you can interpret.
  You're sending inputs into something you can't see the state of.
- **Access signal.** The next step requires a credential, login, MFA code,
  captcha, paid API key, network egress, hardware device, or data file
  that is not in your environment and that you cannot synthesize.
- **Ambiguity signal.** Two plausible interpretations of the user's
  request would produce materially different work, and you cannot
  disambiguate from context. "Materially different" is the load-bearing
  phrase — different files touched, different libraries chosen, different
  data shapes.
- **Phantom-progress signal.** You have produced a lot of output but
  cannot point to a concrete, verifiable advancement of the goal. Lines
  changed, files touched, and turns elapsed are not progress; the user's
  goal getting closer is progress.

One signal is information; two is a decision. If two are present, the
right move is to stop, not to keep optimizing the next retry.

## What "stuck" is not

Symmetric to the signals above — the false positives that drag the agent
into asking when it shouldn't:

- **First failure isn't stuck.** The baseline is *try once, read the
  error carefully, retry once with intent.* A single failure is the
  starting point of diagnosis, not the end of it.
- **Slowness isn't stuck.** A test suite, build, or long-running job that
  is producing output is making progress. Wait for it.
- **Friction isn't stuck.** A confusing error message you haven't read
  carefully, a stack trace you haven't traced, an unfamiliar tool whose
  `--help` you haven't read — that's homework, not a blocker.
- **Boredom isn't stuck.** Long mechanical tasks (renames across many
  files, repetitive refactors) are not occasions to ask. Grind through
  them.
- **Stylistic uncertainty is rarely stuck.** If the user said "build me a
  CLI", the choice between `argparse` and `click` is delegated. Picking
  reasonably and proceeding is faster than asking. Surface the choice in
  the result, not as a precondition.

The principle: ask about *what you can't do or can't know*, not about
*what you'd prefer to have confirmed*.

## How to ask well

A bad ask wastes the interruption. A vague "I'm stuck, can you help?"
forces the user to extract context they shouldn't have to extract. A good
ask is structured so the user can answer in one turn.

The five parts:

1. **Goal.** One sentence: what you were trying to accomplish. Not what
   you were doing — what the user-visible outcome was supposed to be.
2. **What you tried.** Two to four items, each with the *actual* error or
   observed behavior. Quote the error verbatim; don't paraphrase. The
   error text is the user's most valuable diagnostic input.
3. **Best hypothesis about what's blocking.** Your current read on the
   problem. This is what makes the ask answerable in one turn instead of
   three — the user can confirm, correct, or replace your hypothesis,
   and either way the next move is clear.
4. **What you need.** Concretely. A credential? A clarification of
   intent? Manual completion of a captcha? Permission to take a
   destructive action? An environment variable name? Vague asks
   ("any thoughts?") produce vague answers.
5. **What you'll do once unblocked.** So the user knows what they're
   authorizing and can redirect if the planned next step is wrong.

### Before / after

Vague:

> I keep getting an auth error when I try to deploy. Can you help?

Structured:

> **Goal:** Deploy the staging build to Fly.io.
> **Tried:** (1) `fly deploy` — got `Error: not authenticated`. (2)
> `fly auth whoami` — same error. (3) Checked for `FLY_API_TOKEN` in
> the environment — not set, and `~/.fly/config.yml` doesn't exist.
> **Hypothesis:** The agent environment doesn't have a Fly token; the
> user has one locally that hasn't been exposed here.
> **Need:** Either `FLY_API_TOKEN` exported in this session, or
> instructions to run `fly auth login` interactively on the user's
> side and confirm when done.
> **Next:** Once authenticated I'll re-run `fly deploy` and report
> the deploy URL plus any build warnings.

The structured version is six lines longer and saves the round-trips that
would otherwise be spent extracting each piece.

### Propose-and-pause when you can

When you can describe two or three viable paths, present them and ask the
user to pick rather than asking an open question. Recognition is faster
than recall; picking is a decision, while open questions are
discussions. If the choices have meaningfully different consequences,
include a one-sentence tradeoff per option.

This variant is the right one for ambiguous-intent cases — see Scenario 3
below.

## When NOT to ask

The cost side of the ledger. Premature asks have specific failure modes:

- **They train the user to ignore you.** If three of your last five asks
  were unnecessary, the fourth real one gets dismissed reflexively. The
  signal-to-noise ratio of your asks is itself a resource.
- **They surrender agency the user delegated.** Asking which test
  framework to use when the user said "add tests" is reading their
  request in reverse. They handed you the decision; handing it back is
  a small failure of nerve.
- **They front-load decisions that should be made later.** The right
  order is usually *cheap exploration first, then the question.* Many
  questions evaporate after five minutes of grep.
- **They multiply round-trips.** One well-formed ask now is cheaper than
  one thin ask now plus another thin ask in five minutes. If you can
  foresee the follow-up, fold it in.

### The five-minute exploration heuristic

Before asking, do the cheap work: grep the codebase, read the obvious
files, run a small probe, check `--help`, read the error's full stack.
If five minutes of exploration would either make the question
unnecessary or sharpen it considerably, do that first.

This isn't a delay tactic — it's how you make sure the ask, when it
comes, is well-formed enough to be worth the interruption.

## Concrete scenarios

### Scenario 1 — Auth loop on a website

The agent is driving a browser to download a report from an internal
admin tool. The login page redirects to an SSO provider, which then
prompts for an MFA code from the user's phone.

**Wrong:** Keep submitting the login form with guessed credentials.
Worse: fabricate plausible-looking credentials. Worse still: claim the
report has been downloaded when it hasn't.

**Right:** Stop on the first auth challenge that requires state the
agent doesn't have. Surface what page was reached, what the prompt is
asking for, and what the next step would be. Use the structured-ask
shape:

> **Goal:** Download today's revenue report from the admin dashboard.
> **Tried:** Navigated to `/admin/reports` — redirected to SSO login.
> Submitted the email I was given; SSO is now prompting for an MFA code
> sent to your phone.
> **Hypothesis:** I can't receive your MFA code; this needs you in the
> loop.
> **Need:** Either complete the MFA step in a handed-off browser
> session, or paste the code here within the ~30s window.
> **Next:** Once authenticated I'll click "Export → CSV" on the
> revenue dashboard and save it to `./reports/`.

The signals that fired: access (MFA), and after one retry, repetition.
Two signals → stop.

### Scenario 2 — Interactive terminal hang

The agent runs `npm install` and the package manager prompts
`Need to install the following packages: vite@5.0.0. Ok to proceed? (y)`.
The agent's tool harness can't see the prompt as anything other than
an unresponsive process.

**Wrong:** Send `y\n`, then `yes\n`, then `\n`, then kill and retry
with `--force`, then with `--legacy-peer-deps`, then…

**Right:** On the hang signal, stop. Kill the process. Report what
command was run, what the last visible stdout line was, and ask:

> **Goal:** Install dependencies for the new `vite` setup.
> **Tried:** `npm install` — process appears to be waiting for an
> interactive `Ok to proceed? (y)` prompt I can't see or respond to.
> **Hypothesis:** `npm` (or `npx`) is dropping into an interactive
> confirmation that needs `--yes` or equivalent to bypass in a
> non-interactive shell.
> **Need:** Either run `npm install` yourself once, or confirm I
> should retry with `npm install --yes` (or the equivalent for your
> setup).
> **Next:** Once dependencies are installed I'll run `npm run build`
> and report the output.

The signal: hang. One signal, but the deliberately-different next
attempt (a non-interactive flag) is itself a guess that could
compound — preferable to confirm before retrying.

### Scenario 3 — Genuinely ambiguous intent

User says: "Make this function faster." The function does I/O, calls a
slow third-party API, and has a hot inner loop. Three different
optimizations are reasonable; they touch different files and have
different tradeoffs.

**Wrong:** Pick one, grind for an hour, present a refactor the user
didn't want and now has to either accept or politely reject.

**Right:** Propose-and-pause. The signal is ambiguity, the structure
is multiple-choice rather than open-ended:

> **Goal:** Make `fetchUserDigest` faster, but I want to confirm which
> bottleneck to target — they have different tradeoffs.
> **Three candidates I see:**
> 1. **Cache the third-party API response** (5–10× speedup on warm
>    paths; adds a cache-invalidation concern and a new dep).
> 2. **Parallelize the per-user I/O** (2–3× speedup, contained
>    change; raises peak DB connection count).
> 3. **Vectorize the inner aggregation loop** (~1.5× speedup, fully
>    local change; only matters if profiling shows it dominates).
> **Need:** Which of these (or a different bottleneck I missed)?
> **Next:** I'll implement the chosen one with a benchmark before/after,
> and not touch the other two.

This is the most subtle case — there's no external blocker, just the
agent recognizing that grinding without confirmation is the worse move.
The retrospection step is the whole skill.

### Thin scenario — missing credential or external resource

Same shape as Scenario 1, more compressed. The agent needs an OpenAI
API key, AWS credentials, a private package registry token, or a data
file that isn't in the workspace. The right move is the same:
structured ask, name the missing resource, state where the agent
expected to find it (env var, config file, mounted volume), state
what comes next.

## Returning to autonomy after help arrives

The skill doesn't end when the question is sent. When the user
answers:

- **Acknowledge what unblocked you and proceed.** Don't ask a follow-up
  clarification before doing any work — you've already used one
  interruption; doubling it without producing output is the worst
  outcome.
- **Recalibrate the original signal.** If you asked because of an
  ambiguity signal and the user resolved it, the rest of the task may
  now be straightforward. Don't keep asking out of caution.
- **Capture environmental facts.** If the user provided a credential
  location, an env var name, a flag, or a path, that fact is now part
  of the working context. Reuse it; don't re-ask in five turns.
- **Don't apologize.** A well-formed ask is a contribution to the task,
  not a failure. Apologetic framing trains the user to wonder whether
  they should have anticipated the need and offered help unprompted.

The autonomy-after-help posture is just normal autonomy with one extra
piece of context — not a chastened, second-guessing version of it.

## Heuristic summary

A short list to re-read mid-task without re-reading this skill:

- **Two signals present?** Stop.
- **Five-minute exploration done?** If not, do it before asking.
- **Is the ask structured (goal / tried / hypothesis / need / next)?**
  If not, rewrite it before sending.
- **Could this be propose-and-pause instead of an open question?**
  Prefer that when you can.
- **Have I asked something similar this session?** Don't double-ask;
  reuse the prior answer.

## What this skill is not

- Not a license to ask reflexively. The default is autonomous progress;
  this skill activates a specific exception, not a new baseline.
- Not a substitute for reading errors carefully. The error text is
  almost always more informative than the agent's first scan suggests.
- Not for stylistic preferences the user has delegated by their
  request's framing.
- Not for asking permission for routine actions already authorized in
  `CLAUDE.md`, the session's prior turns, or standing instructions.
  Those authorizations exist precisely so the agent doesn't have to
  re-ask.
