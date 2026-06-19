<p align="center">
  <img src="../images/CAF-AotW-banner.svg" width="100%" alt="CAF AotW banner">
</p>

# [Date] &mdash; AotW#[Number]: PISCES &mdash; Multi-Model Consensus Extraction of Biochemical Process Flowsheets

---

## Science Story

Designing economically viable biomanufacturing processes requires comparing dozens of published flowsheets — each buried inside a PDF with its own notation, unit system, and diagram style. Techno-Economic Analysis (TEA) depends on this data, but extracting it by hand is a weeks-long task even for experienced process engineers. A single paper may describe a fermentation train, a separation cascade, and a purification step across five pages of text and one dense Process Flow Diagram (PFD), and the researcher must reconcile all of it into structured numbers before any comparative analysis can begin.

**PISCES** (Process Information from Chemical and Engineering Sources), developed at Lawrence Berkeley National Laboratory, automates this pipeline entirely. Feed it a scientific PDF and it returns a validated, machine-readable **Standard Flowsheet Format (SFF) JSON** — complete with unit operations, stream topology, chemical identities, flow rates, and operating conditions — ready for direct ingestion into BioSTEAM or any TEA platform. PISCES is designed to build the large-scale process databases needed for comparative analysis of bio-based production pathways, directly supporting LBNL's missions in sustainable energy research and biotechnology.

---

## Agentic Motivation

Extracting a process flowsheet from a scientific paper is not a lookup task — it is a multi-step reasoning problem that involves document layout analysis, diagram interpretation, unit reconciliation, and cross-field consistency checking. A single LLM call or a simple OCR pipeline reliably fails on the ambiguities that appear in real literature. PISCES addresses this through a **consensus architecture** that turns model disagreement from a hidden failure into an explicit, auditable signal:

- **Diversity through model heterogeneity:** Three Extractor Agents — built on Gemini, Claude, and Grok — independently parse the same document using different hybrid input strategies (native-PDF versus text-plus-image rendering). No single model's blind spots dominate the output.
- **Diagram-first routing:** A lightweight **Visual Scout** agent identifies the PFD page and bounding box before any extraction begins, focusing the heavier agents on the relevant region.
- **Resolver-driven consensus:** A **Resolver Agent** merges the three extractions field by field, promoting values where at least two agents agree and flagging disagreements for downstream handling.
- **Logic-audited output:** An **Evaluator Agent** performs a final consistency pass (every unit must have at least one input and output stream; mass balances must be internally plausible) and assigns per-field confidence scores.
- **Human-in-the-loop escalation:** Low- and medium-confidence records are automatically routed to a validation UI for expert review before database insertion — preserving automation speed while ensuring data quality on the hard cases.

---

## Implementation

PISCES is implemented as a modular **Python AsyncIO pipeline** deployed as a containerized service on the LBNL CBORG platform (`api.cborg.lbl.gov`). The five pipeline stages execute sequentially per document:

1. **Visual Scout** — a lightweight vision model returns the PFD page index and bounding-box crop.
2. **Parallel Extractors (×3)** — Gemini 3.0 Pro, Claude 4.5 Sonnet, and Grok 4 each produce an independent SFF JSON candidate; a `json_repair` middleware step corrects truncated or malformed LLM output.
3. **Resolver** — merges the three candidates using field-level consensus logic; multi-way disagreements are flagged with low confidence.
4. **Evaluator** — validates the merged SFF for stream connectivity and emits a confidence score.
5. **HITL Gateway** — confident records are inserted into the process database; records below threshold go to a Streamlit-based expert review UI.

The workflow is stateless per document, enabling horizontal scaling. Pydantic schema validation is enforced throughout to ensure that every SFF output conforms to a fixed, versioned schema regardless of which model produced which field.

---

## To Know More

### Source Code
- **Repository:** https://github.com/sustainability-software-lab/project-pisces-extraction
- **License:** Lawrence Berkeley National Labs BSD variant license (SPDX: BSD-3-Clause-LBNL)

### Additional Resources
- **Website:** https://projectpisces.org/
- **Contact:** Yuting Chen — yutingchen@lbl.gov
- **Contact:** Tyler Huntington — tylerhuntington222@lbl.gov
- **Contact:** Corinne Scown — cdscown@lbl.gov

---

*Last Updated: [Date]*
*Contributed by: Yuting Chen, Tyler Huntington, Corinne Scown, Meili Gong, Sarang Bhagwat — Lawrence Berkeley National Laboratory*
