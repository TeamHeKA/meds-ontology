# MEDS Ontology (OWL)

<!-- Build & Validation -->
<!-- ![SHACL Validation](https://github.com/albertomarfoglia/meds-ontology/actions/workflows/validate.yml/badge.svg)
![Lint](https://img.shields.io/badge/RDF%20syntax-Valid-brightgreen)
![OWL](https://img.shields.io/badge/OWL%202-✓-blue) -->

<!-- Documentation -->
[![Documentation](https://img.shields.io/badge/docs-GitHub%20Pages-blue)](https://albertomarfoglia.github.io/meds-ontology/)
[![Widoco](https://img.shields.io/badge/docs-Widoco%20generated-9cf)](https://github.com/dgarijo/Widoco)

<!-- Versioning -->
![Release](https://img.shields.io/github/v/release/albertomarfoglia/meds-ontology)
![Version](https://img.shields.io/badge/Ontology%20version-0.1.0-blueviolet)

<!-- License -->
![License](https://img.shields.io/github/license/albertomarfoglia/meds-ontology)

<!-- DOI (Zenodo) – replace with real DOI once minted -->
[![DOI](https://img.shields.io/badge/DOI-Zenodo-grey)](https://zenodo.org/)

This repository contains an OWL ontology and SHACL shapes that formally encode the **Medical Event Data Standard (MEDS)** conceptual data model.  
It provides a semantic framework for representing subjects, clinical measurements, code metadata, dataset-level provenance, subject splits, and prediction labels.

This repository includes:

- `ontology/meds.ttl` — OWL ontology (Turtle). Base IRI: `https://albertomarfoglia.github.io/meds/0.1.0/ontology#`.
- `shacl/meds-shapes.ttl` — SHACL shapes implementing MEDS validation rules.
- `examples/examples.ttl` — RDF instance examples.
- `docs/` — Widoco-generated documentation (published via GitHub Pages).
- `modeling.md` — **full modeling rationale** and detailed mapping to MEDS (based on McGuinness & Noy’s *Ontology Development 101*).
- `.github/workflows/validate.yml` — CI pipeline for SHACL validation.

## References

This ontology is based on the official **Medical Event Data Standard (MEDS)** specifications:

- MEDS GitHub: https://github.com/Medical-Event-Data-Standard/med
- MEDS Website / Docs: https://medical-event-data-standard.github.io/

The OWL ontology and SHACL shapes are **direct semantic translations** of the MEDS schemas:
DataSchema, CodeMetadataSchema, DatasetMetadataSchema, SubjectSplitSchema, and LabelSchema.  
A detailed alignment is provided in `modeling.md`.

## Documentation

The complete ontology documentation is available in `docs/` (generated with Widoco).

For a rigorous and academically motivated description of all modeling decisions, see:

👉 **`modeling.md` — Important Modeling Considerations & Design Rationale**  
(including mappings to MEDS, tables, and adherence to McGuinness & Noy’s *Ontology Development 101*)

This file explains:

- ontology domain & scope  
- reuse strategy  
- term enumeration  
- class hierarchy & conceptual design  
- object/datatype properties  
- constraints and validation  
- design alternatives considered  
- justification of OWL vs SHACL boundaries  
- detailed MEDS → OWL mappings (tabular)

## Goals & Scope

This ontology captures the core conceptual structure defined in the MEDS standard:

- `Subject`  
- `Measurement` (events/observations with timestamp and code)  
- `Code` (vocabulary entries)  
- `DatasetMetadata`  
- `SubjectSplit`  
- `LabelSample` (prediction samples)  
- extensible value modalities (e.g., image, waveform data)

It prioritizes **semantic clarity**, **reusability**, and **validation support**.

Operational dataset constraints (e.g., sorted order, contiguous subject blocks) are documented but intentionally **not encoded** in OWL/SHACL.

## Quickstart

### View the ontology
Open `ontology/meds.ttl` in Protégé.

### Validate dataset instances (locally)
Install `pyshacl`:

```bash
python -m pip install pyshacl rdflib
```

Run validation:

```bash
pyshacl -s shacl/meds-shapes.ttl -i examples/examples.ttl -f ttl
```

## Repository structure
```
.
├── ontology/            # OWL ontology
├── shacl/               # SHACL validation shapes
├── examples/            # Example RDF instance data
├── docs/                # Widoco-generated documentation
├── modeling.md          # Detailed modeling rationale (Ontology 101)
└── README.md
```

## Citation

If you use this ontology, please cite the MEDS project and this repository (citation block TBD).

