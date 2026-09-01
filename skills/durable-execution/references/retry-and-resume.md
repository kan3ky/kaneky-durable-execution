# Retry, budgets and what a resume replays

The half of durable execution that decides how much a failure costs and whether
a restart pays for it again.

## Classification before retry

Retrying everything with a backoff is not a policy. A refused permission, a
malformed argument and a missing tool fail identically on every attempt.
Retrying them spends the budget, delays the only useful signal, and turns one
clear error into a slow one.

Three verdicts, and the third is the one people leave out:

- **retryable** — timeout, 429, connection reset. The same call can plausibly
  succeed later.
- **terminal** — refusal, malformed input, missing tool. Trying again cannot
  change the answer.
- **unclassified** — nothing said either way.

**Unclassified is not terminal.** It is *our ignorance*, not a verdict about
the error, and it must be a distinct recorded state. A report saying
"unclassified" reads as a prompt to classify the error; a report saying
"terminal" claims a judgement nobody made, and nobody ever goes back to check
it.

### Why the default is deny

The costs are asymmetric:

- Guessing **retryable** spends the budget, delays the error, and can re-fire a
  side effect that already happened.
- Guessing **terminal** costs one clear error, delivered immediately, to
  somebody who can act on it.

The cheap mistake is the correct default. The zero-value policy — what a caller
who has not thought about retries gets — permits exactly one attempt. Assert
that in a test, including for a negative or nonsensical budget; a caller who
said nothing must not get a retry loop.

### Where the verdict comes from

Precedence, highest first:

1. **A marker on the error itself**, attached by the layer that saw the
   failure. Read it off the error chain and take the **outermost** marker: the
   layer closest to the caller knows most about what the failure means to *this*
   step.
2. **A classifier supplied by the workflow**, consulted only for errors that
   carry no marker. This lets a workflow teach the engine its own error shapes —
   an HTTP status, a provider code — without every tool having to wrap.
3. **Unclassified**, meaning do not retry.

Two properties the marker must have: it must **not swallow the error it
carries** (callers still need to match on what actually failed), and marking a
nil error must stay nil rather than inventing a failure out of a success.

## The four failure kinds

A single `failed` flag cannot say what happened. Persist a kind alongside it:

| Kind | Meaning | Settled? | What a resume does |
|---|---|---|---|
| open | attempts spent, budget unsettled — the process died or the context ended mid-retry | no | continues spending the attempts that remain |
| terminal | classified as not worth retrying, and not retried | yes | replays the failure |
| unclassified | nobody said it was retryable, so it was not retried | yes | replays the failure |
| exhausted | retried, and the budget ran out still failing | yes | replays the failure |

Replaying a **settled** failure rather than re-running it is what makes a retry
budget survive a suspend. Skip it and "retry everything forever" comes back
wearing a resume — every wake of a long-parked run re-attempts a refusal that
has already been refused.

The settling rule is small enough to test exhaustively:

- terminal → terminal, always, regardless of budget;
- retryable with attempts remaining → open;
- retryable at **or past** the budget → exhausted (past matters: a stale count
  must not reopen a spent budget);
- anything else → unclassified.

Enforce the vocabulary in **storage as well as code**: a check constraint on
the permitted kinds, and a second one asserting a completed step carries no
failure kind. A typo written by a future caller is otherwise invisible until
something quietly fails to match it.

## Persisting the budget

- **The budget counts executions across ALL replays**, not per resume, and not
  calls into the step wrapper.
- **Write after every attempt**, not once at the end. A crash between attempt
  two and three must leave two spent. It is one small write per failure, and
  failures are already the slow path.
- **Keep the write alive past cancellation.** When the caller's context has
  ended, do the bookkeeping write on a detached context with a small grace
  timeout. A cancellation that also loses the attempt count is how an exhausted
  budget quietly becomes a fresh one — and it happens at exactly the moment the
  system is under stress.
- **A dead context is not a spent budget.** On cancellation, return **without**
  settling the failure. The step keeps its remaining attempts for a later
  resume instead of being recorded as having used them up while nothing was
  running.

## Backoff

Exponential from a base, doubling, capped at a maximum, with sane defaults when
either is unset. Nothing exotic.

**Jitter is deliberately absent, and the reason belongs in the comment.** Jitter
exists to de-correlate many clients hammering one server. A single local process
retrying one step has nothing to de-correlate, and a deterministic delay is one
a test can assert. In a fleet of workers against a shared dependency the answer
flips — which is exactly why the reasoning has to be written down rather than
the behaviour merely chosen.

Make the sleep injectable, and record the delays in tests so the schedule is
still asserted rather than skipped. A test that really sleeps is a test nobody
runs.

**But do not let the injected sleep own the cancellation check.** The loop must
test the context itself, before waiting, precisely *because* the wait is
caller-supplied — an implementation that ignores its context (including the
obvious test double that returns nil) spins the loop through the whole budget
after the run has been torn down. This was a real defect, found by a test that
cancelled mid-step and asserted the attempt count.

## What a resume replays, in order

1. **A record whose fingerprint differs from the current definition** → the run
   is poisoned. Neither replay nor re-run. See the SKILL for why re-running
   downstream is unsound.
2. **A completed step** → return the stored result; the function is not called.
   This is the guarantee that makes replay affordable.
3. **A settled failure** → replay the failure; the function is not called.
4. **An open budget** → resume spending the attempts that remain.

Note that (1) applies to recorded **failures** as well as successes. An edited
definition might have succeeded where the old one failed, so replaying the old
failure is the same lie in the other direction.

## Reads on the replay path

**A missing row and a read error are different facts.** Treating a query
failure as "this step never ran" re-executes recorded work: for a paid model
call, paying twice; for a side effect, doing it twice. Distinguish them at
every read, and let the error propagate.

The mirror case: **if the work succeeded but recording it failed, do not report
success.** The next replay would silently repeat it. Return the error, and
return an outcome carrying **no status** — the journal genuinely does not know
what state the step is in, and any existing status would be a claim it cannot
support. Keep both errors in the chain: the step's own failure is what the
caller is handling, the write failure is why there is no record of it.

## Waking a parked run exactly once

A run parks on a wait key and consumes nothing while it waits — no goroutine,
no timer, only a row. Two properties on the wake:

- **Clear the wait key in the same statement that flips the status.** An
  at-least-once event source will deliver the same signal twice, and the second
  delivery must then match nothing.
- **Parking is a compare-and-swap on status**, and losing it is not a database
  error. Check the affected-row count: reporting a lost race as a win strands a
  finished run forever.

## The reset hatch

The only way past a poisoned run or a settled failure is an explicit,
per-step reset that forgets the record so the next replay executes it again.

- **Nothing calls it automatically.** It re-runs the step, side effects
  included, and that is a decision a human makes with the specific step in
  front of them.
- **It refuses an empty key**, so "reset every step in this run" is not one
  typo away.
- **Its documentation states the cost**, not just the effect.

An escape hatch that is not documented as dangerous will be used routinely,
and then the fingerprint check has been converted into a prompt people click
through.
