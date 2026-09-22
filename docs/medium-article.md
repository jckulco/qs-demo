# A walkthrough of IBM QualityStage, from a messy CRM to a validated cross-system match

## Teaching a machine to spot "Juan Perez Gomez" and "JUAN PEREZ GOMEZ" are the same person

Every company eventually hits the same wall: a CRM full of customers typed in by different people, over different years, in different moods. "Juan Perez Gomez" three rows away from "JUAN PEREZ GOMEZ." "Ma. Fernanda Lopez R." sitting next to "Maria Fernanda Lopez Ruiz." Nobody notices until someone in Marketing asks a deceptively simple question: *"which of our CRM customers are already registered in the ERP?"* — and nobody can answer it with real confidence.

That question is exactly what **IBM QualityStage** — the data-quality engine bundled with DataStage — is designed to answer. Not with keyword search or spreadsheet formulas, but with genuine probabilistic record matching: the same family of techniques used in census deduplication and healthcare record linkage, pointed at a customer database instead.

I wanted to see the whole pipeline work end to end, on real (if deliberately messy) data, inside a real Cloud Pak for Data instance. So I built a small CRM — 15 customers, 5 duplicate pairs seeded on purpose, with the usual typos and abbreviations — plus an 8-row ERP file to cross-check against. Then I ran the full QualityStage toolkit against it, stage by stage, until I had a clean, validated answer.

## The pipeline: what each stage is actually for

QualityStage isn't a grab-bag of tools — it's a deliberate pipeline where each stage hands the next one exactly what it needs.

**Investigate** goes first, and it doesn't clean anything. It's a profiler: point it at a column and it tells you what patterns actually live inside it — is this name "First Last"? "Last, First"? Something messier? You need to know the shape of your data before you write a single rule to fix it.

**Standardize** comes next. This is where a raw name gets parsed into `PrimerNombre` and `Apellido`, where an address gets split into street type, number, and unit — using region-specific **rule sets** (I used Mexico's `MXNAME`, `MXADDR`, and `MXAREA`). Standardize doesn't just tidy data for human eyes; it produces special `Match*` columns purpose-built for the comparison engine downstream.

**Match Frequency** is the statistician of the group. Before any two records get compared, QualityStage needs to know how common each value is — "Garcia" is common, an unusual last name isn't, and a match on a rare value should carry more weight than a match on a common one. This stage computes that distribution, and every match stage after it leans on the result.

**One-Source Match** is the deduplicator: feed it one file, and it groups together records that are probably the same real person, scoring every comparison against the frequency statistics above. It designates one record per group as the "master" and routes the rest to their own output.

**Survive** decides, for each group of duplicates, which version of each field actually wins — the more complete name, the non-blank phone number — and produces a single clean record per group.

**Two-Source Match** is the finale: the same probabilistic engine, but pointed at two *different* datasets instead of deduplicating one. This is the stage that finally answers the original question — who exists in both systems, and who's exclusive to each.

## Setting up the test

The CRM had 15 rows, 5 of them deliberate duplicate pairs. The ERP had 8 rows — 6 of them true overlaps with the CRM, 2 of them ERP-only customers never entered into the CRM. I knew the "correct" answer going in; the point was to see whether QualityStage, run for real against real data, would land on it independently.

## Investigate and Standardize: giving the data a shape

Investigate profiled the raw name, address, and city/state/zip fields first, confirming what kind of mess I was actually dealing with before touching anything. Standardize then ran the Mexico-specific rule sets across the CRM, splitting full names into first/last components and addresses into their constituent parts, and producing the `Match*` columns — `MatchPrimerNombre_MXNAME`, `MatchApellido_MXNAME`, `MatchNombreCalle_MXADDR` — that the matching stages would later compare.

Alongside those rules, a small Transformer normalized phone numbers and zip codes into consistent, comparable formats. By the end of this stage, "Ma. Fernanda Lopez R." and "Maria Fernanda Lopez Ruiz" weren't just two strings anymore — they were two sets of parsed, comparable components, ready to be scored against each other.

## Match Frequency and One-Source Match: finding the duplicates

Match Frequency ran over the standardized columns, building the statistical backbone the matching engine needed. Then came One-Source Match: a match specification with `ZIP_STD` as a blocking key (so the engine only compares records that share a zip code, instead of every record against every other one) and four weighted comparisons — first name, last name, street name, and phone.

The result mapped exactly onto the 5 duplicate pairs seeded in the data: each pair collapsed into one designated "master" record, with the corresponding "copy" routed separately, and the 5 genuinely unique customers passing through untouched. Fifteen records in, ten distinct people identified.

## Survive: choosing the best version of the truth

For each pair of duplicates, Survive picked the winning value per field — favoring the record the matching engine had already flagged as the master, with fallback rules for completeness on names and phone numbers. The output: five clean survivor records, one per duplicate group, each one the best available version of that customer.

Combined with the five customers who never had a duplicate in the first place, that produced a single ten-row master file — the CRM, fully deduplicated, ready to be compared against the outside world.

## Standardizing the ERP: same rules, second dataset

Before any cross-system comparison could happen, the ERP file needed to go through the exact same Standardize treatment as the CRM — same rule sets, same parsed `Match*` columns, same normalized phone and zip formats. This is the part of QualityStage's design that makes the eventual comparison fair: you're never comparing a polished, standardized CRM name against a raw, untouched ERP string. Both sides get the identical preparation before the engine ever compares them.

## Two-Source Match: closing the loop

With both sources standardized and their respective match frequencies computed, the final stage brought them together: the deduplicated CRM as the reference source, the standardized ERP as the comparison source, matched on the same four weighted fields — first name, last name, street, phone — blocked on zip code exactly as before.

The result answered the original question directly:

```
Match (in both systems):        6 customers
Data-only (ERP exclusive):      2 customers
Reference-only (CRM exclusive): 4 customers
Clerical (needs manual review): 0
```

Six real people, correctly identified as existing in both the CRM and the ERP — not because their names matched character-for-character, but because the engine recognized "Carlos A. Sanchez" and "Carlos Alberto Sanchez" as describing the same person, with a calculated confidence weight backing up that decision. Two customers who only ever existed in the ERP. Four who only ever existed in the CRM. Nothing left for a human to review by hand.

## The bigger point

None of this works because QualityStage is smarter than a spreadsheet formula — it works because it treats matching as a *measurable* problem instead of a guess. Every decision in this pipeline — which records are probably duplicates, which field wins when two versions disagree, which customer in the ERP is probably the same person as a customer in the CRM — comes with a number attached. A weight. A probability. Something you can inspect, defend, and hand to an auditor instead of a shrug.

That's the difference between "our data team thinks these are the same customer" and "here's the confidence score, here's the blocking key, here's exactly why the engine made this call."

...

*If your CRM has ever made you wonder how many "customers" are actually the same person typed in twice, the lesson here isn't that QualityStage is magic — it's that record matching stops being guesswork the moment you give it real statistics to work with. Profile first, standardize before you compare, let the frequencies do the weighting, and trust the engine's confidence score over your own intuition. Do that consistently, and the question "who's actually in this database" turns from a shrug into a number you can stand behind.*

---

*Full setup, real intermediate outputs, and the imported platform project used for this walkthrough: [`qs-demo` on GitHub](https://github.com/<your-username>/qs-demo).*
