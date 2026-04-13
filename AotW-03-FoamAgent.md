<p align="center">
  <img src="images/CAF-AotW-banner.svg" width="100%" alt="CAF AotW banner">
</p>


# 02/26/2026 — AotW#3: Foam-Agent — Automating OpenFOAM CFD with a Composable Multi-Agent System

## Science Story

Computational Fluid Dynamics (CFD) enables high-fidelity simulation for aerospace, energy, and many other engineering domains, but **real-world CFD remains difficult to use**: users must stitch together meshing, solver selection, boundary condition specification, numerical scheme tuning, runtime scripts, debugging, and visualization—often across both local machines and HPC clusters.

**Foam-Agent** is an LLM-based multi-agent framework that automates this end-to-end OpenFOAM workflow from a single natural language prompt, including **mesh generation/import, case file synthesis, HPC job script creation, execution, iterative self-correction, and post-processing visualization**. On **FoamBench (110 tasks)**, Foam-Agent reports an **88.2% executable success rate**—showing that specialized agents can substantially reduce the barrier to running correct, complete CFD simulations.

![Foam-Agent System Architecture](https://github.com/csml-rpi/Foam-Agent/blob/main/overview.png)

---

## Agentic Motivation

CFD automation is not a “single-file codegen” problem—it is a long-horizon workflow with strong constraints:

* **Multi-stage pipeline with hard dependencies.** OpenFOAM cases are a structured hierarchy (`system/`, `constant/`, `0/`, scripts, meshes). Many fields (e.g., turbulence model choice, transport properties, boundary patch names) must stay consistent across files.
* **Tool- and environment-heavy execution.** A realistic run may require meshing tools (OpenFOAM meshing or Gmsh), solver execution, MPI decomposition, Slurm submission, and post-processing libraries (ParaView/PyVista).
* **Deterministic failures demand iterative repair.** CFD runs fail with concrete error logs (missing fields, mismatched patch names, invalid scheme keywords). A capable system must **read logs, localize faults, patch files, and re-run** until success.
* **Different workflows need different “interfaces.”** Sometimes you want a full “prompt → results” pipeline; other times you want *composable* capabilities (e.g., “just generate a mesh” or “just write `fvSchemes`”) that can be invoked by IDE agents or external orchestrators.

Foam-Agent is designed around these realities: it plans, generates, runs, debugs, and visualizes, while also exposing core capabilities as reusable tools via MCP.

---

## Implementation

### 1) A workflow-aligned agent team (not a monolithic prompt)

Foam-Agent decomposes CFD automation into specialist agents that mirror the simulation lifecycle:

* **Architect Agent (Planner):** interprets the prompt, classifies the case (domain / solver / category), and produces a *file-and-folder plan* with required artifacts and generation order.
* **Meshing Agent:** supports multiple meshing pathways:

  * OpenFOAM-native meshing (e.g., `blockMeshDict`, `snappyHexMeshDict`)
  * **Gmsh-based generation** from natural-language geometry and boundary definitions (producing `.msh`, then converting via `gmshToFoam`)
  * **External mesh import** for user-provided meshes (notably `.msh`)
* **Input Writer Agent:** generates OpenFOAM configuration files (e.g., `controlDict`, `fvSchemes`, `fvSolution`, transport / turbulence properties, `0/U`, `0/p`, etc.).
* **Runner Agent (Local/HPC):** executes runs locally or prepares/launches cluster jobs (e.g., Slurm), capturing logs for analysis.
* **Reviewer Agent (Self-correct loop):** reads failure logs, proposes targeted fixes, and iterates until the case runs.
* **Visualization Agent:** generates and runs post-processing scripts (ParaView/PyVista) to output images for requested quantities.

This separation is the “Foam-Agent essence”: each agent has a well-scoped responsibility, and the system can execute *different paths* depending on the prompt (e.g., external mesh vs. generated mesh; local run vs. HPC run; visualization requested vs. skipped).

---

### 2) Hierarchical Multi-Index RAG: “right context, right time”

A core reason CFD agents fail is **retrieval noise**: a single, monolithic RAG index often retrieves irrelevant snippets (wrong solver, wrong physics, wrong file patterns), which then contaminates downstream file generation.

Foam-Agent instead uses **hierarchical / multi-index retrieval** to segment knowledge into stage-specific indices (e.g., case metadata, directory structures, file syntax patterns, execution scripts/commands). Practically, this enables:

* **Plan-time retrieval**: “what files should exist for this class of case?”
* **Write-time retrieval**: “what does a correct `fvSchemes` / turbulence model block look like for this solver?”
* **Run-time retrieval**: “what execution scripts and commands are typical for this workflow?”

This design matches how human experts work: first pick a *case family*, then reuse the correct structural template, then fill in details.

---

### 3) Dependency-aware and parallel file generation (two modes)

Foam-Agent recognizes a key tradeoff: **first-pass correctness vs. speed/cost**. It supports two Input Writer generation modes:

* **`sequential_dependency` (dependency-aware, higher first-pass success):**

  * Generates files **in a dependency-respecting order** (`system → constant → 0 → others`).
  * Later files may include earlier generated files as context, enforcing consistency across patch names, models, and parameters.
  * Best when runs are expensive (HPC jobs, long simulations), because it reduces “fail → review → rewrite” cycles.

* **`parallel_no_context` (parallel, fast iteration):**

  * Generates files **in parallel** without cross-file context (cheaper/faster prompts).
  * Relies on the Reviewer loop to correct mismatches after execution feedback.
  * Best when runs are cheap (quick local tests).

This explicit support for both modes is a practical highlight: Foam-Agent can behave like a careful “one-shot planner” for expensive CFD, or like a rapid prototyper when fast iteration is available.

---

### 4) Self-correcting execution loop grounded in solver logs

Foam-Agent’s reliability comes from treating OpenFOAM as the arbiter of correctness:

1. Generate a complete case
2. Run it (locally or via HPC scheduler)
3. If it fails, parse logs into structured error signals
4. **Reviewer Agent proposes minimal patches** that preserve user constraints
5. Apply fixes and re-run until success (or max iterations)

Because OpenFOAM errors are deterministic and specific (e.g., missing dictionary entries, undefined keywords, patch mismatches), this “execute → diagnose → patch” cycle is much more robust than purely prompt-based generation.

---

### 5) Meshing + HPC + visualization are first-class, not afterthoughts

Many “CFD LLM agents” stop at writing solver dictionaries. Foam-Agent pushes beyond that by integrating the steps that dominate real workflows:

* **Mesh generation/import**

  * Supports **Gmsh `.msh`** workflows (notably GMSH ASCII 2.2 format)
  * Converts meshes into OpenFOAM-compatible format and proceeds with the case setup
* **HPC execution**

  * Generates Slurm scripts and run wrappers for large cases
  * Supports parallelization patterns (e.g., decomposition into subdomains for MPI runs)
* **Visualization**

  * Produces ParaView/PyVista scripts that can be executed to render requested fields

These capabilities make Foam-Agent closer to an “end-to-end CFD co-pilot” than a file generator.

---

### 6) MCP support: composable CFD tools for other agent ecosystems

Foam-Agent can be used as a **Model Context Protocol (MCP) server**, exposing its capabilities as callable tools (e.g., plan structure, generate file content, generate mesh, run simulation, check status, generate visualization). This matters because:

* IDE agents (Cursor / Claude Code) can invoke Foam-Agent tools directly.
* Users can build custom orchestrations (e.g., mesh-only workflows, or “generate + validate + commit” pipelines).
* Foam-Agent becomes an *infrastructure component* for larger agentic systems, not a closed application.

---

### 7) Practical deployment: Docker + flexible model backends

Foam-Agent emphasizes “works on real machines”:

* **Docker-first setup** via a prebuilt image (`leoyue123/foamagent`) that includes OpenFOAM and a ready-to-run Foam-Agent environment.
* **Multiple LLM backends**, including:

  * OpenAI API key (usage-based)
  * Anthropic API key (usage-based)
  * AWS Bedrock
  * **Ollama (local models)** for open-source / on-prem deployments
  * **ChatGPT/Codex subscription sign-in** (experimental, no API key), leveraging a local Codex auth cache

This backend flexibility is a strong practical differentiator: teams can choose between frontier models, enterprise endpoints, or local open-source deployments depending on constraints.

---

## To Know More

### Source Code

* **Repository:** [https://github.com/csml-rpi/Foam-Agent](https://github.com/csml-rpi/Foam-Agent)
* **DeepWiki (interactive docs):** [https://deepwiki.com/csml-rpi/Foam-Agent](https://deepwiki.com/csml-rpi/Foam-Agent)

### Papers & Benchmarks

* **Foam-Agent 2.0 (Composable end-to-end framework, MCP, 88.2% on FoamBench):** [https://arxiv.org/abs/2509.18178](https://arxiv.org/abs/2509.18178)
* **Foam-Agent (earlier version):** [https://arxiv.org/abs/2505.04997](https://arxiv.org/abs/2505.04997)
* **CFDLLMBench / FoamBench benchmark:** [https://arxiv.org/abs/2509.20374](https://arxiv.org/abs/2509.20374)

---

*Last Updated: 04/08/2026*
*Contributed by: Ling Yue, [yuel2@rpi.edu](mailto:yuel2@rpi.edu). Shaowu Pan, [pans2@rpi.edu](mailto:pans2@rpi.edu)*
