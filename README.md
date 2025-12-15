<h1 align="center">MEDS Ontology (OWL)</h1>

<p align="center">
  <!-- Build & Validation -->
  <img src="https://github.com/hekameds.github.io/meds-ontology/actions/workflows/validate.yml/badge.svg" alt="SHACL Validation"/>
  <img src="https://img.shields.io/badge/RDF%20syntax-Valid-brightgreen" alt="RDF Syntax Valid"/>
  <img src="https://img.shields.io/badge/OWL%202-✓-blue" alt="OWL 2"/>
  <img src="https://img.shields.io/github/v/release/hekameds.github.io/meds-ontology" alt="Latest Release"/>
  <img src="https://img.shields.io/badge/Ontology%20version-0.1.0-blueviolet" alt="Ontology Version"/>
  <img src="https://img.shields.io/github/license/hekameds.github.io/meds-ontology" alt="License"/>
    <!-- <a href="https://hekameds.github.io/meds-ontology">
    <img src="https://img.shields.io/badge/docs-GitHub%20Pages-blue" alt="Documentation"/>
  </a>
  <a href="https://github.com/dgarijo/Widoco">
    <img src="https://img.shields.io/badge/docs-Widoco%20generated-9cf" alt="Widoco"/>
  </a> -->

  <!-- DOI placeholder (update when minted)
  <a href="https://zenodo.org/">
    <img src="https://img.shields.io/badge/DOI-Zenodo-grey" alt="DOI"/>
  </a> -->
</p>

This repository contains an OWL ontology and SHACL shapes that formally encode the **Medical Event Data Standard (MEDS)** conceptual data model.  
It provides a semantic framework for representing subjects, clinical measurements, code metadata, dataset-level provenance, subject splits, and prediction labels.

This repository includes:

```
.
├── ontology/            # OWL ontology
├── shacl/               # SHACL validation shapes
├── examples/            # Example RDF instance data
├── docs/                # Widoco-generated documentation
├── modeling.md          # Detailed modeling rationale (Ontology 101)
└── README.md
```

## References

This ontology is based on the official **Medical Event Data Standard (MEDS)** specifications:

- MEDS GitHub: https://github.com/Medical-Event-Data-Standard/med
- MEDS Website / Docs: https://medical-event-data-standard.github.io/

The OWL ontology and SHACL shapes are **direct semantic translations** of the MEDS schemas:
DataSchema, CodeMetadataSchema, DatasetMetadataSchema, SubjectSplitSchema, and LabelSchema.  
A detailed alignment is provided in `MODELING.md`.

## Documentation
This ontology provides multiple ways to explore and understand its structure:

- **Interactive Visualization (WebVOWL):**  
  Explore the ontology graphically, including classes, properties, and hierarchy.  
  [Open in WebVOWL](https://service.tib.eu/webvowl/#iri=https://hekameds.github.io/meds-ontology/ontology.owl)

- **Full Ontology Documentation (Widoco):**  
  Access comprehensive class/property descriptions, annotations, and usage examples.  
  [View the documentation](https://hekameds.github.io/meds-ontology)

- **Modeling Rationale:**  
  For a rigorous discussion of design decisions, alignment with MEDS schemas, and adherence to *Ontology Development 101*, see [`MODELING.md`](MODELING.md).  
  This includes tables, mapping details, and constraints implemented in OWL and SHACL.

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

## Citation

If you use this ontology, please cite the MEDS project and this repository (citation block TBD).

