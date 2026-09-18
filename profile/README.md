# Formulary

A curated collection of composable open-source compliance tools.

Each tool does one thing. Every tool reads and writes [gemara](https://github.com/gemaraproj/gemara)-compatible artifacts. Any tool's output is a valid input to the next. No tool requires an AI model to operate — but all are designed to be orchestrated by one.

---

## Why Formulary

Manual compliance work does not scale. A compliance program manager running multiple certifications simultaneously is doing the same extraction, mapping, and documentation work across each one — most of it deterministic.

Formulary separates the deterministic work (citation extraction, coverage math, evidence scheduling, drift diffing, document assembly) from the judgment work (prioritization, communication, stakeholder management). The deterministic work runs in a terminal or a CI pipeline. The judgment work is where human and AI attention belongs.

The name comes from pharmacology: a **formulary** is an official curated collection of treatments, each precisely specified, each independently useful, each combinable with others. Compliance programs need the same thing — a shelf of tested, precise tools you can reach for when the work demands it.

---

## Tools

| Tool | Theme word | What it does |
|---|---|---|
| [substrate](https://github.com/Formulary-Labs/substrate) | lab growth medium | Core library — gemara SDK integration, shared CLI conventions, provenance writer |
| [probe](https://github.com/Formulary-Labs/probe) | diagnostic probe | Validate a gemara artifact — clean exit codes for CI |
| [assay](https://github.com/Formulary-Labs/assay) | systematic test | Control assessment engine — fill templates from product docs, produce gemara Layer 5 artifacts |
| [titer](https://github.com/Formulary-Labs/titer) | concentration measure | Control coverage matrix — coverage % by family, owner gaps, evidence gaps |
| [specimen](https://github.com/Formulary-Labs/specimen) | collected sample | Risk register and POA&M — with post-audit feed-forward ingest |
| [dose](https://github.com/Formulary-Labs/dose) | dosing schedule | Evidence collection calendar — `.ics` and markdown, shift-left scheduled |
| [exhibit](https://github.com/Formulary-Labs/exhibit) | examination exhibit | Auditor-filtered compliance posture view — static HTML, no internal data |
| [challenge](https://github.com/Formulary-Labs/challenge) | challenge test | Adversarial compliance artifact interrogation — 10 deterministic patterns |
| [vital](https://github.com/Formulary-Labs/vital) | vital signs | Program health snapshot — internal dashboard for program managers |
| [decay](https://github.com/Formulary-Labs/decay) | radioactive decay | Longitudinal compliance drift detection — 13 named patterns across 2+ audit cycles |
| [scan](https://github.com/Formulary-Labs/scan) | diagnostic scan | External regulatory and threat monitoring — 5 source categories, structured relevance scoring |
| [compound](https://github.com/Formulary-Labs/compound) | compounding | Management system document generator — Annex SL Clauses 4–10 for any ISO management system standard |
| [formula](https://github.com/Formulary-Labs/formula) | compound formula | Deterministic artifact generation pipeline — SoA CSV, risk CSV, impact assessment, evidence registry, and more |

The tool names are real lab science vocabulary. `assay` is a systematic analytical test procedure. `titer` is a quantitative concentration measurement. `decay` is the gradual degradation of a substance over time. The theme is borrowed from the card game [Antidote](https://boardgamegeek.com/boardgame/145369/antidote): scientists in a lab deducing which compound is the cure before time runs out. Compliance programs are the same work.

---

## How the pieces fit

```
gemara schemas + gemara-go SDK        ← upstream data model and Go SDK
        ↓
gemara-mcp (MCP server)               ← AI agent interface to gemara schemas
Formulary tools (this org)            ← human/CI interface; produces gemara artifacts
        ↓
complytime                            ← continuous cloud-native assessment automation
```

Formulary tools sit between the gemara schema layer and complytime's continuous pipeline. They are the ad-hoc, human-facing complement to complytime's automated assessment — not a replacement for it.

**[gemara](https://github.com/gemaraproj/gemara)** provides the schema and Go SDK that every Formulary tool builds on. `substrate` wraps `gemara-go` so tools never duplicate validation logic.

**[gemara-mcp](https://github.com/gemaraproj/gemara-mcp)** is the MCP server for AI agents to interact with gemara schemas. It and Formulary tools are parallel, not sequential: one feeds AI agents context, the other produces artifacts that both AI agents and CI pipelines consume.

**[complytime](https://github.com/complytime/complytime)** is a living design document for continuous automated compliance assessment in cloud-native systems. Formulary tools are designed to produce gemara Layer 5 artifacts that complytime's pipeline can ingest.

**[Traust](https://github.com/traust-security/traust)** is a security auditing workflow engine for code, images, and packages. Formulary manages GRC compliance programs — different domain, different audience, natural bridge: Traust security findings are compliance evidence that `titer` and `specimen` can consume.

---

## AI orchestration

These tools are AI-optional at the tool level and AI-beneficial at the orchestration level.

A practitioner runs `assay` from a terminal. A CI pipeline runs `probe` in a GitHub Action. No model required, no API key, no token budget.

An AI agent calling these tools reduces token overhead significantly: instead of loading an entire framework document, product documentation corpus, and evidence history into context, the agent calls `assay` and gets back a structured gemara Layer 5 artifact. The deterministic extraction work happens outside the context window. The agent's reasoning is applied only where judgment is needed — triage, prioritization, stakeholder communication.

The reference agent orchestration layer for Formulary is **[regimen](https://github.com/Formulary-Labs/regimen)** — a principal-level compliance program management agent that governs program intake through audit closure, manages program memory across sessions, and orchestrates Formulary tools for all deterministic work. Regimen and the Formulary tools are designed to work together but neither requires the other to function.

All tools emit machine-readable output first (`--format json` default). Exit codes are meaningful (0 = clean, 1 = validation failure, 2 = tool error). No interactive prompts without `--interactive`. `--dry-run` on all write operations.

---

## Composability

```bash
# Assess a product, add high-severity gaps to risk register
assay --framework iec62443 --product docs/ | titer --severity high | specimen add

# Challenge an assessment artifact before submitting to auditors
assay --framework iso27001 --product docs/ > assessment.yaml
challenge assessment.yaml --patterns all --severity-threshold medium

# Detect drift since last quarter
decay --from 2026-Q2 --to 2026-Q3 | vital --format md > status.md

# Generate an auditor-safe exhibit from current program state
exhibit --program iso42001 --format html > exhibit.html

# Ingest post-audit feed-forward for the next cycle
specimen ingest --feed-forward post-audit/2026-feed-forward.json

# Validate a gemara artifact in CI
probe artifact.yaml && echo "valid gemara artifact"
```

---

## Status

Active development. Sprint 1 (Foundation) in progress.

See [CONTRIBUTING.md](../CONTRIBUTING.md) to contribute or add a new tool.

---

## License

Apache License 2.0. See individual tool repositories for details.
