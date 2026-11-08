# Scientific PDF-to-Markdown Preprocessing for NLO RAG

## Purpose

This document records the proposed preprocessing architecture for the CK NLO research knowledge base before scaling from a small pilot set to approximately 100–300 research papers.

The main idea is:

> **Do not embed raw scientific PDFs directly when a faithful structured representation can be created first.**

For the NLO AutoResearch system, the preferred path is:

```text
Original PDF
    ↓
Faithful scientific document conversion
    ↓
Structured Markdown
    ↓
BGE-M3 embeddings
    ↓
LanceDB
    ↓
Accuracy-optimised retrieval
    ↓
Local LLM
    ↓
Grounded scientific answer
```

The Markdown should be a **faithful structural conversion**, not an AI-written summary.

---

## Why this is needed

The first six-paper AnythingLLM pilot exposed several retrieval problems when embedding PDFs directly:

- relevant facts could be present in the paper but not retrieved;
- important values located in tables could be missed;
- the reference section could contaminate retrieval;
- document titles and source identity could be confused;
- duplicated chunks could appear;
- a small LLM could fill missing retrieved evidence with plausible but unsupported scientific statements.

An example from the pilot was the 2022 Journal of Molecular Structure paper. The requested static and experimental second hyperpolarizability values were explicitly present in Table 6 and Table 8, but the RAG sometimes returned “insufficient evidence” because the relevant chunks were not retrieved.

This indicates that the quality of the knowledge representation before embedding is important, not only the choice of embedding model or LLM.

---

## Core rule

### Convert, do not interpret

The preprocessing stage must preserve the scientific source exactly as far as practical.

It should:

- preserve numerical values;
- preserve units;
- preserve compound labels;
- preserve equations;
- preserve tables;
- preserve section hierarchy;
- preserve experimental conditions;
- preserve the distinction between experimental and theoretical results;
- preserve figure captions;
- preserve exact article metadata.

It must **not**:

- rewrite conclusions;
- infer missing values;
- harmonise conflicting values silently;
- merge experimental and theoretical quantities;
- invent chemical explanations;
- paraphrase article titles;
- replace the authors' wording with a model-generated interpretation;
- use general scientific knowledge to fill gaps.

If the source is ambiguous or internally inconsistent, the Markdown should preserve that ambiguity and, where useful, add a clearly labelled note such as:

```text
SOURCE NOTE: Values reported in the Results section differ from values repeated in the Conclusion.
```

The processor must not silently choose one as correct.

---

## Proposed two-layer knowledge architecture

The original PDFs remain the permanent source of truth.

```text
                 Original research PDFs
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
      Source-of-truth archive   Structured Markdown
                                RAG-ready representation
                                     │
                                     ↓
                                   BGE-M3
                                     │
                                     ↓
                                  LanceDB
```

### Layer 1 — Original PDF

Used for:

- provenance;
- visual verification;
- final checking of tables, equations and figures;
- audit trail;
- correction of parsing mistakes.

### Layer 2 — Structured Markdown

Used for:

- embedding;
- retrieval;
- local RAG;
- AutoResearch literature reasoning.

The Markdown should always retain a direct reference to the original PDF filename and DOI.

---

## Recommended per-paper file structure

A paper should ideally be split into at least two logical files.

### Main scientific evidence

```text
JMS_2022_FRGS.main.md
```

Contains:

- metadata;
- abstract;
- introduction;
- methodology;
- experimental conditions;
- computational methods;
- results;
- discussion;
- conclusions;
- tables;
- equations;
- figure captions.

This file is embedded into the **primary scientific evidence RAG**.

### References

```text
JMS_2022_FRGS.references.md
```

Contains only the bibliography/reference list.

This file should **not** be embedded into the primary evidence RAG.

It may later be embedded into a separate **literature discovery RAG**.

This separation prevents a common failure:

```text
Question: Which uploaded papers discuss third-order NLO?
          ↓
Raw PDF retrieval finds bibliography titles
          ↓
LLM incorrectly treats cited papers as uploaded workspace papers
```

---

## Suggested Markdown template

```markdown
---
document_id: CK-NLO-0001
source_file: JMS_2022_FRGS.pdf
title: Exact article title
authors:
  - Exact author list
journal: Journal of Molecular Structure
year: 2022
volume: 1267
article_number: 133584
doi: 10.1016/j.molstruc.2022.133584
document_type: primary_research
research_group: CK
research_area:
  - chalcone
  - nonlinear optics
  - Z-scan
  - DFT
experimental:
  z_scan: true
  laser_regime: CW
  wavelength_nm: 532
computational:
  dft: true
  software:
    - Gaussian09
---

# Abstract

Exact source text or faithful extraction.

# Introduction

...

# Experimental

## Synthesis

...

## Z-scan

...

# Computational Methods

...

# Results and Discussion

## Static and Frequency-Dependent Hyperpolarizabilities

### Table 6 — Static (Hyper)polarizabilities

| Compound | gamma_static |
|---|---:|
| I | ... |
| II | ... |
| III | ... |

## Third-Order NLO

### Table 8 — Experimental Third-Order NLO Parameters

| Compound | alpha2 | n2 | chi3 | gamma_experimental |
|---|---:|---:|---:|---:|
| I | ... | ... | ... | ... |
| II | ... | ... | ... | ... |
| III | ... | ... | ... | ... |

# Conclusion

...

# Figure Captions

...

# Source Notes

- Any parsing uncertainty.
- Any internal inconsistency retained from the paper.
- Any table or equation requiring visual verification.
```

---

## Metadata strategy

Each paper should have a canonical identity independent of the filename.

Recommended fields:

- `document_id`
- `source_file`
- exact `title`
- `authors`
- `journal`
- `year`
- `volume`
- `issue`
- `article_number`
- `doi`
- `document_type`
- `research_group`
- keywords/research areas
- experimental techniques
- laser regime
- excitation wavelength
- computational software
- functional
- basis set
- solvent model

This supports later queries such as:

- Which papers used femtosecond Z-scan?
- Which papers used CAM-B3LYP?
- Which studies contain experimental gamma rather than computational gamma?
- Which compounds were computational only?
- Which papers used DMF?
- Which papers reported higher-order NLO beyond third order?

---

## Scientific table handling

Tables must be treated as first-class scientific evidence.

Do not reduce a table to a loose prose summary when exact values can be retained.

For each table:

1. preserve the table number and title;
2. preserve units;
3. preserve column headings;
4. preserve compound identifiers;
5. preserve symbols exactly where possible;
6. keep experimental and theoretical columns distinct;
7. add a source-page marker if practical.

This is particularly important for:

- NLO parameter tables;
- static and dynamic hyperpolarizabilities;
- Z-scan coefficients;
- SCXRD parameters;
- UV-Vis data;
- excitation energies;
- transition dipole moments;
- NMR assignments;
- DFT descriptors.

---

## Equation handling

Equations should be retained in readable Markdown/LaTeX form.

Where conversion is uncertain:

```markdown
SOURCE EQUATION — verify against original PDF page 12
```

The preprocessing model should never derive a corrected equation unless explicitly asked in a later scientific review stage.

---

## Figure handling

The Markdown should preserve:

- figure number;
- figure caption;
- nearby explanatory text.

If a figure contains essential quantitative information not available in text, mark it:

```text
FIGURE_REQUIRES_VISUAL_REVIEW: true
```

The original PDF remains the source for visual checking.

---

## Reference-section policy

For the primary RAG:

> References are not primary scientific evidence unless the cited paper itself has been uploaded and indexed as its own source document.

This distinction should be explicit:

- **workspace/source paper finding**
- **literature cited by that paper**

A primary-RAG answer must not turn a bibliography entry into a result attributed to the uploaded paper.

---

## Proposed processing workflow for 100–300 papers

Do not immediately process the full collection.

### Pilot

Use the current six-paper benchmark.

1. Keep original PDFs.
2. Convert each PDF to structured Markdown.
3. Separate main text and references.
4. Add canonical metadata.
5. Embed only the `.main.md` files using BGE-M3.
6. Store vectors in LanceDB.
7. Run the same benchmark questions used for the raw-PDF pilot.
8. Compare retrieval accuracy.

### Scale-up

Only after the structured-Markdown pilot is clearly better:

1. process 20 papers;
2. validate retrieval;
3. process 50 papers;
4. validate again;
5. proceed to 100–300 papers.

This staged approach avoids creating a large low-quality vector database that later requires complete rebuilding.

---

## Benchmark criteria

The RAG should be tested for at least four capabilities.

### 1. Exact extraction

Example:

- donor–acceptor configuration;
- exact chi(3);
- exact gamma;
- exact wavelength.

### 2. Table retrieval

Example:

- retrieve static gamma from Table 6;
- retrieve experimental gamma from Table 8.

### 3. Evidence separation

Example:

- distinguish experimental result from DFT prediction;
- distinguish uploaded paper from bibliography citation.

### 4. Cross-paper synthesis

Example:

- compare recurring structure–property factors across several papers while citing the correct source documents.

---

## Relationship to the AutoResearch architecture

The cleaned RAG is intended to become the scientific long-term memory layer of the NLO AutoResearch system.

```text
AutoResearch Agent
        │
        ├── ML
        │     └── finds quantitative structure–property patterns
        │
        ├── Physics Reasoning
        │     ├── RAG scientific knowledge
        │     ├── physics rules
        │     └── mechanistic hypotheses
        │
        ├── DFT / Multiwfn
        │     └── generates mechanistic evidence
        │
        └── Decision Layer
              └── selects next molecules / calculations
```

Working principle:

> **ML finds patterns, Physics validates mechanisms, RAG provides scientific knowledge, DFT provides physical evidence, and AutoResearch decides the next step.**

The RAG therefore must prioritise provenance, fidelity and retrieval precision over fluent summarisation.

---

## Current recommended local stack

Pilot configuration:

```text
Structured Markdown
        ↓
BGE-M3 via Ollama
        ↓
LanceDB
        ↓
Accuracy Optimized retrieval
        ↓
~6 high-quality chunks
        ↓
Local Qwen model
        ↓
Grounded answer
```

The LLM should be instructed to:

- answer only from retrieved evidence for library questions;
- preserve exact numerical values;
- distinguish static, dynamic and experimental quantities;
- avoid reconstructing article titles;
- avoid unsupported mechanistic explanations;
- state “insufficient evidence” when retrieval does not provide the requested fact.

---

## Design principle

A better embedding model alone is not enough.

The expected hierarchy of improvements is:

1. better source representation;
2. cleaner metadata;
3. reference separation;
4. stronger embedding model;
5. improved retrieval/reranking;
6. sufficient context window;
7. stronger LLM if needed.

For a scientific AutoResearch knowledge base, the quality of the evidence passed to the LLM is more important than simply increasing LLM size.

---

## Status

**Proposed architecture — to be validated using the existing six-paper NLO benchmark before scaling to the full research library.**
