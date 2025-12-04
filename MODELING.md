# MEDS Ontology — Modeling considerations & design rationale

**Version:** 0.1.0  
**Author:** Alberto Marfoglia (ontology design)  
**Date:** 2025-12-04

## Abstract

This document provides a systematic, academically-oriented justification of the design choices embodied in `ontology/meds.ttl`. It follows the methodological recipe of Deborah L. McGuinness and Natalya F. Noy — *Ontology Development 101* — and maps each step to the MEDS conceptual schema. The aim is to make the ontology's structure, constraints, and operational responsibilities explicit so that practitioners can reason about correctness, maintenance, and validation.

---

## Table of contents

1. Determine the domain and scope  
2. Reuse of existing ontologies and identifiers  
3. Enumeration of important terms  
4. Class taxonomy and modeling strategy  
5. Properties: object vs datatype, domains and ranges  
6. Facets, cardinalities and constraints (OWL vs SHACL)  
7. Instance examples & use cases  
8. Mappings between MEDS schema elements and ontology artifacts (tabular)  
9. Design alternatives considered and rationale  
10. Enforcement strategy and limitations  
11. Recommended extensions and future work

---

## 1. Determine the domain and scope

**Domain:** Electronic Health Record (EHR) / claims-style *medical events*; specifically the MEDS conceptual model describing sequences of observations for subjects, and dataset metadata necessary for reproducible ML experiments.

**Intended users:**
- Data engineers and ETL authors converting raw EHR/claims sources into MEDS-compliant datasets.
- ML engineers and data scientists performing reproducible experiments.
- Ontology engineers and knowledge-graph practitioners integrating MEDS with other vocabularies (LOINC, SNOMED, OMOP).
- Tooling authors (validators, data catalogs, dataset profilers).

**Primary competency questions (CQs):**
1. For a given measurement, who is the subject and when did it occur?
2. Does the measurement carry a numeric or textual value (or additional modalities)?
3. What is the canonical code for this measurement and what human-readable description or parent codes exist?
4. Which subjects belong to the training split and which to held-out?
5. For a label sample, what is the prediction time and which label-type/value is present?
6. What provenance metadata (dataset name, MEDS version, ETL version) is associated with a dataset?
7. Which MEDS special codes (MEDS_BIRTH, MEDS_DEATH) are present?

These CQs drove the selection of classes, properties, and enforced constraints.

---

## 2. Consider reuse of existing ontologies

**Reused patterns and vocabularies:**
- `rdfs:label`, `rdfs:comment` for human-readable annotations.
- `dcterms:` (Dublin Core Terms) used to annotate provenance fields (e.g., `dcterms:created`, ontology metadata).
- `xsd:` datatypes for literal typing (`xsd:dateTime`, `xsd:string`, `xsd:double`, `xsd:boolean`, `xsd:integer`).

**Deliberate non-imports / minimal linking:**
- **SNOMED CT / LOINC / OMOP** were not imported directly due to licensing considerations and scope: MEDS is primarily a data-format standard. Instead, the ontology includes `:externalCodeId` (or encourages `skos:exactMatch`) so that `:Code` instances may link to external controlled vocabularies. This keeps the ontology license-flexible and permits downstream mapping without embedding third-party ontologies.

**Rationale:** reuse essential infrastructural vocabularies for interoperability; avoid heavy clinical ontologies inside the core to maintain broad usability.

---

## 3. Enumerate important terms

High-level terms distilled from MEDS:

- Subject (primary entity, subject_id)
- Measurement (event/observation)
- Code (vocabulary metadata for codes)
- DatasetMetadata (dataset-level provenance)
- SubjectSplit (train/tuning/held_out)
- LabelSample (prediction sample)
- Value modalities (numeric_value, text_value, image, waveform, etc.)
- Special codes (MEDS_BIRTH, MEDS_DEATH)

---

## 4. Classes and class hierarchy (taxonomy)

**Modeling strategy:** *middle-out*, with `Measurement` as central.

**Top-level classes:**
- `:Subject` — primary entity; declared with a functional key `:subjectId`.
- `:Measurement` — core per-row observation. Declared disjoint with `:LabelSample`.
- `:Code` — vocabulary entry; required to have `:codeString`.
- `:DatasetMetadata` — dataset provenance & conversion metadata.
- `:SubjectSplit` — enumeration-style class for dataset partitions.
- `:LabelSample` — supervised learning sample (subject × prediction_time × label).
- `:ValueModality` (abstract) with concrete subclasses (e.g., `:ImageValue`) to model extensions.

**Design note:** `StaticMeasurement` was considered but not made a strict subclass — staticity (absence of time) is modeled via optionality of the `:time` property rather than dedicated subclassing, to preserve data flexibility.

---

## 5. Class properties (slots)

**Object properties (selected):**
- `:hasSubject` (Measurement|LabelSample → Subject) — primary join.
- `:hasCode` (Measurement → Code) — link to code metadata.
- `:hasValueModality` (Measurement → ValueModality) — extensible modalities.
- `:assignedSplit` (Subject → SubjectSplit) — split assignment.
- `:describedInDataset` (Measurement|Code|Subject|LabelSample → DatasetMetadata) — provenance.

**Datatype properties (selected):**
- `:subjectId` (Subject → xsd:string) — functional key.
- `:time` (Measurement → xsd:dateTime) — optional (0..1).
- `:codeString` (Measurement|Code → xsd:string) — canonical code literal.
- `:numericValue` (Measurement → xsd:double) — optional numeric.
- `:textValue` (Measurement → xsd:string) — optional textual.
- `:predictionTime` (LabelSample → xsd:dateTime) — required.
- `:booleanValue`, `:integerValue`, `:floatValue`, `:categoricalValue` (LabelSample) — mutually exclusive by design (SHACL enforces exactly-one-of).
- Dataset metadata properties: `:datasetName`, `:medsVersion`, `:createdAt`, `:tableName`, etc.

**Domain/range design principle:** define domains and ranges to support automatic validation and documentation, but avoid over-constraining the open nature of MEDS data.

---

## 6. Facets of the properties (constraints)

**OWL-level constraints (expressed as restrictions):**
- `Measurement` has exactly one `:hasSubject` and exactly one `:hasCode` (the ontology encodes a cardinality of 1 but SHACL relaxes this to allow `codeString` alternative).
- `Code` must have exactly one `:codeString` (OWL `hasKey` is used).
- `LabelSample` must have exactly one `:hasSubject` and one `:predictionTime`; each label value property has `maxCardinality 1`.

**SHACL-level constraints (enforcement & pragmatic choices):**
- SHACL shapes enforce:
  - `Measurement` requires either `:hasCode` (link to `:Code`) **or** `:codeString` literal (reflects MEDS flexibility).
  - `LabelSample` must contain **exactly one** of `{booleanValue, integerValue, floatValue, categoricalValue}` via `sh:or`.
  - `DatasetMetadata` requires `:datasetName`, `:medsVersion`, and `:createdAt` (this is a stricter profile than the permissive MEDS JSON schema and can be relaxed).
  - `SubjectSplit.splitName` limited to enumerated values `train`, `tuning`, `held_out`.

**Why OWL + SHACL?**  
OWL expresses conceptual constraints and enables reasoning; SHACL provides practical, tractable validation for dataset-level conformance checks, including cardinalities and mutually-exclusive datatypes that are awkward in OWL. The design intentionally uses OWL for modeling and SHACL for operational validation.

---

## 7. Class instances (example uses)

See `examples/examples.ttl`. Instances demonstrate:
- `:Dataset_MIMIC_DEMO :datasetName "MIMIC-IV-Demo" ; :medsVersion "0.3.3" ; ...`
- `:subj12345678 a :Subject ; :subjectId "12345678" ; :assignedSplit :trainSplit .`
- `:meas1 a :Measurement ; :hasSubject :subj12345678 ; :hasCode :LAB_51237_UNK ; :time "2178-02-12T11:38:00"^^xsd:dateTime ; :numericValue 1.4 .`
- `:label_12345678_20210401 a :LabelSample ; :hasSubject :subj12345678 ; :predictionTime "2021-04-01T09:30:00"^^xsd:dateTime ; :booleanValue true .`

---

## 8. Direct mapping table: MEDS data model → ontology

The following table explicitly maps MEDS components (as specified in MEDS documentation) to the ontology elements in `ontology/meds.ttl`. Use this mapping as the authoritative bridge when ingesting MEDS data into RDF.

> **Legend:**  
> — MEDS = original MEDS concept / schema element (columns, files).  
> — Ontology = class / property or instance in `ontology/meds.ttl`.  
> — Enforcement = where the constraint is enforced (OWL, SHACL, or Procedural/ETL).

### 8.1 Core DataSchema → `:Measurement` and related

| MEDS element | Ontology mapping | Datatype / range | Required? (MEDS) | Enforcement |
|---|---:|---|---:|---|
| `subject_id` (DataSchema column) | `:Measurement :hasSubject → :Subject` ; `:Subject :subjectId` holds the literal | `xsd:string` for `:subjectId` | Yes | SHACL: `MeasurementShape` requires `:hasSubject`; SHACL: `SubjectShape` requires `:subjectId`; OWL: `:subjectId` is functional |
| `time` (DataSchema column) | `:Measurement :time` | `xsd:dateTime` | Yes (nullable only for static measurements) | SHACL: `time` `maxCount 1` and `datatype xsd:dateTime` |
| `code` (DataSchema column) | `:Measurement :hasCode → :Code` or fallback `:Measurement :codeString` (literal) | `xsd:string` | Yes | SHACL: `MeasurementShape` `sh:or` requires `hasCode` or `codeString`; OWL: `Code` `hasKey :codeString` |
| `numeric_value` | `:Measurement :numericValue` | `xsd:double` | Optional | SHACL: `numericValue` `maxCount 1` and `datatype xsd:double` |
| `text_value` | `:Measurement :textValue` | `xsd:string` | Optional | SHACL: `textValue` `maxCount 1` |

### 8.2 CodeMetadataSchema → `:Code`

| MEDS element | Ontology mapping | Datatype / range | Required? | Enforcement |
|---|---:|---|---:|---|
| codes.parquet rows (`code`) | `:Code` instances; `:codeString` holds value | `xsd:string` | `Code` table expected to contain all unique codes (MEDS guidance) | OWL: `:Code` `hasKey :codeString`; SHACL: `CodeShape` `minCount 1` |
| `description` | `:Code :codeDescription` | `xsd:string` | Optional | SHACL: `CodeShape` allows optional `codeDescription` |
| `parent_codes` | `:Code :parentCode → :Code` | IRI linking to other `:Code` | Optional | SHACL: `parentCode` nodeClass `:Code` |

### 8.3 DatasetMetadataSchema → `:DatasetMetadata`

| MEDS element | Ontology mapping | Datatype / range | Required? (ontology profile) | Enforcement |
|---|---:|---|---:|---|
| `dataset_name` | `:DatasetMetadata :datasetName` | `xsd:string` | Required (in current SHACL profile; MEDS JSON schema treats fields as optional) | SHACL: `DatasetMetadataShape` requires `datasetName`; OWL: `DatasetMetadata rdfs:subClassOf` cardinality 1 (current design) |
| `meds_version` | `:DatasetMetadata :medsVersion` | `xsd:string` | Required (profile) | SHACL: `minCount 1` |
| `created_at` | `:DatasetMetadata :createdAt` | `xsd:dateTime` | Required (profile) | SHACL: `minCount 1` |
| `table_names` | `:DatasetMetadata :tableName` (repeatable) | `xsd:string` | Optional / repeatable | SHACL: `tableName` `minCount 0` |
| `code_modifier_columns`, `raw_source_id_columns`, `site_id_columns`, `additional_value_modality_columns` | Represented as `:DatasetMetadata` datatype properties (e.g., `:siteIdColumns`) | `xsd:string` (or repeated triples) | Optional | Procedural: ETL and dataset documentation recommended |

### 8.4 SubjectSplitSchema → `:SubjectSplit`

| MEDS element | Ontology mapping | Datatype / range | Required? | Enforcement |
|---|---:|---|---:|---|
| `subject_id`, `split` | `:Subject :assignedSplit → :SubjectSplit` ; `:SubjectSplit :splitName` | `xsd:string` | Split assignment optional per subject | SHACL: `SubjectSplitShape` restricts `:splitName` values to `train`, `tuning`, `held_out` |

### 8.5 LabelSchema → `:LabelSample`

| MEDS element | Ontology mapping | Datatype / range | Required? | Enforcement |
|---|---:|---|---:|---|
| `subject_id` | `:LabelSample :hasSubject → :Subject` | IRI referencing `:Subject` | Required | SHACL: `LabelSampleShape` requires `hasSubject` |
| `prediction_time` | `:LabelSample :predictionTime` | `xsd:dateTime` | Required | SHACL: `minCount 1` |
| label value columns (boolean/integer/float/categorical) | `:LabelSample` datatype properties (`:booleanValue`, `:integerValue`, `:floatValue`, `:categoricalValue`) | variety of xsd types | Exactly one of these is expected per sample | SHACL: `LabelSampleShape` enforces exactly-one-of via `sh:or` |

---

## 9. Design alternatives considered and rationale

1. **Representation of label values**  
   - *Option A (chosen):* Four datatype properties on `LabelSample` with SHACL enforcing exactly-one-of.  
     *Rationale:* Simpler RDF serialization; aligns directly with MEDS `LabelSchema` columns. SHACL can validate mutual exclusivity.  
   - *Option B (alternative):* Introduce `LabelValue` individuals (object property `:hasLabelValue`) with disjoint typed subclasses (`BooleanLabelValue`, `FloatLabelValue`, etc.) and a single `:hasLabelValue` cardinality of 1.  
     *Rationale for alternative:* Better OWL reasoning support for “one-of” constraints and richer provenance/annotation on label values. More verbose in RDF. Use Option B if stronger reasoning is required.

2. **`code` as literal vs resource**  
   - *Option A (chosen):* Support both: `:Measurement :hasCode → :Code` *or* inline `:Measurement :codeString "..."` literal.  
     *Rationale:* MEDS allows datasets without a `codes.parquet`; this hybrid supports both workflows while enabling richer semantics when `:Code` resources exist.
   - *Option B (alternative):* Force `hasCode` to always reference a `:Code`.  
     *Rationale:* Simpler reasoning and consistent URI referencing but would penalize datasets lacking a separate code table.

3. **Dataset metadata strictness**  
   - *Option A (chosen in current artifacts):* Enforce `datasetName`, `medsVersion`, `createdAt` as required in SHACL.  
     *Rationale:* Practical for datasets intended for publication; ensures essential provenance.  
   - *Option B (looser alternative):* Keep all metadata optional and recommend best-practice documentation.  
     *Rationale:* More faithfully mirrors MEDS JSON schema where fields are optional.

4. **Where to enforce operational invariants**  
   - The ontology records **subject contiguity** and **temporal ordering** as annotations (procedural constraints). Enforcement is delegated to ETL and validator scripts. SHACL/OWL cannot reliably enforce file-shard level contiguity or sequence ordering across many triples.

---

## 10. Enforcement strategy and limitations

**OWL role:** conceptual modeling, unique keys, class hierarchy, disjointness, and machine-interpretable semantics. Useful for knowledge integration and inferencing (e.g., linking `Code` parent hierarchies).

**SHACL role:** practical validation of dataset instances (datatype correctness, cardinality, mutually-exclusive label presence, presence of required dataset metadata). The repository supplies `shacl/meds-shapes.ttl` for this purpose.

**Procedural role:** ETL and dataset-level checks for:
- Subject contiguity across physical shards.
- Temporal ordering within a subject.
- Global uniqueness of `subjectId` across very large datasets (can be implemented via SPARQL-based SHACL shapes, but may be more efficiently executed in data-processing systems).

**Limitations:**
- OWL cannot represent file-level or ordering constraints.
- SHACL can express many structural constraints but will be limited in scale for very large graphs; for very large datasets use streaming or database-backed validation.
- Exact-one-of for datatypes is implemented via SHACL `sh:or` rather than an OWL axiom — this is a deliberate trade-off for tractable validation.

---

## 11. Recommended extensions and next steps

1. **LabelValue object-pattern**: if you need stronger OWL reasoning, migrate to the `LabelValue` pattern (object property) and update SHACL accordingly. I can supply a migration patch.

2. **SKOS vocabulary for `SubjectSplit`**: represent splits as `skos:ConceptScheme` for cleaner vocabulary management.

3. **SPARQL-based SHACL constraints**: add SPARQL-based shapes to check uniqueness constraints (e.g., ensure `subjectId` is unique across `:Subject` individuals).

4. **Provenance / audit trail**: integrate `prov:` properties (W3C PROV) for ETL provenance (`prov:wasGeneratedBy`, `prov:wasDerivedFrom`) and link dataset validation runs.

5. **Integration examples**: provide mapping examples to OMOP/LOINC/SNOMED via `skos:exactMatch` or `rdfs:seeAlso` for canonical `:Code` instances.

6. **Validator toolchain**: produce a small Python CLI that runs SHACL, SPARQL uniqueness checks, and file-level contiguity/ordering checks on parquet files (I can provide a starter script).

---

## Appendix A — Quick reference: files in this repository

- `ontology/meds.ttl` — OWL ontology (Turtle)
- `shacl/meds-shapes.ttl` — SHACL shapes for validation
- `examples/examples.ttl` — example dataset instances
- `docs/` — (optional) generated Widoco docs for GitHub Pages
- `.github/workflows/validate.yml` — CI validation workflow
- `MODELING.md` — this document (detailed modeling rationale)

---

## Appendix B — Citation / provenance

Please cite the MEDS project and the original MEDS documentation when publishing datasets validated against this ontology. Suggested citation placeholder:

> Medical Event Data Standard (MEDS). GitHub: `https://github.com/Medical-Event-Data-Standard/med`. MEDS website: `https://medical-event-data-standard.github.io/`.

If you use this ontology in academic publications, include the ontology version and the repository URL (for reproducibility).

---

## Acknowledgements

This modeling exercise follows the methodological guidance of **Deborah L. McGuinness & Natalya F. Noy — "Ontology Development 101"**. Design and implementation choices reflect the MEDS project’s stated philosophy and the practicalities of dataset validation in a production environment.