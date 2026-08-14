# log — the findings log

`autoresearch-results.tsv`, 830 rows, append-only. One row per investigated
question: what was asked, what was measured, what the verdict was — including
the refutations, which are kept deliberately.

**If you are about to test a hypothesis, grep this first.** It may already be
settled, or already dead. The log stands in for the git history of an analysis
workspace that is not published.

## Reading it

The header names the six columns of the original schema
(`iteration, binary, metric, runs, status, description`). The convention
drifted as the work changed shape, and rows are not all the same width — 450
rows have six fields, 291 have four, the rest scatter between three and ten.
The later and commonest shape is:

```
id    date-or-verdict    subject    finding    method    artefacts
```

Read the row, not the header. A row is a record of one measurement, written
when it was made.

## What is in the artefacts column

Paths, as they were at the time. Several point into the private workspace
(`analysis/rtti/`, `analysis/symbols/`, `analysis/backup-shim/`) — that is
where a row's evidence physically lives, and saying so is more useful than
omitting it. Others name tool paths that have since moved: a row citing
`analysis/tools/grn_tagwalk.py` is a historical record, not a working path.
Nothing in this file is rewritten to match a later layout; the log records
what was true when the work was done.
