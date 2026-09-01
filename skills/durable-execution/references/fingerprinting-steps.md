# Fingerprinting a step

The mechanics behind "a keyed replay cache needs a fingerprint over the step's
declared inputs". Everything here is a decision that was made deliberately and
would be wrong the other way round.

## The value

```
sha256-1:<hex sha256 over length-prefixed fields>
```

Fed in order, each as `name:length:value`:

```
op:<n>:<operation>
model:<n>:<model>
prompt:<n>:<instruction text>
args:<n>:<canonical json>
```

Substitute your own field names. What generalises is the *selection rule*, the
*prefixing*, the *canonicalisation* and the *scheme tag*.

## Choosing the fields

The rule: **hash everything that changes what the result MEANS, and nothing
that changes only how or how hard it was obtained.**

Applying it produces a short list of inclusions and a longer, more interesting
list of exclusions.

### Included

- **Operation** — which tool or function was invoked. A result from one tool
  replayed as another's is not a result at all.
- **Model** — where a model-backed operation routes. Switch models and a resume
  would serve the previous one's output as though the new one had produced it.
- **Prompt / instruction text** — the recorded answer belongs to a question
  nobody is asking any more.
- **Arguments** — a limit of 10 replayed for a request for 50 under-reports and
  looks entirely fine.

### Excluded, each for its own reason

- **The lookup key.** It is the address the row is found by, not behaviour.
  Hashing it makes the lookup circular, and a changed key is already a
  different step — it finds no record and executes.
- **The retry budget and its backoff.** These change how hard we try, never
  what the answer means. A result computed under three permitted attempts is
  the same result as one computed under five. Include it and every operator who
  tunes a timeout poisons every parked run in the system — a self-inflicted
  outage caused by the safety mechanism.
- **Run id, user, wall clock, anything per-run.** Include one and every step
  mismatches on every run, which is indistinguishable from having no check at
  all. Worse: the failure mode is a flood of poisoned runs, which trains
  everyone to ignore the state.

A useful test for a borderline field: *if two runs differed only in this, would
I be happy for one to return the other's cached answer?* Yes means exclude.

## Length prefixing

Without delimiters, `op="ab", model="c"` and `op="a", model="bc"` hash
identically. Two genuinely different steps then agree that nothing changed —
the exact silence the fingerprint exists to break.

A separator character is not enough on its own, because a value containing the
separator reintroduces the ambiguity. Prefix each field with its byte length.
Pin it with a test that asserts the two colliding definitions differ; it is
three lines and it fails loudly if someone simplifies the encoding later.

## Canonicalising structured arguments

Two calls passing the same arguments with different whitespace, or in a
different map iteration order, describe **the same step**. A byte-wise hash
reports them as edits.

This matters more than it sounds. **A check that cries wolf gets switched off,
and switching it off reopens exactly the gap it guards.** False positives here
are not a nuisance — they are how the mechanism dies.

So: decode, then re-encode with a canonical form (most JSON encoders sort
object keys on encode; verify yours does).

Two traps in the decode:

- **Number precision.** The default decode of a JSON number into a generic
  value goes through a 64-bit float, so `10000000000000001` and
  `10000000000000002` — two different record ids — canonicalise to the same
  text and fingerprint identically. Decode numbers as their literal text.
- **Trailing content.** `{"a":1} {"b":2}` decodes successfully as `{"a":1}` if
  you stop at the first value. Two different argument sets then share a
  fingerprint. Reject anything after the first value rather than silently
  hashing a prefix of the input.

And normalise the empty cases together: absent, empty, whitespace-only and an
explicit null are all the same step, so canonicalise them all to one value.

## Stamping the scheme

Prefix the stored value with a tag naming the recipe, not merely the algorithm
— `sha256-1`, incremented when the *field selection* changes even if the hash
function does not.

This is what makes a future change to the recipe self-announcing. Without it,
new fingerprints are compared against old ones computed under different rules,
and they disagree for a reason the error message cannot explain — or, worse,
agree by accident. With it, every in-flight run mismatches loudly on its next
resume, which is the correct and intended behaviour for a recipe change.

## Rows written before the fingerprint existed

A migration that adds the column defaults it to empty. Empty must read as a
**mismatch**, and the error message must say so in words rather than showing a
blank: *"recorded before step versioning"*.

"We do not know what computed this" is not a reason to trust it. The
alternative — treating an empty recorded fingerprint as "matches anything" —
gives exactly the runs most likely to be stale (the oldest ones) a permanent
exemption from the check.

## The limit, stated where the type is defined

The hash covers the **declared** inputs, not the function body. Someone who
rewrites the closure while leaving the declared fields alone gets no warning.

That makes the contract explicit and it belongs in the doc comment on the
definition type: **the step function must be a pure function of its
declaration — anything it closes over that can change belongs in the
arguments.**

A guard whose boundary is undocumented gets trusted past its boundary. Naming
the limit is what lets a reviewer notice a step capturing a mutable value.

## Refusing rather than degrading

A step whose fingerprint cannot be computed — no key, malformed arguments,
trailing content — must **refuse to run**, with a distinct error kind callers
can match on. A step with no stable version is a step whose replay can never be
checked, so running it produces a record that is unverifiable forever.

Do not fall back to hashing the fields that did parse. A partial fingerprint is
worse than none, because it looks like a fingerprint.

## Testing it: assert the negatives

The valuable assertions are the ones about what must **not** move. A table with
a `differs` boolean and a one-line reason per case documents the design while
testing it:

| Mutation | Fingerprint moves? | Reason |
|---|---|---|
| nothing | no | an unchanged step must replay, or every resume dies |
| prompt edited | yes | the recorded answer belongs to a question nobody is asking |
| operation changed | yes | a result from one tool replayed as another's is not a result |
| model changed | yes | the operator would be served the previous model's output |
| an argument value changed | yes | a smaller limit replayed for a bigger request under-reports |
| argument keys reordered | **no** | iteration order is not an edit |
| arguments reformatted | **no** | whitespace is not an edit |
| key changed | **no** | the key is the address; a new key is already a new step |
| retry budget changed | **no** | tuning a timeout must not poison sound journals |

Plus three standalone cases: the field-collision pair, two 17-digit ids that
must stay distinct, and the refusal cases.

Also assert that the stored value **starts with the scheme tag**. That single
check is what catches a future edit that changes the recipe without
incrementing the name.

## The error a mismatch produces

It carries the run id, the step key, **both** fingerprints, and a sentinel the
caller can match on.

Both fingerprints, because "it changed" without the pair is not actionable: the
operator cannot distinguish a real workflow edit from a scheme change, and
those need different responses. The sentinel, because a caller that cannot tell
this apart from any other failure will bucket it with retryable errors and
resume the run forever.
