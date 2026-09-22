# Troubleshooting log — things that weren't in the official docs

This section documents real problems found while building this demo in
Cloud Pak for Data / watsonx.data integration, which weren't covered (or
not in enough detail) in IBM's public documentation. It's here as a quick
reference if you run into the same thing.

## 1. Error `CDIWA0201E: Missing column or columns for Investigate stage`

**Cause:** the Investigate stage gets saved with its investigation type
(Character or Word) configured, but **without having selected columns**
in the "Column investigation selection" section. That section doesn't
throw an error until you try to compile.

**Solution:** inside the stage, in the "Column investigation selection"
section, click **Edit** and **explicitly select** the columns to
investigate. Save and compile again.

## 2. The column selector for "Character investigation" doesn't respond to clicks

**Symptom:** when choosing Investigation type = **Character**, the
column table doesn't show checkboxes — only a pencil icon in the "Mask"
column. Clicking the row doesn't select anything; clicking the pencil
mistakenly triggers the character-mask editor (fills the field with
"T"s).

**Cause:** appears to be a rendering issue specific to Character mode
(possibly a UI bug on the tested version). **Word investigation**, by
contrast, shows normal checkboxes and works fine.

**Workaround that worked:** switch Investigation type to **Word**, select
the column with the checkbox (works correctly there), and apply. If you
specifically need Character (e.g. for short columns like ZIP or phone
where the character-by-character pattern matters, not word vocabulary),
try:
- Fully reloading the page (F5) before retrying.
- Maximizing the window / zooming out the browser (the modal can clip
  buttons if the window is narrow).
- Trying a different browser or a profile with no extensions.

## 3. The "Apply and return" button disappears or becomes unreachable

**Observed cause:** a "Cookie Preferences" banner from the platform can
overlap right on top of the button, or the modal ends up taller than the
visible viewport.

**Solution:** close/accept the cookie banner first; if it persists,
maximize the browser window or reduce the zoom.

## 4. The `MXAREA` rule set accepted 3 columns (`CITY`, `STATE`, `ZIP`) directly

**Not a bug, a useful finding:** the public documentation suggests that
"Area"-type rule sets (`USAREA`/`MXAREA`) expect a single combined
free-text column (e.g. `"Guadalajara, JAL 44100"`). In practice, the
stage allowed selecting `CITY` + `STATE` + `ZIP` as separate columns
directly, and concatenated them internally for the analysis (confirmed
because the output report shows the combined column as
`"CITY+STATE+ZIP"`).

**Practical implication:** there's no need to add a prior Transformer to
manually concatenate these columns before applying `MXAREA` — you can
select all three columns directly in the stage.

## 5. Regional rule sets for Mexico do exist

Unlike what was documented at the start of the project (which assumed
having to use generic `USNAME`/`USADDR` rule sets or create a custom
one), the tested instance does include a `Regions > Central America and
Caribbean > Mexico` folder with:

- `MXADDR` (addresses)
- `MXAREA` (city/state/zip)
- `MXNAME` (person names)
- `MXPREP` (supporting name prepositions)

No phone rule set exists (`MXPHONE` isn't available) — for that column a
simple Transformer with `Convert()` was used to keep digits only.

## 5. The "Individual/Organization" literal contaminates the last name in Standardize

**Symptom:** when configuring the **Standardize** stage (not Investigate)
with the `MXNAME` rule on `FULL_NAME`, and adding the literal
`ZQPINDZQ: Process All As Individual` (thinking it was required to tell
the rule set the records are people, not companies), the result came out
contaminated: `Apellido_MXNAME` showed values like `"GOMEZ ZQPINDZQ"`
instead of `"GOMEZ"`. Worse, the record with the dataset's most complex
name ("Ma. Fernanda Lopez R.") ended up completely unparsed
(`UnhandledData`).

**Cause:** the "Standardization columns" interface doesn't offer an
explicit way to flag the literal as a "control parameter" distinct from
a "data column" — both end up treated as positional arguments the engine
concatenates, and in this specific case `MXNAME` didn't interpret the
second argument as expected.

**Solution:** remove the literal entirely, leaving only the `FULL_NAME`
column in the configuration. The `MXNAME` rule set assumes "individual"
by default when the argument isn't specified — the literal was
unnecessary for this dataset (which only contains people, not
organizations). Once removed, the last name came out clean and the
problematic record parsed successfully (though with a minor limitation,
see point 9).

**When you'd actually need the literal:** only if your dataset mixes
people and companies and you need to tell the engine, record by record,
which is which — typically via a real column in the dataset, not a fixed
literal applied to every record.

## 6. `Str.PadLeft` isn't a valid function in DataStage's expression language

**Symptom:** writing a zero-padding expression in a Transformer using
`Str.PadLeft(Trim(ZIP), 5, '0')` (common .NET/Java syntax), the
expression editor flags an error and won't let you save.

**Cause:** DataStage's Transformer expression engine uses its own
language (similar to BASIC) — it has no native `Str.PadLeft`.

**Solution:** use concatenation (`:`) + `Right()`:
```
Right("00000" : Trim(ZIP), 5)
```
This concatenates 5 leading zeros and then takes the last 5 characters —
works equally well whether the original ZIP had 4 or 5 digits.

## 7. The data type can get lost at every link in the chain, not just at the source

**Symptom:** after fixing the output column type of `ZIP_STD` in the
Transformer (from `INTEGER` to `VARCHAR`) and confirming the real
`ZIP_STD.csv` file already carried the correct string, the ZIP still lost
its leading zero in the final CSV generated by Job 2.

**Real cause:** the **Join** stage (`Join_4`) that merges the branches in
Job 2 had its **own output schema** defined with `ZIP_STD` as `int32`,
regardless of the input column correctly arriving as
`string[variable_max=5]`. The job's log showed the exact warning:
```
WARNING IIS-DSEE-TFIP-00072 <APT_JoinSubOperatorNC in Join_4>
When binding output interface field "ZIP_STD" to field "ZIP_STD":
Implicit conversion from source type "string[variable_max=5]" to result
type "int32": Converting string to number.
```

**Solution:** check and fix the output column's data type **at every
stage it passes through** — don't assume fixing it at the source (or in
an intermediate Transformer) propagates automatically. In this case it
had to be fixed both in the Transformer and in the final Join's output
schema.

**Lesson for the whole demo:** any numeric-looking column with a text
meaning (postal codes, account numbers with leading zeros, product
codes) needs its type checked at **every point in the transformation
chain**, especially in Join-type stages, which often redefine their own
output schema when connected.

## 8. The "Inferred data view" in the data explorer isn't reliable for checking types

**Symptom:** after fixing `ZIP_STD`'s type to `VARCHAR` in Job 1's
Transformer, the platform's data viewer (preview button / "Inferred data
view") was used to confirm the result — and it still showed the ZIP
without the leading zero (`3100` instead of `03100`), triggering a false
alarm.

**Cause:** this viewer **re-infers each column's type from its content**
to display it on screen, regardless of the type actually stored in the
file. Since all the values looked numeric, it reinterpreted them as
numbers and stripped the zero when displaying them — but the real file
did have the correct string.

**Solution:** to check text vs. numeric types, **never trust an
"inferred" preview** (Excel does the same thing with digit-only columns,
too). Download the real CSV and open it in a plain-text editor, or
inspect it with a tool that doesn't reinterpret types.

## 9. Complex records can fail fully or partially in the name rule set, even after fixing the literal

**Symptom:** the record `"Ma. Fernanda Lopez R."` (prefix + initial +
suffix, all together) generated `Apellido_MXNAME = "LOPEZ R"` — the
`MXNAME` rule set couldn't separate the "R." suffix from the real last
name, though it did improve over the total failure it had when the
literal was contaminating the output (point 5).

**Not a configuration bug** — it's a real limitation of the `MXNAME` rule
set with uncommon or heavily compound name patterns. Worth documenting
as a genuine finding in the article: even IBM's predefined rules for
Mexico have limits with real, messy data, and Step 1's Investigate had
already flagged it (see the `?F?I` pattern in the `FULL_NAME` report).

## 10. A stage with the same name may need its schema refreshed manually when the source file is replaced

**Symptom:** after fixing and regenerating `ZIP_STD.csv` with the correct
type, uploading the new file to the project under the same name didn't
automatically update the schema the `ZIP_STDcsv_1` stage already had
saved in Job 2 — the type-conversion warning kept showing up on the
first run after the replacement.

**Practical solution:** after replacing a source file, go into the stage
that reads it, explicitly check its column schema (Output tab or
similar), and if it still shows the old type, fix it manually right
there instead of assuming it updated on its own.

## 11. The Match Specification is created from the project, not from inside the job

**Symptom:** inside the `One-source Match` stage, the "Browse" button to
select the Match specification showed "No assets found," and there was
no visible option in the job to create a new one.

**Cause:** Match Specifications are standalone project assets, not
internal job artifacts. They're created from **"New asset" → search
"datastage"** → **"Create reusable DataStage components"** → type "Match
specification" — a separate flow from building the job.

**Solution:** create the asset first (with its own designer, with
"Provision" and "Test" tabs), finish it, and **then** go back to the job
to select it via "Browse" — at that point it should already appear in
the list.

## 12. The Match Specification asset needs "Provision" (publishing) before it's available to jobs

**Symptom:** even after saving the entire Match Specification
configuration (columns, cutoffs) and seeing it listed under "All assets"
in the project, the stage's "Browse" picker kept showing "No assets
found."

**Cause:** saving the design isn't the same as publishing it. The Match
Specification designer's **"Provision"** tab has an action that
compiles/publishes the specification as an executable artifact — without
that step, the asset exists as a "design" but not as something a
DataStage stage can consume.

**Solution:** on the Match Specification asset, go to the "Provision" tab
and run the publish action. Expected confirmation in the status bar:
`"Publish successful"`. After that, the job's stage does recognize the
asset in its picker.

## 13. The "Overflow value" field can't be left blank when saving a match pass

**Symptom:** after manually adjusting the Match cutoff, the "Save" button
on the pass configuration panel stayed disabled with no visible error
message.

**Cause:** the "Overflow value" field (under "Cutoff values," next to the
"Match" field) can get accidentally emptied while editing nearby values,
and an empty numeric field blocks saving without showing an explicit
on-screen validation message.

**Solution:** check that "Overflow value" has a numeric value (`10000` is
a reasonable default) before trying to save.

## 14. The "Duplicate" port on the One-Source Match stage is required to avoid losing records

**Symptom:** running the One-Source Match stage with only the "Match,"
"Clerical," and "Nonmatched" ports checked, the total output record
count (5+0+5=10) didn't match the input total (15), with no error or
warning to flag it.

**Cause:** when the engine confirms a duplicate group, it designates
**one "master" record** (which goes out the `Match` port) and sends the
**other copies of the same group** to the **`Duplicate`** port — a
separate port from `Match`. If that port isn't checked or connected to
an output file, those records get silently dropped.

**Solution:** if the goal is to account for 100% of the source records
(not just identify which are duplicates of what), **always** check the
"Duplicate" checkbox under "Match outputs" and connect it to its own
output file.

## 15. The leading-zero ZIP problem showed up a fourth time, in a new context

**Context:** after fixing it in Job 1's Transformer and in Job 2's
`Join_4` (see points 6-7), the same string-to-`int32` implicit conversion
problem showed up again in the One-Source Match stage's (Job 4) output
`Sequential file`, confirmed by the same type of log warning:
```
Implicit conversion from source type "string[variable_max=5]" to result
type "int32": Converting string to number.
```

**Reinforced lesson:** this isn't a one-off bug in one stage — it's a
pattern that repeats at **any new point** in the chain where a
numeric-looking text column (postal codes, in this case) connects to a
new stage or connector. The fix is always the same (explicitly correct
the type to `VARCHAR` at every link), but you have to stay alert that
**it can reappear in any new job** that touches that column, not just
the ones already fixed before.

## 16. Survive's comparative techniques (Equals/Not equals/Greater than/Less than) do NOT compare between records in the group

**Context:** in `JOB_05_Survive` (Step 5), the `Less than` technique was
tried on `CUST_ID` to make the record with the lowest ID in the group
survive (the "Minimum" equivalent).

**Symptom:** on save, with the `Data` column empty, this error appeared:
```
Rule 2: Numeric columns can only be compared to numeric data.
```

**Cause:** `Equals`, `Not equals`, `Greater than`, and `Less than` in
Survive's rule editor **compare each record in the group against a fixed
value** typed into the `Data` column — there's no technique that
compares the group's records against each other to pick the minimum or
maximum of a numeric column.

**Real solution:** instead of forcing a numeric comparison on `CUST_ID`,
it helped that One-Source Match already flags the master record in the
`qsMatchType` column (`"MP"` vs `"DA"`). The winning rule was
`AllColumns` = `qsMatchType Equals "MP"`, with `Data = "MP - Master
record"` — so the master donates all its columns (including `CUST_ID`)
with no need to compare values.

## 17. UI bug in Survive: the side summary panel can show incorrect values that DON'T affect the real result

**Symptom:** after configuring 3 rules in the "Define survive rule
columns" modal (one with `Data = "MP - Master record"`, two with no
`Data` value), the stage's side summary panel (outside the modal) showed
the text `"MP"` repeated in the `Data` column of all **three** rows,
including the two that correctly should have had no value at all.

**Investigation:** reopening the full edit modal, all 3 rows showed the
correct values (`Equals`/`"MP - Master record"`, `At least one`/empty,
`Most frequent`/empty). The job was run anyway and the real output CSV
was validated: the `PHONE_STD` and `ZIP_STD` values were real
phone/postal-code data, not the text `"MP"`.

**Conclusion:** the side summary panel has a cosmetic rendering bug
(probably repeating the last text value typed into any `Data` field in
the form). **It doesn't affect what actually gets saved or executed.**
Lesson: when facing a visual discrepancy like this, trust the full edit
modal and, above all, **validate against the real execution result**
before assuming it's a functional bug.

## 18. Clearing a Survive rule's "Data" field can leave a residue that blocks saving

**Symptom:** after typing and then deleting a value in the `Data` field
of a rule using the `At least one` or `Most frequent` technique (which
don't require `Data`), saving kept failing with:
```
Rule 2: This technique does not use data column. Please remove the data value.
```
...even with the field visually empty.

**Apparent cause:** a leftover value (`NaN`) stays in the component's
internal state even though the interface no longer shows text.

**Solution:** delete the entire row (`⋮` menu → `Delete`) and create a
new row from scratch, never touching the `Data` field on techniques that
don't need it.

## 19. Misaligned reader schema after regenerating an already-used source CSV (JOB_06_MasterMerge)

**Context:** while building `JOB_06_MasterMerge` on top of
`customers_crm_survived.csv` (already regenerated with the point-15 ZIP
fix), the reader stage kept the schema from an earlier version of the
file.

**Symptom:** fatal error when running the job:
```
CDICO9999E: The data in row 0 for the qsMatchDataID column is invalid:
For input string: "1.10000000000000000E+01"
```
The scientific-notation value (`"1.1E+01"`) actually belongs to
`qsMatchWeight`, not `qsMatchDataID` — the reader was reading columns
out of alignment.

**Solution:** force a schema re-detection from the current file in the
reader stage. Same root cause as point 12 on this list, but it confirms
the issue **reappears every time a source file gets regenerated**, not
just the first time it was documented.

## 20. Implicit-conversion warnings reappear on new columns every time a new Funnel/Copy is added

**Context:** in `JOB_06_MasterMerge`, `Funnel_1` showed:
```
Exterior_MXADDR: string → int32
PHONE_STD: string → int64
```
Same pattern as points 15/18, now on two more columns. Unlike other
times, in this run **it didn't end up corrupting the real data** in the
final CSV — but the type was preemptively fixed to `VARCHAR` anyway, to
not rely on luck.

**Project-wide consolidated lesson:** the string-to-numeric implicit
conversion loss/corruption pattern has now shown up **6 times** across
different columns (`ZIP_STD` x4, `Exterior_MXADDR`, `PHONE_STD` x2). Any
numeric-looking text column (postal codes, house/exterior numbers, phone
numbers) needs to be checked and explicitly re-typed to `VARCHAR` on
**every new stage** that touches it — no exceptions, even if a
particular run "didn't break."

## 21. A "Defaulting column in transfer" warning on a Funnel can be harmless metadata noise

**Symptom:** in one `JOB_06_MasterMerge` run, 6 warnings appeared:
```
Funnel_1: Defaulting "qsMatchWeight" in transfer from "inRec" to "outRec".
(+ 5 more, one for each diagnostic column exclusive to one branch)
```

**Investigation:** it was suspected the column selection excluding those
6 diagnostic columns had accidentally reverted. The real output CSV was
validated directly: it still had the correct number of columns (99, not
105), with no trace of the diagnostic columns.

**Conclusion:** the warning happens during the *"checking operator"*
phase (metadata validation before execution) when the engine detects
that one branch's declared schema has columns the other doesn't, and
warns it's going to "fill them in" — but if the real output's column
selection already excludes them, they never make it into the final
file. It's internal validation noise, not a functional problem.
**Lesson reinforced for the third time in the project:** with any new
warning in the log, always validate against the real output file before
assuming something needs fixing (see also points 13 and 17 on UI/log
discrepancies that turned out to be harmless).

## 22. A Transformer output column with the same name as its input causes a duplicate copy to be dropped at the next Join (JOB_07_StandardizeERP)

**Context:** while building `JOB_07_StandardizeERP`, the Transformers
generating `ZIP_STD` and `TELEPHONE_STD` were initially left with the
output column name matching the input one (`POSTAL_CODE`, `TELEPHONE`).

**Symptom:**
```
Join_4: Dropping component "POSTAL_CODE" because of a prior component with the same name.
```
And the final CSV had no `ZIP_STD` or `TELEPHONE_STD` column at all —
only the original columns, with the value already cleaned but not under
the expected `_STD` name.

**Cause:** without renaming the Transformer's output column, the engine
overwrites the value in the original column instead of creating a new
one. When it reaches the Join that merges that branch with the others
(which also carry their own copy of the original column, inherited from
the earlier Copy), a name collision happens and the engine silently
drops the duplicate copy.

**Solution:** explicitly rename the Transformer's output column to the
expected `_STD` name, instead of leaving the default name matching the
input.

## 23. Editing a Transformer can silently disconnect a link on another branch of the same Copy

**Context:** while fixing `Transformer_3`'s (ZIP) output name in
`JOB_07_StandardizeERP`, `Transformer_2`'s (phone, on another branch of
the same `Copy_1`) output link got disconnected from the Join that
integrated it into the main flow.

**Symptom:** the `TELEPHONE`/`TELEPHONE_STD` column disappeared entirely
from the output CSV, with no warning to flag it.

**Solution:** after editing any stage that shares an upstream `Copy` with
other branches, visually check that every link is still connected on the
canvas before running the job — a stage's editor doesn't warn you if
saving it broke a connection on another branch.

## 24. An output column named "_STD" can be a disguised pass-through, with no real transformation applied

**Context:** after resolving points 22 and 23, `TELEPHONE_STD` already
showed up in `JOB_07_StandardizeERP`'s CSV, but for the only two ERP
records with dashes in the original phone number (`"442-123-4567"`,
`"999-222-3344"`), the `TELEPHONE_STD` value still had the dashes
uncleaned. The other 6 records proved nothing because they never had
dashes to clean in the first place.

**Cause:** in the Transformer's "Output → Column mapping" tab, the
`TELEPHONE_STD` column had, as its **Derivation**, the reference
`TELEPHONE_STD.TELEPHONE` — DataStage's standard
`link_name.column_name` notation for passing through an input column
**with no function wrapping it at all**. It's a pure pass-through,
disguised by the output column's `_STD` name, which (incorrectly)
suggested cleanup logic had already been applied.

**Solution:** edit the derivation in the expression editor (pencil icon)
and wrap the reference with the real function, e.g.
`Convert("-., #", "", TELEPHONE_STD.TELEPHONE)`.

**Consolidated lesson:** the name of an output column (even one
literally saying `_STD`) is never proof it has a transformation applied
— you always have to open the expression editor and confirm the
expected function is actually present. And records with real edge cases
in the data (here, the ones with dashes) are the only reliable proof
that a cleanup actually works; records that already came in "clean"
prove nothing, even if at a glance they seem to validate the result.

## 25 to 30. JOB_09_TwoSourceMatch — the most complex step in the project

Step 6 (Two-Source Match) chained 6 distinct problems before running
clean, each one revealing a real stage constraint that wasn't
anticipated in the original theoretical plan. Documented in detail in
[`docs/06c-two-source-match-execution.md`](06c-two-source-match-execution.md)
— summary:

- **#25**: the `Two-Source Match` stage requires **4 input links**
  (`Data`, `Reference`, `DataFreq`, `RefFreq`), not 2 like One-Source
  Match.
- **#26**: the `Funnel` stage **doesn't union mismatched schemas** — it
  only concatenates streams with the exact same schema. Combining two
  sources with different column names in a Funnel produces erratic
  results depending on the order the links are connected in. Solution:
  generate independent frequency files per source, instead of forcing a
  combination.
- **#27**: the `qsMatchType`/`qsMatchDataID` columns inherited from an
  earlier matching process (One-Source Match) clash with the internal
  columns the Two-Source Match stage itself generates — they need to be
  dropped before reaching the stage.
- **#28**: the Match Specification's schema import wizard **re-infers
  types from the CSV's content**, ignoring the real declared schema —
  same pattern as point 13, now directly affecting the Match Spec.
  Solution: create a manual Data Definition with the correct types
  explicit, instead of importing directly from the file.
- **#29**: renaming a column in a Transformer without propagating the
  change to the other 3 related places (Data Definition, Match
  Specification, frequency file) produces a chain of distinct errors,
  one for each place not updated.
- **#30**: a theoretical plan's cutoffs are only a starting point — they
  need to be validated against the real weights (`qsMatchWeight`) the
  engine produces with the real data, and adjusted based on the observed
  result. The UI's "Duplicate cutoff" field has no functional effect in
  "one-to-one" mode, but requires `Match ≤ Duplicate` to save — you have
  to raise `Duplicate` before lowering `Match`, in that order.

