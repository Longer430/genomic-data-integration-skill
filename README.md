# Genomic Data Integration

An agent skill (Claude / Codex `SKILL.md` format) for **combining genomic, EHR/phenotype and imaging data across cohorts, biobanks and hospitals**, and for building cohort-discovery or LLM query tools on top of that data.

It walks an agent through:

- **Governance first:** what All of Us, UK Biobank, dbGaP, hospital EHR, pediatric consent and cross-border rules (GDPR, China, the US DOJ bulk-data rule) allow, and why that usually means a federated design
- **Genomic harmonization:** genome build, allele and strand alignment, platform and imputation differences, variant classification, germline vs somatic, and gene-symbol aliases
- **People and samples:** overlap across sources, relatives from trio sequencing, one patient with many rows
- **Phenotypes and time:** EHR phenotypes as noisy labels, per-site validation, symptom onset vs diagnosis date, referral-center lookback, which date to use
- **Analysis design:** per-source analysis plus meta-analysis, heterogeneity, replication cohorts
- **Output privacy:** small-cell and complementary suppression, differencing attacks
- **AI-generated queries:** enforcing privacy at the data layer, a pytest-style validator for LLM-written SQL, and how to investigate a count that changed after an update
- **Templates:** a cohort-query skeleton, nested cohorts, and short emails to investigators

The rule it keeps coming back to: whether you *can* merge is decided by governance, whether the merge is *right* is decided by definitions, and whether anyone can *trust* the result is decided by reproducibility.

Companion skills: [medical-report-label-refinement](https://github.com/Longer430/medical-report-label-refinement-skill) (labels from report text) and [mri-study-pipeline](https://github.com/Longer430/mri-study-pipeline-skill) (the imaging side).

## Origin

Written while preparing scenario-based design exercises on multi-modal research informatics (multi-source cohort integration, pediatric rare-disease networks, LLM-assisted cohort queries). It draws on published practice: the GIANT consortium's GWAS meta-analysis QC protocol, eMERGE work on phenotype portability, HGNC symbol guidance, NIH and DOJ policy notices, and GA4GH standards. It contains no patient data.

## Repository layout

```
SKILL.md     the skill (what the agent reads)
LICENSE      MIT
```

## Install

**Claude Code / Claude:** put the folder containing `SKILL.md` at `~/.claude/skills/genomic-data-integration/`, or upload it as a skill.

**Codex:** put the folder at `~/.codex/skills/genomic-data-integration/`, restart the session, and invoke it with `$genomic-data-integration`.

## Scope and limits

- This is guidance for research design and data engineering. It is not legal advice, and it does not replace your IRB, privacy office or data access committee.
- The regulatory section reflects the situation as of November 2026, and these rules change often. The skill tells the agent to check current status before stating any rule.
- Thresholds such as small-cell cutoffs vary by institution. Use your own policy.

## License

Copyright (c) 2026 Wenting Long ([@Longer430](https://github.com/Longer430)).

Released under the [MIT License](LICENSE). You may use, modify and redistribute this skill, including commercially, as long as the copyright notice and license text are kept.
