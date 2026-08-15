# Research discipline

**Status:** Standing
**Purpose:** The rules that keep a measurement honest. Read before measuring anything.

Every rule here was learned by getting it wrong first. They are cheap to read
and expensive to rediscover.

## Two decoders or it isn't known

A format is understood when **two independently written decoders agree**. One
decoder that looks plausible is not evidence — it is a hypothesis that has
not been tested.

"Independently written" is load-bearing. A second implementation
transliterated from the first makes the parity diff *structurally incapable
of failing*: it will agree with the original's bugs. The Python and GDScript
`.GRN` walkers were each written from a specification and from direct byte
reads, never by opening the other. That is why their agreement means
something.

Gates that enforce it: `verify_ref.py` vs `verify.gd`, `grn_parity.sh`,
`replay_diff.sh`.

## An import is not a call

`gtk_init_check` plus 326 nearby `strtod` sites made the locale mechanism
look thoroughly established. Interposing `setlocale` showed **zero calls**.

If the question is "is this mechanism used", the answer comes from
interposing it and counting, never from grepping for it.

## A clean arm without a control is not a result

`taskset -c 0` came back 20/20 clean and looked like a fix. The control also
came back 20/20.

Any measured arm needs a control arm run under the same instrument in the
same session. This applies to hit rates too: `pax_c8.gd` tests the real ids
against 70 random ids in the same numeric range, because without that a hit
rate means nothing.

## A truncated scan is not a negative

The instruction-query tool **silently caps at 200,000 instructions**, setting
`truncated: true` and a `next_start`. A `count: 0` from a truncated scan is
indistinguishable from a real negative.

Chunk until *every* chunk reports `truncated: false`. And read matches rather
than counting them — `95` decimal is `0x5F`, `'_'`, so a numeric scan is
noisy with identifier parsing.

## A metric can be hollow

"Parse rate" for the script bytecode looked healthy while the parser was
reading the wrong offsets, because every unlisted tag is a zero-width no-op
and absorbed the garbage silently. The honest metric was **consumes its body
exactly**.

Before trusting a number, ask what it would look like if the thing being
measured were completely broken. If the answer is "about the same", it is not
a metric.

Related: `race_sweep.sh`'s metric was invalidated **four times** before being
rewritten. Never compare a new sweep against an old table.

## Case sensitivity has cost us a false finding

A comparison loop using lowercase filenames reported three shipped `.bin`
files missing from the Windows disc. Windows ships them capitalised; all
eight are byte-identical. Had that not been re-checked, a false "Linux-only
data files" claim would have entered the record.

## Refutations are findings

The findings log records `CORRECTION` and refuted hypotheses on equal footing
with confirmations, and this repository keeps retracted claims as footnotes
rather than deleting them. Re-deriving a refutation costs more than storing
it, and "do not retry this" is often the more useful half of a result.

## Clean-room boundary

`community/unpack-tools/` and outside projects such as Iris1 are
**documentation of behaviour**, never a source to copy from. Tag numbers,
node names and record layouts are facts about a file format and are used as
such; code is not read, translated or adopted. This mattered for correctness
before it mattered for licence provenance — a borrowed template is a
hypothesis you did not test.

Where an outside reading was consulted and found wrong, the disagreement is
recorded: Iris1 reads `0x0C02..0x0C05` as a start/end bracket; they are
distinct node types.

---
Provenance: every rule here traces to a `CORRECTION` row in
`log/autoresearch-results.tsv`.

