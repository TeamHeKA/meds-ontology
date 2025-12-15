# MEDS Ontology — Modeling considerations & design rationale

**Author:** Alberto Marfoglia (ontology design)
**Date:** 2025-12-04

## Abstract

This document provides a systematic justification of the design choices embodied in `ontology/meds.ttl`. It follows the "Ontology Development 101" methodology and maps MEDS conceptual schema elements to ontology artifacts. The goal is to make the ontology’s structure, constraints, and operational responsibilities explicit so practitioners can reason about correctness, maintenance, and validation.

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
11. Recommended extensions and next work

---

## 1. Determine the domain and scope

**Domain:** Electronic Health Record (EHR) / claims-style *medical events*; specifically the MEDS conceptual model describing sequences of observations for subjects, and dataset metadata necessary for reproducible ML experiments.

**Intended users:**

* Data engineers and ETL authors converting raw sources into MEDS-compliant datasets.
* ML engineers and data scientists performing reproducible experiments.
* Ontology engineers and knowledge-graph practitioners integrating MEDS with other vocabularies (LOINC, SNOMED, OMOP).
* Tooling authors (validators, data catalogs, dataset profilers).

**Primary competency questions (CQs):**

1. For a given event, who is the subject and when did it occur?
2. Does the event carry a numeric or textual value (or additional modalities)?
3. What is the canonical code for this event and what human-readable description or parent codes exist?
4. Which subjects belong to the training split and which to held-out?
5. For a label sample, what is the prediction time and which label-type/value is present?
6. What provenance metadata (dataset name, MEDS version, ETL version) is associated with a dataset?
7. Which MEDS special codes (MEDS_BIRTH, MEDS_DEATH) are present?

These CQs drove the selection of classes, properties, and enforced constraints.

---

## 2. Reuse of existing ontologies and identifiers

**Reused patterns and vocabularies:**

* `rdfs:label`, `rdfs:comment` for human-readable annotations.
* **Dublin Core Terms (`dct:`)** for dataset-level metadata: `dct:title`, `dct:hasVersion`, `dct:created`, `dct:license`.
* **DCAT (`dcat:`)** for dataset distributions: `dcat:Dataset`, `dcat:Distribution`, `dcat:downloadURL`, `dcat:accessURL`.
* **PROV-O (`prov:`)** for provenance and ETL modeling: `prov:Activity`, `prov:wasGeneratedBy`, `prov:wasAssociatedWith`.
* `xsd:` datatypes for literal typing (`xsd:dateTime`, `xsd:string`, `xsd:double`, `xsd:boolean`, `xsd:integer`).

**Deliberate non-imports / minimal linking:**

* Clinical ontologies (SNOMED CT, LOINC, OMOP) are **not imported** directly to avoid licensing and scope complexity. The ontology supports linking to external codes (e.g., `skos:exactMatch`, `:externalCodeId`) so that downstream mappings can be created without embedding those ontologies.

**Rationale:** reuse infrastructure vocabularies for interoperability; avoid heavy clinical ontology imports to keep the core light and broadly reusable.

---

## 3. Enumerate important terms

High-level terms distilled from MEDS:

* Subject (primary entity, `subject_id`)
* Event (event/observation)
* Code (vocabulary metadata for codes)
* DatasetMetadata (dataset-level provenance)
* SubjectSplit (train / tuning / held_out)
* LabelSample (prediction sample)
* Value modalities (image, waveform, etc.)
* Special codes (MEDS_BIRTH, MEDS_DEATH)

---

## 4. Classes and class hierarchy (taxonomy)

**Modeling strategy:** *middle-out*, with `Event` as central.

**Top-level classes:**

* `meds:Subject` — primary entity; declared with a functional key `meds:subjectId`.
* `meds:Event` — core per-row observation. Declared disjoint with `meds:LabelSample`.
* `meds:Code` — vocabulary entry; required to have `meds:codeString`.
* `meds:DatasetMetadata` — dataset provenance & conversion metadata; **subclass of both `prov:Entity` and `dcat:Dataset`** so that provenance and catalog semantics apply.
* `meds:SubjectSplit` — enumeration-style class for dataset partitions (train/tuning/held_out).
* `meds:LabelSample` — supervised learning sample (subject × prediction_time × label).
* `meds:ValueModality` (abstract) with concrete subclasses (e.g., `meds:ImageValue`) to model extensions.

**Design note:** static events (no time) are modeled by optional `time` property rather than a bespoke `StaticEvent` subclass to remain flexible.

---

## 5. Class properties (slots)

**Object properties (selected):**

* `meds:hasSubject` (Event|LabelSample → Subject) — primary join.
* `meds:hasCode` (Event → Code) — link to code metadata.
* `meds:hasValueModality` (Event → ValueModality) — extensible modalities.
* `meds:assignedSplit` (Subject → SubjectSplit) — split assignment.
* `prov:wasDerivedFrom` (Event|Code|Subject|LabelSample → DatasetMetadata) — provenance.

**Datatype properties (selected):**

* `meds:subjectId` (Subject → xsd:string) — functional key.
* `meds:time` (Event → xsd:dateTime) — optional (0..1).
* `meds:codeString` (Event|Code → xsd:string) — canonical code literal.
* `meds:numericValue` (Event → xsd:double) — optional numeric.
* `meds:textValue` (Event → xsd:string) — optional textual.
* `meds:predictionTime` (LabelSample → xsd:dateTime) — required.
* `meds:booleanValue`, `meds:integerValue`, `meds:floatValue`, `meds:categoricalValue` (LabelSample) — mutually exclusive by design (SHACL enforces exactly-one-of).
* Dataset metadata: use **standard** properties where possible: `dct:title` (dataset_name), `dct:hasVersion` (dataset_version), `dct:created` (created_at), `dct:license` (license), `dcat:distribution` (location/description). MEDS-specific repeated-literal properties remain in `meds:` for column lists (e.g., `meds:rawSourceIdColumn`).

**Domain/range design principle:** define domains and ranges to support automatic validation and documentation, but avoid over-constraining open MEDS data.

---

## 6. Facets of the properties (constraints)

**OWL-level constraints (expressed as restrictions):**

* `meds:Event` has exactly one `meds:hasSubject` and OWL-level cardinality constraints guide modeling.
* `meds:Code` has a key on `meds:codeString`.
* `meds:LabelSample` must have one `meds:predictionTime` and at most one of each label datatype property.
* `prov:wasDerivedFrom` values for Events/Codes/Subjects/LabelSamples are declared to be `meds:DatasetMetadata`.

**SHACL-level constraints (enforcement & pragmatic choices):**

* SHACL shapes enforce practical validation:

  * `Event` requires either `meds:hasCode` (link to `meds:Code`) **or** `meds:codeString` literal (reflects MEDS flexibility).
  * `LabelSample` must contain **exactly one** of `{booleanValue, integerValue, floatValue, categoricalValue}` using `sh:or`.
  * `DatasetMetadata` requires `dct:title`, `meds:medsVersion`, and `dct:created` in the chosen profile.
  * `SubjectSplit.splitName` restricted to `train`, `tuning`, `held_out`.

**Why OWL + SHACL?**
OWL expresses conceptual constraints and supports reasoning; SHACL provides tractable, dataset-level validation (datatype correctness, cardinality, mutually exclusive label values) that is practical for ETL and CI pipelines.

---

## 7. Class instances (example uses)

Instances demonstrate recommended patterns:

* Dataset instance (note `dct:title`, `dct:hasVersion`, `dcat:distribution`, and ETL via `prov:Activity`):

  ```turtle
  :Dataset_MIMIC_DEMO a meds:DatasetMetadata ;
      dct:title "MIMIC-IV-Demo" ;
      dct:hasVersion "2.2" ;
      meds:medsVersion "0.3.3" ;
      dct:created "2025-03-28T13:47:38.809053"^^xsd:dateTime ;
      dcat:distribution :dist_dataset1 ;
      prov:wasGeneratedBy :etlActivity1 ;
      meds:tableName "data/train/0.parquet", "data/train/1.parquet" .
  ```

* ETL activity:

  ```turtle
  :etlActivity1 a prov:Activity ;
      rdfs:label "meds-etl" ;
      dct:hasVersion "0.0.4" ;
      rdfs:comment "ETL notes: normalization, mapping and table joins." .
  ```

* Event with code object or inline codeString fallback:

  ```turtle
  :meas1 a meds:Event ;
      meds:hasSubject :subj12345678 ;
      meds:hasCode :LAB_51237_UNK ;
      meds:time "2178-02-12T11:38:00"^^xsd:dateTime ;
      meds:numericValue 1.4 ;
      prov:wasDerivedFrom :Dataset_MIMIC_DEMO .
  ```

* Label sample:

  ```turtle
  :label_12345678_20210401 a meds:LabelSample ;
      meds:hasSubject :subj12345678 ;
      meds:predictionTime "2021-04-01T09:30:00"^^xsd:dateTime ;
      meds:booleanValue true ;
      prov:wasDerivedFrom :Dataset_MIMIC_DEMO .
  ```

---

## 8. Direct mapping table: MEDS data model → ontology

The following table maps MEDS components to the ontology elements in `ontology/meds.ttl`. Use this as the authoritative bridge when ingesting MEDS data into RDF.

> **Legend:**
> — MEDS = original MEDS concept / schema element (columns, files).
> — Ontology = class / property or instance in `ontology/meds.ttl`.
> — Enforcement = where the constraint is enforced (OWL, SHACL, or Procedural/ETL).

### 8.1 Core DataSchema → `meds:Event` and related

| MEDS element          |                                                                             Ontology mapping | Datatype / range                  |                 Required? (MEDS) | Enforcement                                                                                                                   |
| --------------------- | -------------------------------------------------------------------------------------------: | --------------------------------- | -------------------------------: | ----------------------------------------------------------------------------------------------------------------------------- |
| `subject_id` (column) | `meds:Event meds:hasSubject → meds:Subject`; `meds:Subject meds:subjectId` holds the literal | `xsd:string` for `meds:subjectId` |                              Yes | SHACL: EventShape requires `meds:hasSubject`; SHACL: SubjectShape requires `meds:subjectId`; OWL: `meds:subjectId` functional |
| `time` (column)       |                                                                       `meds:Event meds:time` | `xsd:dateTime`                    | Yes (nullable for static events) | SHACL: `time` `maxCount 1` + datatype                                                                                         |
| `code` (column)       |              `meds:Event meds:hasCode → meds:Code` **or** fallback `meds:codeString` literal | `xsd:string`                      |                              Yes | SHACL: `EventShape` `sh:or` enforces either `hasCode` or `codeString`; OWL: `Code` hasKey `meds:codeString`                   |
| `numeric_value`       |                                                                          `meds:numericValue` | `xsd:double`                      |                         Optional | SHACL: `numericValue` `maxCount 1`                                                                                            |
| `text_value`          |                                                                             `meds:textValue` | `xsd:string`                      |                         Optional | SHACL: `textValue` `maxCount 1`                                                                                               |

### 8.2 CodeMetadataSchema → `meds:Code`

| MEDS element   |                                              Ontology mapping | Datatype / range |                Required? | Enforcement                                                              |
| -------------- | ------------------------------------------------------------: | ---------------- | -----------------------: | ------------------------------------------------------------------------ |
| code rows      | `meds:Code` instances; `meds:codeString` holds canonical text | `xsd:string`     | Expected (MEDS guidance) | OWL: `meds:Code` `hasKey meds:codeString`; SHACL: CodeShape `minCount 1` |
| `description`  |                              `meds:Code meds:codeDescription` | `xsd:string`     |                 Optional | SHACL optional                                                           |
| `parent_codes` |                       `meds:Code meds:parentCode → meds:Code` | IRI              |                 Optional | SHACL can check node class if desired                                    |

### 8.3 DatasetMetadataSchema → `meds:DatasetMetadata`

| MEDS element                                             |                                                                                                   Ontology mapping | Datatype / range                                 |                             Required? (ontology profile) | Enforcement                                                                                                                                                                   |
| -------------------------------------------------------- | -----------------------------------------------------------------------------------------------------------------: | ------------------------------------------------ | -------------------------------------------------------: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dataset_name`                                           |                                                                                                        `dct:title` | `xsd:string`                                     |                                   **Required** (profile) | SHACL: `DatasetMetadataShape` requires `dct:title`; OWL: cardinality restriction                                                                                              |
| `dataset_version`                                        |                                                                                                   `dct:hasVersion` | `xsd:string`                                     |                                                 Optional | SHACL: `maxCount 1`                                                                                                                                                           |
| `meds_version`                                           |                                                                                                 `meds:medsVersion` | `xsd:string`                                     |                                             **Required** | SHACL: `minCount 1`                                                                                                                                                           |
| `created_at`                                             |                                                                                                      `dct:created` | `xsd:dateTime`                                   |                                             **Required** | SHACL: `minCount 1`                                                                                                                                                           |
| `license`                                                |                                                                                                      `dct:license` | IRI or literal (prefer IRI)                      |                                                 Optional | SHACL: maxCount 1 (IRI preferred)                                                                                                                                             |
| `location_uri` / `description_uri`                       |                             `dcat:distribution` → `dcat:Distribution` with `dcat:downloadURL` and `dcat:accessURL` | IRI                                              | Optional (but dataset profile requires >=1 distribution) | SHACL: check `dcat:distribution` node and its properties                                                                                                                      |
| `table_name`                                             |                                                                                      `meds:tableName` (repeatable) | `xsd:string`                                     |                                    Optional (repeatable) | SHACL: repeatable literal                                                                                                                                                     |
| `raw_source_id_columns`                                  |                                                                              `meds:rawSourceIdColumn` (repeatable) | `xsd:string`                                     |                                                 Optional | Repeated-literal property (one triple per column)                                                                                                                             |
| `code_modifier_columns`                                  |                                                                             `meds:codeModifierColumn` (repeatable) | `xsd:string`                                     |                                                 Optional | Repeated-literal                                                                                                                                                              |
| `additional_value_modality_columns`                      |                                                                  `meds:additionalValueModalityColumn` (repeatable) | `xsd:string`                                     |                                                 Optional | Repeated-literal                                                                                                                                                              |
| `site_id_columns`                                        |                                                                                   `meds:siteIdColumn` (repeatable) | `xsd:string`                                     |                                                 Optional | Repeated-literal                                                                                                                                                              |
| `other_extension_columns`                                |                                                                           `meds:otherExtensionColumn` (repeatable) | `xsd:string`                                     |                                                 Optional | Repeated-literal                                                                                                                                                              |
| `etl_name`, `etl_version`, `etl_notes`, `protocol_notes` | Represented on a `prov:Activity` linked via `prov:wasGeneratedBy` (`rdfs:label`, `dct:hasVersion`, `rdfs:comment`) | `xsd:string` and `xsd:dateTime` where applicable |                                                 Optional | Procedural: ETL authors should construct the `prov:Activity` instance; SHACL validates presence of `prov:wasGeneratedBy` pointing to a `prov:Activity` if required by profile |

> **Note:** ETL metadata is purposely **not** modeled as custom datatype properties on `meds:DatasetMetadata`. This keeps dataset metadata cleaner and uses PROV-O for provenance semantics.

### 8.4 SubjectSplitSchema → `meds:SubjectSplit`

| MEDS element          |                                                                           Ontology mapping | Datatype / range |            Required? | Enforcement                                                                       |
| --------------------- | -----------------------------------------------------------------------------------------: | ---------------- | -------------------: | --------------------------------------------------------------------------------- |
| `subject_id`, `split` | `meds:Subject meds:assignedSplit → meds:SubjectSplit` ; `meds:SubjectSplit meds:splitName` | `xsd:string`     | Optional per subject | SHACL: `SubjectSplitShape` restricts `splitName` to `train`, `tuning`, `held_out` |

### 8.5 LabelSchema → `meds:LabelSample`

| MEDS element                                            |                                                                                                              Ontology mapping | Datatype / range               |                        Required? | Enforcement                                     |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------: | ------------------------------ | -------------------------------: | ----------------------------------------------- |
| `subject_id`                                            |                                                                             `meds:LabelSample meds:hasSubject → meds:Subject` | IRI referencing `meds:Subject` |                         Required | SHACL: `LabelSampleShape` requires `hasSubject` |
| `prediction_time`                                       |                                                                                        `meds:LabelSample meds:predictionTime` | `xsd:dateTime`                 |                         Required | SHACL: `minCount 1`                             |
| label value columns (boolean/integer/float/categorical) | `meds:LabelSample` datatype properties (`meds:booleanValue`, `meds:integerValue`, `meds:floatValue`, `meds:categoricalValue`) | variety of xsd types           | Exactly one per sample (profile) | SHACL `sh:or` enforces exactly-one-of           |

---

## 9. Design alternatives considered and rationale

1. **Representation of label values**

   * **Option A (chosen):** Four datatype properties on `LabelSample` with SHACL enforcing exactly-one-of.
     *Rationale:* Simpler RDF serialization; aligns directly with MEDS `LabelSchema`. SHACL enforces mutual exclusivity.
   * **Option B (alternative):** Use `LabelValue` individuals (object property `:hasLabelValue`) with typed subclasses and a single cardinality of 1. Better for OWL reasoning but more verbose.

2. **`code` as literal vs resource**

   * **Option A (chosen):** Support both: `Event meds:hasCode → meds:Code` *or* inline `meds:codeString`.
     *Rationale:* Accommodates datasets without separate `codes.parquet` and enables richer semantics when `meds:Code` exists.
   * **Option B:** Force `hasCode` to always reference `Code` (simpler reasoning, penalizes datasets without `Code` table).

3. **Dataset metadata strictness**

   * **Option A (current):** Enforce `dct:title`, `meds:medsVersion`, `dct:created` in the SHACL profile.
   * **Option B:** Keep all metadata optional to mirror permissive JSON schema.

4. **ETL metadata modeling**

   * **Option A (chosen):** Model ETL as a `prov:Activity` and avoid custom datatype properties on `DatasetMetadata`.
     *Rationale:* Standard provenance semantics, easier to attach version/notes and chain activities.
   * **Option B:** Keep `etlName`, `etlVersion` as properties on `DatasetMetadata` (less expressive, avoided).

5. **Column lists representation**

   * **Option A (chosen):** Repeated-literal `meds:` datatype properties (one triple per column).
   * **Option B:** Model `Column` individuals for richer metadata (only if per-column metadata needed).

---

## 10. Enforcement strategy and limitations

**OWL role:** conceptual modeling, keys, class hierarchy, and disjointness. Useful for integration and limited inferencing.

**SHACL role:** practical validation of dataset instances (datatype correctness, cardinality, mutually exclusive label presence, presence of required dataset metadata). The repository supplies `shacl/meds-shapes.ttl` for this.

**Procedural role:** ETL and dataset-level checks for:

* Subject contiguity across physical shards (file-level constraint).
* Temporal ordering within a subject.
* Global uniqueness of `subjectId` for very large datasets (can be enforced via SPARQL shapes or external tooling).

**Limitations:**

* OWL cannot express file-level or record-ordering constraints.
* SHACL is expressive but may be heavy for extremely large graphs — prefer streaming validation or database-backed checks for scale.
* Some invariants (e.g., subject contiguity, file ordering) must be enforced by ETL processes or auxiliary scripts.

---

## 11. Recommended extensions and next steps

1. **LabelValue object-pattern**: migrate to object-pattern if stronger OWL reasoning about label values is required. I can supply a migration patch.
2. **SKOS for SubjectSplit**: model splits as a `skos:ConceptScheme` for cleaner vocab management.
3. **SPARQL-based SHACL constraints**: implement uniqueness checks (e.g., unique `subjectId`) as SPARQL SHACL shapes for rigorous validation.
4. **Provenance / audit trail**: extend PROV-O usage (multiple ETL activities, source raw dataset links).
5. **Integration examples**: provide mappings from `meds:Code` to external vocabularies via `skos:exactMatch` / `rdfs:seeAlso`.
6. **Validator toolchain**: provide a small CLI (Python) that runs SHACL, SPARQL uniqueness checks, and file-level contiguity/ordering checks on parquet files — a starter script is available on request.

---

## Appendix A — Quick reference: files in this repository

* `ontology/meds.ttl` — OWL ontology (Turtle)
* `shacl/meds-shapes.ttl` — SHACL shapes for validation (reflects the `dct:` and `prov:` changes)
* `examples/examples.ttl` — example dataset instances (refactored to use `dct:title`, `prov:Activity`, `dcat:Distribution`)
* `docs/` — (optional) generated Widoco docs for GitHub Pages
* `.github/workflows/validate.yml` — CI validation workflow
* `MODELING.md` — this document (detailed modeling rationale)

---

## Appendix B — Citation / provenance

Please cite the MEDS project and the original MEDS documentation when publishing datasets validated against this ontology. Suggested citation placeholder:

> Medical Event Data Standard (MEDS). GitHub: `https://github.com/Medical-Event-Data-Standard/med`. MEDS website: `https://medical-event-data-standard.github.io/`.

Include the ontology version and repository URL in academic publications for reproducibility.

---

## Acknowledgements

This modeling exercise follows **Deborah L. McGuinness & Natalya F. Noy — "Ontology Development 101"**. Design and implementation choices reflect the MEDS project’s philosophy and the practicalities of dataset validation in production environments.