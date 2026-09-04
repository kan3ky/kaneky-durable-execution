---
name: durable-execution
description: Make work that stops and resumes come back correct — durable workflows, step journals and replay caches, retry budgets that must survive a restart, and runs that park for days waiting on a human. Covers the fingerprint that stops a resumed run replaying a result its current code never computed, why an unclassified failure must not be retried, where a retry budget has to live, and how a resumed run reports which steps actually ran. Use when building or reviewing a workflow engine, a job queue with retries, webhook or event redelivery, a saga or compensating transaction, idempotency keys on a payment API, or any checkpoint-and-resume path — and after a resumed run reported success while producing output nobody can account for.
---

# Durable execution

The model is well established: a run's function re-executes from the top on
resume, and every step that already completed returns its **recorded** result
instead of running again. That is the entire guarantee, and it is what makes
replay affordable — an expensive call inside a step is paid for once, however
many times the run wakes up.

It also contains an assumption nobody states:

> A keyed replay cache is a claim that the code behind the key still computes
> the same thing.

Nothing enforces that claim by default. The run parks, someone edits the
workflow, the run resumes, and the cache answers for a program that no longer
exists — under a key the new code also claims. The run completes. It reports
success. Its output is a mixture of two different programs, and there is no
symptom.

Everything below is about the moment work stops and starts again: what has to
be recorded, what has to be checked before a recording is trusted, and what a
resumed run is allowed to claim about itself.

## The shape

Three parts, and every failure in this file is one of them going wrong:

1. **An address** — a step key, a message id, an idempotency key. Stable by
   design, because that is what makes the lookup work.
2. **A recorded result** — the value replay returns instead of re-running.
3. **A program that can change under both**, at any point while the run is
   parked, with nothing linking the change to the record.

The address survives the edit. That is the problem: it is the one thing the
system designed to be permanent, and the code it points at is the one thing
designed to change.

## 1. The replay key needs a fingerprint over what the step DOES

A step key is an address, not a description. Two things are needed and they
answer different questions: `Key` says *which row*, a fingerprint says *what
computed it*.

**The fingerprint must be DERIVED, never assigned.** A hand-maintained
`version: 3` field is a field somebody has to remember to bump while they are
busy changing the thing it describes — which is the same class of defect it was
added to catch. Hash the step's declared inputs and there is nothing to
remember.

The recipe that worked, in full detail in `references/fingerprinting-steps.md`:

```
sha256-1:<sha256 over length-prefixed (op, model, prompt, canonical-json args)>
```

### What is hashed, and what is deliberately not

| Field | In the hash? | Why |
|---|---|---|
| Operation / tool name | **yes** | A result from one tool replayed as another's is not a result. |
| Model | **yes** | The operator switched models and would be served the old one's output. |
| Prompt / instruction text | **yes** | The recorded answer belongs to a question nobody is asking now. |
| Arguments | **yes** | A limit of 10 replayed for a request for 50 under-reports and looks fine. |
| **The lookup key** | no | Hashing the address the row is found by is circular, and a changed key is already a different step. |
| **The retry budget** | no | It changes how hard we try, never what the answer means. Hashing it poisons every sound journal the moment somebody tunes a timeout. |
| **Run id, user, clock** | no | Per-run values make every step mismatch on every run, which is indistinguishable from having no check at all. |

Those three exclusions are the interesting part of the design, and the second
one is the one people get wrong: retry policy *feels* like part of the step. It
is not. A result computed under three permitted attempts is the same result as
one computed under five.

### Four encoding traps that turn the check into decoration

- **Length-prefix the fields.** Without it `Op="ab", Model="c"` and `Op="a",
  Model="bc"` hash identically, and two genuinely different steps agree that
  nothing changed. Concatenating fields into a hash is the default thing to
  write and it is wrong.
- **Canonicalise structured arguments before hashing.** Two calls passing the
  same arguments with different whitespace, or a different map iteration order,
  describe the same step; a byte-wise hash calls them edits. **A check that
  cries wolf gets switched off, which reopens exactly the gap it guards** —
  false positives are not a cosmetic problem here, they are how the mechanism
  dies.
- **Do not round numbers while canonicalising.** Decoding JSON into generic
  values sends integers through a float by default, so `10000000000000001` and
  `10000000000000002` — two different record ids — become the same step. Decode
  numbers as text.
- **Stamp the algorithm into the stored value** (`sha256-1:`). The prefix names
  *what goes into* the hash, not merely how it is computed, so changing the
  recipe later is self-announcing: in-flight runs mismatch loudly on their next
  resume instead of comparing new fingerprints against old ones and agreeing by
  accident.

### Refuse what you cannot fingerprint

A step with no stable version is a step whose replay can never be checked, so
the correct response to unhashable input — no key, malformed arguments,
trailing content after the JSON value — is to refuse to run it, not to hash
what you can and continue.

And the migration case: rows written **before** versioning existed carry an
empty fingerprint. Empty must read as a **mismatch**, with a message that says
so in words ("recorded before step versioning"). "We do not know what computed
this" is not a reason to trust it.

### The honest limit, which belongs in the doc comment

This hashes the **declared** inputs, not the function body. A caller who
rewrites the closure while leaving the declared fields alone gets no warning.
That makes the contract explicit and worth writing down where the type is
defined: *the step function must be a pure function of its declaration —
anything it closes over that can change belongs in the arguments.*

State the limit. A guard whose boundary is undocumented gets trusted past it.

## 2. On mismatch, refuse — and know why "just re-run downstream" is wrong

The friendly-sounding alternative is to re-run the changed step and everything
downstream of it. It is unsound in any system that keys steps without linking
them:

- **There is no DAG.** Nothing records that step C consumed step B's output, so
  "downstream" is guessable only from creation order.
- **That guess is wrong the moment a result escaped through a side effect a
  re-run cannot retract** — a message sent, a file written, a payment made.
  Silently re-firing those is worse than stopping.

So the run stops, and it stops in a state of its own:

- **`poisoned` is not `failed`.** The two deserve different answers: a failed
  run may be worth resuming, a poisoned one never is — its journal is no longer
  a faithful record of the program that produced it, so no amount of resuming
  makes it correct. One status covering both means an operator's "retry the
  failures" sweep silently includes runs that must not be retried.
- **A poisoned step is neither executed nor replayed**, so it belongs in
  neither count. Filing it under either makes one of the two numbers a lie.
- **The error names the key and BOTH fingerprints.** "It changed" without the
  pair is not actionable — the operator has to be able to tell a real edit from
  a scheme change.
- **A poisoned run needs its own read path.** It waits on no signal and no
  retry reaches it, so it is the one state nothing else will ever surface. If
  the only place it appears is a log line, the loud failure is loud in a file
  nobody opens. Give it an index and an operator list.

**State the cost honestly, in the code, at the decision:** editing a workflow
while runs are parked **breaks those runs**. A long-suspended run dies on
resume instead of adapting. That is deliberate, and the trade is worth naming
because someone will otherwise "fix" it: a dead run with a named cause is
recoverable by a human in one decision — start it again, or accept the old
output. A run whose output blends two programs is not recoverable at all,
because nobody finds out it happened.

The escape hatch is a **manual, per-step reset** that forgets the record so the
next replay executes it again. Nothing may call it automatically, and it should
refuse an empty key so that "reset every step in this run" is not one typo
away. Its docstring says what it costs: it re-runs the step, side effects
included.

## 3. Retry defaults to DENY

Retrying everything with a backoff is not a policy. A refused permission, a
malformed argument and a tool that does not exist fail identically on every
attempt: retrying them spends the budget, delays the only useful signal, and
turns one clear error into a slow one.

So a failure is **classified before it is retried**, and the classification has
three values, not two:

- **retryable** — a timeout, a 429, a connection reset. The same call can
  plausibly succeed later.
- **terminal** — a refusal, a malformed input, a missing tool. Trying again
  cannot change the answer.
- **unclassified** — nobody said either way.

**Unclassified means not retried, and it is not the same as terminal.** It is
our ignorance, not a verdict about the error, and it must be recorded as its
own state so the report says "nothing classified this" rather than claiming a
judgement nobody made. It reads as a prompt to classify the error.

The asymmetry is what makes deny the right default: guessing "retryable" costs
the budget and can re-fire a side effect; guessing "terminal" costs one clear
error the operator sees immediately. **The cheap mistake is the correct
default.**

Two structural notes that make the classification usable:

- **Let the error carry its own verdict**, wrapped at the layer that knows —
  the tool that saw the 429. Read the verdict off the chain and take the
  **outermost** marker: the layer closest to the caller knows most about what
  the failure means to *this* step.
- **Let the workflow supply a classifier for unmarked errors only.** A workflow
  can teach the engine its own error shapes (an HTTP status, a provider code)
  without every tool having to wrap, and a marker on the error still beats it.

### The failure kinds are the report

A single `failed` flag cannot express the distinction that matters after the
fact. Four kinds, and the last three are settled while the first is not:

| Kind | Meaning | What a resume does |
|---|---|---|
| open | attempts spent, budget unsettled — the process died or the context ended mid-retry | continues spending what is left |
| terminal | classified as not worth retrying, and not retried | replays the failure |
| unclassified | nobody said it was retryable, so it was not retried | replays the failure |
| exhausted | retried, and the budget ran out still failing | replays the failure |

Replaying a **settled** failure rather than re-running it is what makes a retry
budget survive a suspend. Without it, "retry everything forever" comes back
wearing a resume.

Enforce the vocabulary in the **storage layer as well as the code** — a check
constraint on the allowed kinds, plus one asserting a completed step carries no
failure kind. A row written by a future caller with a typo is invisible until
something fails to match it.

## 4. The retry budget lives in the journal, not in the loop

A budget held in a local variable is a budget that resets every time the
process restarts, and "restart" is the normal case in a system built to
suspend.

- **`MaxAttempts` counts executions across ALL replays, not per resume.**
- **Persist after every attempt, not once at the end.** A crash between attempt
  two and attempt three must leave two spent. It costs one small write per
  failure, which only happens when something is already going wrong.
- **Keep the bookkeeping write alive when the caller's context has already
  died.** Do the record on a detached context with a small grace timeout.
  Otherwise the attempt count is lost exactly when it matters most — a
  cancellation mid-retry — and **an exhausted budget quietly becomes a fresh
  one**.
- **A dead context is not a spent budget.** On cancellation, stop *without*
  settling the failure, so a later resume gets the remaining attempts rather
  than a step that "used them up" while nothing was running.

On backoff: exponential from a base, capped. **Jitter was left out on purpose**
— jitter de-correlates many clients hammering one server, and a single local
process retrying one step has nothing to de-correlate, while a deterministic
delay is one a test can assert. Include the reasoning wherever you omit
something people expect, or the next reader adds it back.

Make the sleep injectable. A test that really sleeps is a test nobody runs.

## 5. A resume must say which steps actually ran

If the only outcome is "completed", that word covers a pass that executed
everything and a pass that executed nothing equally well — and the operator
cannot tell an approval that finally did the work from a replay that only
re-read it.

Every step call returns, on success *and* on failure:

- **replayed or executed**, stated positively as the claim the report makes;
- **total attempts ever**, across every replay, not just this pass;
- **the failure kind**, if it failed;
- the fingerprint it ran under.

Three details that came out of writing it:

- **The per-pass log is in memory and per-pass on purpose.** "Which steps ran
  just now" is a fact about *this attempt at the run*, not about the run.
  Persisting it immediately raises the question of which pass the stored answer
  belongs to. The durable per-step facts live in the journal and are read back
  separately.
- **Always emit the lists, even empty.** A missing key reads as "no work" when
  it means "no data" — the same absence-versus-failure collapse this whole file
  is about.
- **A replayed failure carries the recorded text and no live error value.**
  Only the text survives a restart; inventing an error object to stand in for
  it would let callers match on something that was never returned.

## 6. Two defects the tests found, and one they nearly missed

None of these were visible by reading the code. Each is a test that asserts a
*negative*.

**A caller-supplied wait function was trusted to notice cancellation.** The
retry loop called the injected sleep and let it decide whether the context was
dead. An inattentive implementation — including the obvious test double,
`func(...) error { return nil }` — spins the loop through the entire budget
after the run has already been torn down. The fix is to check the context in
the loop, before the wait, precisely *because* the wait is caller-supplied.
Generalise: **anything injectable is a place where your invariant leaves your
control.** Re-check it on your side of the boundary.

**Any journal read error was treated as "this step never ran."** A missing row
and a failed query took the same path, so a connection reset re-executed
recorded work — for a paid model call, paying twice; for a side effect, doing
it twice. This is the absence-versus-failure collapse in the one place where it
costs money. Distinguish "no row" from "could not ask" at every read on a
replay path.

**And the one adjacent to them:** if the work succeeded but the *record* of it
failed to persist, do not report success. The next replay would silently repeat
it. Return the error, and return an outcome with **no status at all** — the
journal genuinely does not know what state the step is in, and any of the
existing statuses would be a claim it cannot support.

## 7. Work started by a request must not inherit the request's context

The smallest version of "work that has to survive" is a handler that kicks off a
job and answers immediately. If the job is started on the **request's** context,
it dies the moment the response is written — and the failure is disguised, twice
over.

- **The runtime's own words become a claim about the domain.** The job records
  `context canceled` and the page renders it beside the thing the job was
  working on. A read that never happened is reported as a finding about the
  subject it never reached.
- **No test with a synthetic request can see it.** In Go, `httptest.NewRequest`
  carries a background context that nobody cancels, so a handler tested this way
  passes forever; only a real server cancels the context at the end of the
  response. The same asymmetry exists wherever a test harness fabricates the
  request object instead of making one.

Three things follow, and they are cheap:

1. **Detach deliberately** at the point of submission — `context.WithoutCancel`,
   a job-scoped context, or the worker's own — and give the work a deadline of
   its own, because "not the request's lifetime" must not mean "no lifetime".
2. **Cover it with one test over a real listener.** Start the server, post over
   a real client, let the response complete, then assert the work finished.
   That single test is the only thing that distinguishes the two contexts.
3. **Never surface a runtime's cancellation string to a reader.** Translate it:
   *this run was stopped before it read anything, so it did not happen* — and
   say explicitly that it implies nothing about the subject. A cancellation is a
   fact about your process, and a reader cannot be expected to know that.

The general shape, and it is the reason this sits in this skill: **the lifetime
of the work and the lifetime of the thing that asked for it are different, and
every layer that conflates them fails silently.** The same conflation appears
as a connection pool closed while a background flush is in flight, a shutdown
that cancels in-flight jobs it should have drained, and a cache entry tied to a
session that outlives it.

## 8. The determinism rule that makes any of this safe

Code *between* steps runs again on every replay. So anything non-deterministic
— the clock, randomness, a network call, a UUID — must happen **inside** a
step, or the run drifts from its own history: a timestamp read between steps
produces a different value each pass, and every decision downstream of it is
made on a different basis than the one recorded.

This is the rule every workflow engine states and the one every new user
breaks. Write it at the top of the package, not in a wiki.

## Where else this exact shape appears

The fingerprint-the-inputs lesson is not specific to workflow engines. Look for
it wherever a stable key returns stored work:

- **Workflow engines** (Temporal, Inngest, Restate, DBOS) — the same problem,
  solved with explicit workflow versioning and patch APIs. Their existence is
  the evidence: everyone who builds this eventually has to answer "what happens
  to in-flight executions when the code changes", and the answers are *version
  it* or *break it loudly*, never *hope*.
- **Idempotency keys on payment APIs** — the mature ones store the response
  against the key *and* a hash of the request parameters, and reject a reused
  key whose body differs rather than returning the old response. That rejection
  is exactly the poison-on-mismatch decision, made by someone who learned it
  the expensive way.
- **Job queues with retries** — the budget must ride with the message, or a
  redelivery restarts an exhausted one. Same defect, one layer down.
- **Webhook and event redelivery** — at-least-once means the second delivery of
  the same signal must wake nothing. Clear the wait key **in the same statement
  that flips the status**, so the second delivery has nothing left to match.
- **Sagas and compensating transactions** — the whole pattern exists because
  some effects escaped and can only be *compensated*, not retracted. That is
  the same fact that makes "re-run downstream" unsound.
- **Build and action caches** — a cache key over the sources but not the
  compiler version is a replay cache with no fingerprint over its inputs, and
  it returns artifacts built by a toolchain that no longer exists.
- **HTTP caching** — a cache entry with no validator, serving a body for a
  resource that changed underneath it.

If a system answers a repeated question from storage, ask what it hashes into
the key, then ask what *else* could change the right answer. The gap between
those two lists is the defect.

## What to review in any resume path

1. **Find the replay key and ask what it does NOT capture.** If the answer is
   "the definition of the work", that is the finding.
2. **Confirm the version is derived, not assigned.** A hand-bumped field is a
   TODO with a number in it.
3. **Check the exclusions have reasons written down**, especially anything
   operational — a policy, a timeout, a budget — that would poison sound
   records every time it is tuned.
4. **Try a reordered / reformatted argument object.** If the fingerprint moves,
   the check will be disabled within a month.
5. **Try two steps whose concatenated fields collide.** If they agree, the
   fields are not delimited.
6. **Ask what happens on mismatch**, and if the answer is "re-run", ask which
   already-completed step's side effect that re-fires.
7. **Confirm the mismatch state is distinct from failure** and has a read path
   an operator will actually see.
8. **Force an unclassified error** and confirm it is not retried, and that the
   record says *unclassified* rather than *terminal* or *exhausted*.
9. **Kill the process mid-retry, resume, and count the attempts.** A fresh
   budget here is the most expensive silent bug in the file.
10. **Cancel the context mid-retry and check the attempt was still recorded.**
11. **Make the journal read fail** and confirm the step does not execute.
12. **Ask a completed resume which steps it executed.** If it cannot answer,
    its success report means less than it appears to.

## A dial that moved has to reach the loop already waiting on the old one

A long-running loop usually looks like: do the work, compute the wait, sleep,
repeat. The wait is read once, when the sleep begins. That is the bug, and it
does not look like one:

```
for {
    cycle()
    wait := config.Interval()   // read ONCE
    time.Sleep(wait)            // and honoured for its whole duration
}
```

Shorten the interval from two minutes to twenty seconds and nothing happens for
up to two minutes. The setting is applied — a later read returns the new value,
the change is recorded, the UI shows it — and the running process keeps the old
schedule anyway, because it is asleep on a timer that was created before anyone
touched the dial.

**Why it survives review.** Everything about it is correct in isolation. The
store took the write. The loop reads the current config *each iteration*. Nobody
wrote `interval := 2 * time.Minute` anywhere. The defect lives entirely in the
gap between "the next iteration reads it" and "an iteration can last as long as
the old value".

**How it surfaced, and why that is worse than it sounds.** A liveness check that
compares "time since last cycle" against "the configured interval" will start
reporting the process as LATE the moment the interval is shortened — correctly,
by its own definition, because the process genuinely is past its new deadline.
So the first symptom is a health indicator accusing a healthy process, and the
tempting fix is to loosen the detector. Loosening it removes the only thing that
noticed.

**The fix is to make the sleep interruptible, and to recompute against the
current value each time it wakes:**

```
for {
    cycle()
    last := now()
    for {
        wait := config.Interval()        // re-read on EVERY wake
        remaining := last.Add(wait).Sub(now())
        if remaining <= 0 { break }      // already due under the new value
        select {
        case <-ctx.Done():  return
        case <-changed:                  // a dial moved; recompute
        case <-time.After(remaining):
        }
        if now().Sub(last) >= wait { break }
    }
}
```

Two properties worth keeping: the deadline is measured from the LAST CYCLE
rather than from the moment of the change, so shortening an interval does not
grant a free extra run and lengthening one does not cost a cycle; and the wake
channel is buffered to one with a non-blocking send, because what the loop does
on waking is read the current configuration, so one pending wake is worth
exactly as much as three.

**Where else the same shape appears:** a poll interval, a retry backoff ceiling,
a lease renewal period, a batch window, a rate limit, a cache TTL held in a
sleeping refresher. Anything where a duration is read once and then honoured for
its own length.

**The test that catches it.** Not "does the setting take effect" — it does.
Start the loop on a long interval, wait for one cycle, shorten the interval,
then assert the NEXT DEADLINE moved, without waiting for the old one to expire.
Assert the other direction too: lengthening must not cost a cycle. A test that
only checks the value after the sleep will pass against the broken version.

## Adjacent skills

- **integrations** — at-least-once delivery, deduplication across redelivery,
  and an empty result that is really a dead source. The same absence-versus-
  failure collapse, seen from the data side.
- **capability-honesty** — a resume reporting "completed" without saying what
  it executed is a declaration nobody checked, and the report section above is
  that skill's requirement applied here.
- **agent-guardrails** — the security half of a side effect that cannot be
  retracted: which capabilities exist for this caller at all.
- **diagnosis** — when the symptom of a poisoned replay surfaces somewhere
  unrelated, days later, as output that does not match any code you can find.

## References

- `references/fingerprinting-steps.md` — the hash recipe end to end: field
  selection with reasons, length prefixing, JSON canonicalisation and number
  precision, scheme stamping, unversioned rows, the closure limit, and the
  mutation table that tests it.
- `references/retry-and-resume.md` — classification, the four failure kinds,
  persisting the budget, cancellation grace, backoff without jitter, replaying
  settled failures, and the manual reset hatch.
