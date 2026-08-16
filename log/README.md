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

Every row is now the four fields above, and `tools/parity/log_check.py`
enforces it from the first row. It was not always so: the file grew through
five schemas — an `iteration, binary, metric, runs, status, description`
table, then several shapes of three, five and six fields where field 2 held a
status word rather than a date — and three rows where raw tab-separated tool
output leaked in and split the paragraph.

Those 492 rows were normalised into the current contract, and **no text was
lost**: each extra column is folded into the paragraph in its original order,
joined with ` | `, and the status tag (`VERIFIED`, `REFUTED`, `CORRECTION`, …)
leads the paragraph wherever it is not the slug. A row that had only a tag to
name it has that tag, lowercased, as its slug — a poor slug, but a true one,
where a slug guessed from the prose would have been invention.

**The dates on those rows are inferred, not measured.** They never carried
one, and the repo's first commit imported the whole log at once, so `git blame`
dates every legacy line to the import rather than to the work. Each takes the
date its own prose names — where that date falls inside the window its
neighbours allow — otherwise the nearest dated row above it. Treat a legacy
date as *ordering*, which is sound because the file is append-only, and not as
evidence of a day.

## What is in the artefacts column

Paths, as they were at the time. Several point into the private workspace
(`analysis/rtti/`, `analysis/symbols/`, `analysis/backup-shim/`) — that is
where a row's evidence physically lives, and saying so is more useful than
omitting it. Others name tool paths that have since moved: a row citing
`analysis/tools/grn_tagwalk.py` is a historical record, not a working path.
Paths are not rewritten to match a later layout; the log records what was true
when the work was done. Only the row *shape* was normalised, and only because
a record nothing can parse is not much of a record.
