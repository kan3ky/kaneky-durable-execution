# kaneky-durable-execution

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-6C4FF7)](https://docs.claude.com/en/docs/claude-code)

A Claude Code skill for work that stops and starts again — durable workflows,
step journals, replay caches, retry budgets that have to survive a restart, and
runs that park for days waiting on a human.

The guarantee everyone builds first is that a step which already completed
returns its recorded result instead of running again. The assumption nobody
states is underneath it.

## The question it asks

> A keyed replay cache is a claim that the code behind the key still computes
> the same thing. What enforces that?

Usually nothing. The run parks, someone edits the workflow, the run resumes,
and the cache answers for a program that no longer exists — under a key the new
code also claims. The run completes, reports success, and its output is a
mixture of two different programs.

## Install

```sh
git clone https://github.com/kan3ky/kaneky-durable-execution
cp -r kaneky-durable-execution/skills/durable-execution ~/.claude/skills/
```

No configuration. No dependencies.

## Use it

Ask in plain language. The skill loads when the work matches it.

```
Review this workflow engine's replay path before we let runs suspend for days.
Our retry budget resets whenever the worker restarts — where should it live?
This resumed run said completed and produced output nobody can account for.
Design idempotency keys for this payment endpoint.
Should an unclassified error be retried?
What breaks if we edit a workflow while runs are parked?
```

## What it covers

| The decision | The answer it argues for |
|---|---|
| Keying a replay cache | Add a fingerprint over the step's declared inputs, **derived** — a hand-bumped version field is a TODO with a number in it |
| What goes in the hash | Not the lookup key (circular), not the retry budget (tuning a timeout would poison sound journals), not per-run values (everything mismatches, which equals no check) |
| Encoding it | Length-prefix the fields, canonicalise structured arguments, keep number precision, stamp the algorithm into the value |
| A mismatch on resume | Refuse. Without a graph, "re-run downstream" is a guess, and it is wrong the moment a result escaped through a side effect a re-run cannot retract |
| Naming that state | `poisoned`, distinct from `failed` — a failed run may be worth resuming, a poisoned one never is |
| An unclassified failure | Do not retry. That is our ignorance, not a verdict, and it is a distinct state from "retried and exhausted" |
| Where the budget lives | In the journal, written after every attempt, and written even when the caller's context has already died |
| What a resume may claim | Which steps executed and which were replayed — otherwise "completed" covers a pass that did everything and one that did nothing |

Two reference files carry the detail: the hash recipe end to end, and
retry/resume mechanics.

## Where the shape repeats

Workflow engines, job queues, webhook and event redelivery, sagas and
compensating transactions, idempotency keys on payment APIs, build caches, HTTP
caching. Anywhere a stable key returns stored work, ask what the key captures
and then what *else* could change the right answer. The gap is the defect.

## Part of a collection

One of a set of Claude Code skills about failures that look like success — the
ones that pass review, deploy green, and are wrong anyway.

```sh
/plugin marketplace add kan3ky/kaneky-skills
```

The [collection](https://github.com/kan3ky/kaneky-skills) lists every skill.

## Licence

MIT.
