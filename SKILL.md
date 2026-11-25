---
name: "genomic-data-integration"
description: "Use when merging genomic, EHR and imaging data across cohorts or sites, writing cohort queries or validating AI-generated ones, or advising investigators on governance and pitfalls."
---

# Genomic Data Integration

**When to use:** combining genomic variant data with clinical/EHR phenotypes (and often imaging or radiology reports) across several cohorts, biobanks or hospitals; designing a cohort-discovery portal or an LLM query interface over such data; writing cohort SQL; explaining a changed count to an investigator; or answering interview / design-exercise questions on these topics (e.g. the Boston Children's Hospital Research Informatics exercise).

**Related skills:** `medical-report-label-refinement` for turning report text into labels (phenotypes from notes or radiology reports follow the same rules: describe, don't diagnose; keep "not mentioned" separate from "negative"; validate against a gold subset). `mri-study-pipeline` for imaging metadata (series typing from headers, never from descriptions).

**Report to the user in plain language (Chinese for this user).** When drafting emails or answers for investigators, write short, plain English that sounds like a colleague, not a template: no bold section headers unless needed, no perfectly parallel triads, no em dashes.

**The one-line lesson:** whether you *can* merge is decided by governance; whether the merge is *right* is decided by definitions; whether anyone can *trust* the result is decided by reproducibility. Technology choices follow from those three.

## 0. Questions to ask first

1. The scientific question and analysis type (GWAS, PRS, association, prediction, cohort counts). Are summary statistics enough, or is individual-level joint modelling required? This decides the architecture.
2. Approval status per source: platform tier, application ID and scope, dbGaP accession and consent groups, IRB protocol, DUAs; for pediatric data, whether consent covers genomic research.
3. Genomic data per source: WGS / WES / array, genome build, imputation panel, raw or QC'd, variant classification source.
4. Phenotype definitions per source (ICD codes, surveys, problem list, chart review) and whether a validated subset exists.
5. Population: ancestry, age range (pediatric vs adult), ascertainment.
6. Possible overlap: shared patients between hospitals, relatives, the same participants in several studies.
7. Outputs: publication, sharing obligations, return of results, timeline.

## 1. Governance decides the architecture

Default conclusion: individual-level data mostly cannot be pooled, so design **federated**: the same containerized pipeline runs inside each environment and only approved summary results leave.

| Source / setting | Typical constraint | Design consequence |
|---|---|---|
| All of Us | Analysis only in the Researcher Workbench; genomic data needs Controlled Tier; no individual-level download; small-count dissemination rules | Compute in the platform; export summary stats that pass its rules |
| UK Biobank | Analysis in UKB-RAP; use limited to the approved application scope; no sharing with non-applicants | Same; check scope covers the question |
| dbGaP | DAR approved by a DAC; data-use limits per consent group (GRU, HMB, DS-xxx, NPU, IRB-required); NIH security requirements for the environment | Check each consent group; some may have to be excluded; NIH-compliant enclave |
| Hospital EHR (local, BCH) | HIPAA, IRB, DUA; minimum necessary; de-identified or limited data set | Read-only de-identified views; row-level data only with IRB approval |
| Pediatric | Parental permission + child assent (Common Rule Subpart D); re-consent at 18; genomic consent scope; relatives in the data | Track consent per participant (GA4GH DUO codes); honour opt-outs |
| EU sites | GDPR: genetic and health data are special-category; pseudonymized data is still personal data; DPIA; transfer via EU-US Data Privacy Framework or SCCs | Keep SCCs ready; do not depend on one transfer mechanism |
| China and other countries of concern | China human genetic resources export approval; US DOJ bulk sensitive data rule (28 CFR 202, effective 2025-04-08; genomic data of >100 US persons; applies even if anonymized/encrypted); NIH barred institutions in countries of concern from controlled-access repositories from 2025-04-04 (NOT-OD-25-083) | Federated node only; no access to US genomic data from those sites |

Regulatory status above is as of Nov 2026 and moves fast (e.g. the DPF was upheld by the EU General Court in Sept 2025 and is under appeal). **Search before stating any rule as current.**

Make every source a removable module, so a change in one approval does not invalidate the whole study.

## 2. Genomic harmonization (silent errors live here)

The most dangerous errors do not raise exceptions. Analogy for interviews: in MRI work, wrong slice order capped a model at 0.65 AUC with no error message; allele flips do the same to a meta-analysis.

- **Genome build:** lift everything to one build (GRCh38); record the original build.
- **Allele and strand alignment:** align effect/other alleles; palindromic SNPs (A/T, C/G) are ambiguous; compare allele frequencies to a reference (gnomAD) to catch flips.
- **Platform differences:** WGS vs WES vs array, different imputation panels; analyse shared, well-imputed sites; one QC standard.
- **Variant classification:** P/LP vs VUS vs conflicting ClinVar; reclassification over time; record classification source and date; decide current vs as-of-test classification explicitly.
- **Origin and setting:** germline vs somatic; clinical (CLIA) vs research results.
- **Gene symbols:** map through the HGNC alias table. Aliases and previous symbols are common (NRSF = REST, CoREST = RCOR1; CECR1 became ADA2). Ask whether a family name ("CoREST") means one gene or the family (RCOR1-3).
- **Sex and identity checks:** genotype-inferred sex vs recorded sex; duplicate genotypes within one enclave.

Reference: Winkler et al. 2014, Nature Protocols, "Quality control and conduct of genome-wide association meta-analyses" (GIANT; study-level, meta-level and output-level QC).

## 3. People and samples

- **Overlap across sources** (two hospitals, several dbGaP studies): privacy-preserving record linkage (hashed tokens) or statistical estimates of overlap (e.g. LD score regression cross-trait intercept). Individual-level genotype dedup only inside one enclave.
- **Relatives:** trio/family sequencing puts parents and siblings in the variant table; filter to probands; account for relatedness in analysis.
- **One patient, many rows:** several tests, diagnoses, scans, reanalyses. Deduplicate to one row per patient; count DISTINCT patients; say when a patient appears in several output rows (two genes, two diagnoses).
- **Identity quirks:** merged MRNs; fetal MRI filed under the mother; twins.

## 4. Phenotypes and time

An EHR phenotype is a label extracted from messy records; treat it like report-derived labels.

- **No code is not a control.** Controls need exclusion rules (e.g. PheWAS exclusion ranges).
- **Validate per site:** phenotype algorithms do not port cleanly across EHRs and documentation styles; eMERGE validated by chart review at each site (target PPV ~95%). Report PPV per site.
- **Coding habits differ by site:** compare prevalence per source first; large spreads on subjective conditions are habits, not biology.
- **Noise rules:** require 2+ encounters; exclude rule-out codes; explicit code lists including ICD-9 legacy codes; exclude look-alikes (febrile seizures are not epilepsy).
- **Onset is not the diagnosis date.** ASD is often diagnosed years after first concerns; epilepsy after the first seizure. Define onset as the earliest signal (symptom codes, related meds, EEG, first diagnosis) and rerun with diagnosis date as a sensitivity check.
- **Referral centers:** onset may predate the first visit (outside care). Require a lookback window and report how many it excludes.
- **Which date:** DICOM acquisition date, not PACS import or report date; order vs result date for genetic tests; final reports and addenda; half-open ranges [start, end).
- **Date shifting:** in de-identified data, one consistent shift per patient across all tables.
- **Pediatric vs adult cohorts:** phenotypes and age at measurement are often not comparable; do not force them together.

## 5. Heterogeneity and analysis design

- Analyse per source, then meta-analyse (e.g. METAL); test heterogeneity (I², tau²); leave-one-source-out; treat source as covariate or random effect.
- "The hospital is a variable too": random folds across sites inflated CV by 0.05-0.07 in multi-site imaging work; the same applies to pooled cohorts.
- Pre-register the analysis plan (definitions, QC, covariates); hold out one cohort for replication; do not run many variants and pick the best.
- If a pooled model is trained, cohort membership is a shortcut; do not feed site-identifying metadata.

## 6. Privacy of outputs

- Aggregate counts by default; suppress small cells under the institutional threshold (often <11); complementary suppression so totals cannot reveal hidden cells; no exact totals when cells are suppressed.
- Rare diseases and rare genes: a handful of records can identify a family; expect most cells to be small.
- Differencing attacks: log every query, rate-limit, flag overlapping queries.
- Counts can be preparatory-to-research; patient lists, row-level exports or re-contact go through the IRB / honest broker.
- Never send report text or patient data to an external AI service; run NLP in-house.

## 7. AI-generated queries and reproducibility

**Enforce safety in the database, not in the prompt.** The LLM sees only the schema; queries run under a read-only role on de-identified views that can only return aggregates; suppression happens at the data layer.

**Validator before display (pytest-style):**

```python
def test_sql_is_safe(sql, params):
    assert is_read_only(sql)                       # no writes
    assert tables_used(sql) <= APPROVED_VIEWS      # de-identified views only
    assert dates_are_bound_params(sql)             # no pasted strings
    assert params.start < params.end

def test_logic(c1, c2, links, params):
    assert c1.patient_id.is_unique and c2.patient_id.is_unique   # no duplicate patients
    assert set(c2.patient_id) <= set(c1.patient_id)              # nesting
    assert dates_within(c1, params.r1) and dates_within(c2, params.r2)
    assert (links.event_date > links.index_date).all()           # order of events
    assert links.index_test_id.isin(c1_tests.test_id).all()      # linked to the qualifying test

def test_known_fixture(query):
    # synthetic patients with known answers: two tests, a sequenced parent,
    # a research test, an amended report, an event outside the window
    assert query.run(FIXTURE_DB) == EXPECTED

def test_output(output):
    assert not output.has_patient_level_columns()
    assert all_small_cells_suppressed(output)
    flag_if_empty(output)                           # do not show an empty result as a true zero
```

Top failure modes it catches: duplicate patients from joins; broken nesting; date errors (wrong field, bounds, order); wrong population (relatives, research tests, wrong test type, amended reports counted as events, diagnosis instead of indication); unsafe output (writes, raw tables, row-level data, small cells).

**When a count changes (e.g. 42 -> 29 after an update):**
1. Tell the investigator not to use either number yet; own it if it followed a release.
2. Pull both runs from the query log (SQL, semantic-layer and prompt versions, data snapshot).
3. Rerun both SQL versions on the same snapshot: logic change vs data change.
4. Diff the patient sets internally; find the filter each missing patient fails.
5. Usual causes: a definition changed (LP included or not, onset dating), data changed (reclassification, new earlier records), the LLM wrote different SQL, the first count was inflated (relatives, duplicates).
6. Deliver: definitions confirmed with the investigator, a reviewed and locked query, a dated snapshot, a one-page methods note with counts after each filter (attrition table) so it can be cited and rerun.
7. Prevent: regression tests on saved queries before every release; version-pin saved queries.

## 8. Cohort query skeleton

Build in this CTE order, one row per patient at each step, aggregate only at the end:

```sql
WITH genes    AS (SELECT hgnc_symbol FROM hgnc_aliases WHERE alias IN (...)),
carriers AS (SELECT DISTINCT s.patient_id, v.gene FROM variants v JOIN samples s USING (sample_id)
             WHERE v.gene IN (SELECT hgnc_symbol FROM genes)
               AND v.classification IN ('Pathogenic','Likely pathogenic')
               AND v.origin = 'germline' AND s.relationship = 'proband'),
dx       AS (SELECT patient_id, dx_group, MIN(dx_date) dx_date FROM mapped_dx
             WHERE dx_type <> 'rule_out' GROUP BY 1,2 HAVING COUNT(DISTINCT encounter_id) >= 2),
onset    AS (/* earliest symptom signal: symptom codes, meds, EEG, first dx */),
imaging  AS (/* by procedure code; acquisition_date < onset; age < 18; lookback; one status per patient */)
SELECT gene, dx_group, imaging_status,
       CASE WHEN COUNT(DISTINCT patient_id) < 11 THEN '<11' ELSE COUNT(DISTINCT patient_id)::text END
FROM carriers JOIN dx USING (patient_id) JOIN imaging USING (patient_id)
GROUP BY 1,2,3;
```

Nested cohorts: build cohort 1 as all qualifying index events; build cohort 2 by joining the follow-up event to cohort 1's qualifying event IDs (not to the patient), so the subset holds by construction; then dedupe per patient.

## 9. Writing to investigators

- Lead with the conclusion in one sentence ("we can't pool this, but a federated design still answers your question").
- Frame governance as the reason for the design, not as "you can't".
- Questions as a short numbered list; plan in 3-4 lines; constraints in 3-4 lines; a concrete next step (30-minute call, pilot).
- Plain words over jargon; 150-300 words; offer a bilingual version (English + Chinese) when the user asks.

## 10. How common each issue is (for interviews)

- **Textbook (mention briefly):** build/allele/strand QC, relatedness and stratification, EHR phenotype portability and per-site PPV, site heterogeneity, gene aliases, sample overlap, data cannot leave platforms, small-cell suppression.
- **Common but often missed (best differentiators):** onset vs diagnosis date, referral-center lookback, PACS import date, trio relatives, amended report vs reanalysis, consistent date shifting, differencing attacks, nesting that is not really nested.
- **New since 2025 (shows currency):** DOJ bulk data rule and NIH country-of-concern restrictions; validating LLM-generated SQL, result drift after semantic-layer updates, enforcing privacy at the data layer.

## 11. References

- Winkler TW et al. Quality control and conduct of genome-wide association meta-analyses. Nat Protoc 2014;9:1192-1212.
- eMERGE / PheKB phenotype algorithm portability and validation papers (Kirby et al. JAMIA 2016; Kho et al. Sci Transl Med 2011).
- HGNC and HGNChelper (alias and outdated symbol correction).
- NIH NOT-OD-25-083 (controlled-access data, countries of concern); 28 CFR Part 202 (DOJ data security program).
- GA4GH: Data Use Ontology (DUO), Phenopackets; HPO; Mondo; OMOP CDM.

---

Copyright (c) 2026 Wenting Long ([@Longer430](https://github.com/Longer430)). Released under the MIT License.
