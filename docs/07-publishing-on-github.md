# Publishing this project on GitHub

These steps are yours to run from your own machine/account (for security
reasons, this conversation can't create repos or handle your GitHub
credentials).

## 1. Create the repository

On GitHub: **New repository** → suggested name `qs-demo` (or
`datastage-qualitystage-demo` if you'd rather something more descriptive)
→ public visibility → don't initialize with a README (we already have
one).

## 2. Push the content

From the project's root folder:

```bash
cd qs-demo
git init
git add .
git commit -m "End-to-end QualityStage demo: Investigate, Standardize, Match Frequency, One/Two-Source Match, Survive — 30 real findings documented"
git branch -M main
git remote add origin https://github.com/<your-username>/qs-demo.git
git push -u origin main
```

## 3. Final repository structure

```
qs-demo/
├── README.md
├── LICENSE
├── data/
│   ├── customers_crm.csv
│   ├── customers_erp_master.csv
│   └── expected_output_notes.md          (validation checklist with the real numbers)
├── specs/
│   ├── investigate_spec.md
│   ├── standardize_rules.md
│   ├── match_specification_onesource.md
│   ├── match_specification_twosource.md  (theoretical plan — see docs/06c for the real config)
│   └── job2_architecture.md
├── docs/
│   ├── 00-step-by-step-guide.md            (master guide, start here)
│   ├── 01-investigate.md
│   ├── 02-standardize.md
│   ├── 03-match-frequency.md
│   ├── 04-one-source-match.md
│   ├── 05-survive.md
│   ├── 05b-master-merge.md
│   ├── 06-two-source-match.md            (original theoretical plan)
│   ├── 06a-standardize-erp.md
│   ├── 06b-match-frequency-twosource.md
│   ├── 06c-two-source-match-execution.md (the real execution, the longest doc)
│   ├── 08-troubleshooting.md             (30 real problems, each with a fix)
│   └── medium-article.md
├── outputs/                               (real output from every job, validated)
│   ├── investigate_reports/
│   ├── standardize_reports/
│   ├── onesource_match_reports/
│   ├── survive_reports/
│   ├── master_clean_reports/
│   ├── erp_standardize_reports/
│   ├── match_frequency_twosource_reports/
│   └── twosource_match_reports/
├── images/
│   └── pipeline-overview.svg              (summary diagram of the nine jobs)
└── platform-export/                       (real Cloud Pak for Data export —
                                             see platform-export/README.md)
```

## 4. About `/platform-export`

This folder holds the **real project export** from Cloud Pak for Data /
watsonx.data integration (`Manage > Export project`): all 9 real flows
(`JOB_01` through `JOB_09`), their metadata, the 2 manual Data
Definitions, and the connections to the data assets. It's not screenshots
or a recreation — it's the real project, exactly as it stood when the
final result was validated.

Anyone who clones this repo can import it directly into their own
instance (`Manage > Import project`) and have all 9 jobs already built,
ready to run against the CSVs in `/data`, instead of rebuilding them from
scratch following the step-by-step guide. See
`platform-export/README.md` for the detail on what it contains and how
to import it.

## 5. Link the Medium article

Once the article is published, add the link in the README (the "Related
article" section) pointing back to the repo, and link the repo from the
Medium article — so they reinforce each other for SEO and credibility.
