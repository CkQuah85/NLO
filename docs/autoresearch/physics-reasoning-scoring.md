# Physics Reasoning and Scoring in NLO AutoResearch

## Core principle

The Physics Agent should **not directly invent a Physics Score**. The architecture separates reasoning from scoring:

- **LLM / Physics Agent:** interpret results, connect mechanisms, retrieve relevant NLO knowledge, identify supporting and conflicting evidence.
- **RAG:** ground the reasoning in NLO literature and domain knowledge.
- **Gaussian / ORCA / Multiwfn:** generate the physical and electronic-structure evidence.
- **Python / deterministic rules:** calculate the numerical Physics Score from a predefined rubric.

This keeps the Physics Score reproducible and auditable while allowing the LLM to perform higher-level scientific reasoning.

## Proposed workflow

```text
Candidate molecule
      |
      +--> ML model ------------------------> ML score
      |
      +--> DFT / TD-DFT / Multiwfn
                    |
                    v
             Physics descriptors
                    |
                    v
        NLO Physics Reasoning Agent
          (LLM + RAG + rules)
                    |
          mechanism interpretation
                    |
                    v
        deterministic scoring engine
                    |
                    v
              Physics score
                    |
                    +----------------------+
                                           |
ML score ----------------------------------+
                                           v
                              Acquisition / ranking score
                                           |
                                  Select candidates
                                           |
                                     New DFT data
                                           |
                                      Retrain ML
                                           |
                                          loop
```

## Example Physics Score

A possible structure is

```text
S_Physics =
    w1 * S_SOS
  + w2 * S_CT
  + w3 * S_conjugation
  + w4 * S_electronic
  + w5 * S_mechanism_consistency
  + w6 * S_literature_support
```

Possible evidence:

| Component | Possible evidence |
|---|---|
| SOS | dominant states, excitation energy, transition dipole, delta-mu, state contributions to gamma |
| Charge transfer | hole-electron analysis, CT distance, overlap, delta-r |
| Conjugation | BLA, HOMA, ELF-pi |
| Electronic structure | HOMO-LUMO gap, MPI, ESP, pEDA, cSAR |
| Mechanism consistency | whether independent descriptors support a coherent physical mechanism |
| Literature support | evidence retrieved from the NLO RAG knowledge base |

The LLM is especially useful for **mechanism consistency** and **literature interpretation**, but the final score should be calculated from frozen rules wherever possible.

## ML and Physics must remain distinct

The ML model retains its own validation metrics, e.g. Q2, RMSE and MAE. Physics reasoning is not inserted into the calculation of Q2.

An acquisition score may subsequently combine both layers, for example:

```text
S_final = 0.70 * S_ML + 0.30 * S_Physics
```

The 70:30 ratio is a starting hypothesis, **not an assumed optimum**. It should be calibrated and tested.

## Important scientific caution

Descriptors should not automatically be treated as monotonic rules. For example:

- a smaller HOMO-LUMO gap does not necessarily imply a larger gamma;
- stronger charge transfer is not necessarily always beneficial;
- different excited states can contribute with different signs;
- SOS terms can reinforce or cancel one another.

Therefore, Phase II should establish the NLO structure-property relationships and determine which physics descriptors are sufficiently reliable to enter the scoring rubric.

## Proposed Phase II -> Phase III transition

**Phase II**

Experimental + DFT dataset -> NLO SAR -> identify robust physics descriptors -> calibrate Physics Scoring Function.

**Phase III**

Freeze the Physics Scoring Function -> deploy it inside AutoResearch -> prospectively evaluate new candidates.

This prevents the scoring rules from being repeatedly changed to favour the desired result.

## Validation experiment

A strong methodological comparison is:

**ML-only Active Learning**

versus

**ML + Physics-guided Active Learning**

Compare, over repeated iterations:

- best gamma discovered;
- hit rate for high-performing candidates;
- number of expensive DFT calculations required;
- chemical-space exploration;
- extrapolation to new molecular families;
- stability of candidate ranking.

The Physics Score therefore becomes a **testable scientific hypothesis**, rather than an arbitrary LLM judgement.

## Role of the NLO RAG

There is no immediate need to train a dedicated NLO LLM. A more practical architecture is:

**general LLM + NLO RAG + physics rules + MCP computational tools**

The RAG supplies domain knowledge and literature evidence; the LLM reasons over that evidence; computational tools provide molecule-specific physical evidence; deterministic code produces the score.

### Design principle

> ML finds statistical patterns. Physics tests and explains the patterns. RAG supplies grounded domain knowledge. DFT supplies molecule-specific physical evidence. AutoResearch decides what to investigate next.
