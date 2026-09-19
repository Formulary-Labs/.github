# Formulary

Composable open-source compliance tools.

Each tool does one thing. Every tool reads and writes [gemara](https://github.com/gemaraproj/gemara)-compatible artifacts — any tool's output is valid input to the next. No tool requires another to operate at runtime. Composition is pipes and files.

One tool standalone, a few piped together, or the full suite under [regimen](https://github.com/Formulary-Labs/regimen). Any subset works.

## Why

Manual compliance work does not scale. Running multiple certifications simultaneously means doing the same extraction, mapping, and documentation work across each one — most of it deterministic.

Formulary separates the deterministic work (citation extraction, coverage math, evidence scheduling, drift detection, document assembly) from the judgment work (prioritization, communication, stakeholder management). The deterministic work runs in a terminal or a CI pipeline. The judgment work is where human and AI attention belongs.

## Tools

| Tool | What it does |
|---|---|
| [substrate](https://github.com/Formulary-Labs/substrate) | Core library — gemara SDK wrappers, shared CLI conventions, provenance writer |
| [probe](https://github.com/Formulary-Labs/probe) | Validate a gemara artifact — structured results and clean exit codes for CI |
| [assay](https://github.com/Formulary-Labs/assay) | Control assessment engine — fill templates from product docs, produce gemara Layer 5 artifacts, resumable by checkpoint |
| [titer](https://github.com/Formulary-Labs/titer) | Control coverage matrix — coverage % by family, owner gaps, evidence gaps |
| [specimen](https://github.com/Formulary-Labs/specimen) | Risk register and POA&M — stable IDs, severity matrix, post-audit feed-forward ingest |
| [dose](https://github.com/Formulary-Labs/dose) | Evidence collection calendar — RFC 5545 `.ics` and Markdown, shift-left scheduled to working days |
| [exhibit](https://github.com/Formulary-Labs/exhibit) | Auditor compliance posture view — static HTML, no JavaScript, no internal-only data |
| [challenge](https://github.com/Formulary-Labs/challenge) | Adversarial artifact interrogation — 10 deterministic patterns for audit readiness review |
| [vital](https://github.com/Formulary-Labs/vital) | Program health snapshot — traffic-light ratings across coverage, risks, evidence, decisions |
| [decay](https://github.com/Formulary-Labs/decay) | Longitudinal compliance drift detection — 13 named patterns across two audit cycles |
| [scan](https://github.com/Formulary-Labs/scan) | External regulatory and threat monitoring — 5 source categories, structured relevance scoring |
| [compound](https://github.com/Formulary-Labs/compound) | Management system document generator — Annex SL Clauses 4–10 for ISO 27001, ISO 42001, IEC 62443 |
| [formula](https://github.com/Formulary-Labs/formula) | Deterministic artifact generation — SOA CSV, risk CSV, evidence registry, system card, and more |
| [regimen](https://github.com/Formulary-Labs/regimen) | Compliance program management agent — orchestrates Formulary tools, manages program memory across sessions, governs intake through audit closure |

## How the pieces fit

```
                gemara schemas + go-gemara SDK
                         │
               ┌─────────┴──────────┐
         gemara-mcp               Formulary CLIs
    (optional MCP server)    (any subset, standalone)
               │                    │
               └─────── regimen ────┘   ← optional agent layer
                            │
                       complytime         ← optional consumer
```

No connection in this diagram is mandatory. Each layer is independently useful.

**[gemara](https://github.com/gemaraproj/gemara)** provides the schema and Go SDK that every Formulary tool builds on. `substrate` wraps `go-gemara` so tools share validation logic without duplicating it.

**[gemara-mcp](https://github.com/gemaraproj/gemara-mcp)** is the MCP server for AI agents interacting with gemara schemas. It and Formulary tools are parallel consumers of the same schema layer: `gemara-mcp` feeds AI agents context; Formulary tools produce artifacts that both AI agents and CI pipelines consume.

**[complytime](https://github.com/complytime/complytime)** is a continuous automated compliance assessment system for cloud-native environments. Formulary tools produce gemara Layer 5 artifacts that complytime's pipeline can ingest.

## AI orchestration

These tools are AI-optional at the tool level and AI-beneficial at the orchestration level.

A practitioner runs `assay` from a terminal. A CI pipeline runs `probe` in a GitHub Action. No model required, no API key, no token budget.

An AI agent calling these tools reduces token overhead: instead of loading an entire framework document, product documentation corpus, and evidence history into context, the agent calls `assay` and receives a structured gemara Layer 5 artifact. Deterministic extraction happens outside the context window. The agent's reasoning applies only where judgment is needed.

Formulary tools work without regimen. Regimen works without any given Formulary CLI — its function specs contain enough guidance for an agent to execute manually when a CLI is not installed. All tools emit machine-readable output first (`--format json` default). Exit codes are meaningful: 0 = clean, 1 = validation failure, 2 = tool error.

## Composability

```bash
# Assess a product, add high-severity gaps to the risk register
assay --framework iec62443 --catalog catalog.yaml --product-source docs/ \
  | titer --severity high \
  | specimen add

# Interrogate an assessment artifact before submitting to auditors
assay --framework iso27001 --catalog catalog.yaml --product-source docs/ > assessment.json
challenge assessment.json --severity-threshold medium

# Detect drift since last quarter
decay --from snapshots/2026-Q2.json --to snapshots/2026-Q3.json \
  | vital --format md > health.md

# Build an auditor-ready exhibit from current program state
titer --catalog catalog.yaml --soa soa.csv > coverage.json
exhibit --program iso42001 --coverage coverage.json --risks risks.json \
        --evidence evidence.json --provenance logs/provenance.jsonl \
        > dashboard.html

# Ingest post-audit findings for the next cycle
specimen ingest --feed-forward post-audit/2026-feed-forward.json

# Validate a gemara artifact in CI
probe catalog.yaml && echo "valid"
```

## Status

Active development. Sprint 1 (Foundation) in progress.

See [CONTRIBUTING.md](../CONTRIBUTING.md) to contribute or propose a new tool.

## License

Apache License 2.0. See individual tool repositories for details.
