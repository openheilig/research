# log — the findings log

**Purpose:** The append-only findings log: what was asked, what was measured, what the
verdict was.

`autoresearch-results.tsv`, 800+ rows, append-only. One row per investigated
question: what was asked, what was measured, what the verdict was — including
the refutations, which are kept deliberately.

**If you are about to test a hypothesis, grep this first.** It may already be
settled, or already dead. The log stands in for the git history of an analysis
workspace that is not published.

## Writing a row

A new row is **exactly four tab-separated fields**:

```
id    YYYY-MM-DD    kebab-case-slug    the finding, one paragraph
```

The slug names the *result*, not the topic — `startcode-placement-closed-tag-04-is-the-position`,
not `startcode`. The verdict lives in the slug and in the text's opening
clause. Append programmatically and assert the field count before writing: a
stray tab reshapes the row silently, and every later `awk -F'\t'` reads it
wrong.

## Reading it

The header names the six columns of the *original* schema
(`iteration, binary, metric, runs, status, description`), and field 2 used to
hold a status word rather than a date. Both conventions drifted as the work
changed shape, so rows are not all the same width or the same schema — roughly
450 rows have six fields, 290 have four, the rest scatter between three and
ten; the date form took over around row 600 and is now essentially universal.

Read the row, not the header. A row is a record of one measurement, written
when it was made. **Nothing here is rewritten to match a later convention** —
repairing the file that substitutes for history is worse than the
inconsistency.

## What is in the artefacts column

Paths, as they were at the time. Several point into the private workspace
(`analysis/rtti/`, `analysis/symbols/`, `analysis/backup-shim/`) — that is
where a row's evidence physically lives, and saying so is more useful than
omitting it. Others name tool paths that have since moved: a row citing
`analysis/tools/grn_tagwalk.py` is a historical record, not a working path.
Nothing in this file is rewritten to match a later layout; the log records
what was true when the work was done.
