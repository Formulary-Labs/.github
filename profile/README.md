# Formulary Labs

**Composable CLIs for compliance compute.**

Formulary Labs builds small, deterministic Go tools for governance and compliance
program management. OSCAL is the interchange language. Gemara remains a supported
dialect. Each tool does one job. Tools never import each other. Shared
contracts live in [`substrate`](https://github.com/Formulary-Labs/substrate).

Install only what you need. Each tool has one responsibility, shares a common
runtime contract via `substrate`, and can be versioned or swapped independently.

Orchestrators (Make pipelines, agent layers such as `regimen`) compose tools into a
run. They are never required for a tool to work alone.

## Why it exists

Compliance programs drown in prose and one-off scripts. Formulary hollows out the
non-deterministic middle: catalog validation, coverage math, plan assembly, drift
detection, and post-audit feed-forward become exit-code-clean CLIs. Judgment —
comms drafting, evidence confidence tagging, vendor scoring — stays with humans
and LLM agents, marked explicitly as `[DATA NEEDED]` / `[INFERRED]`.

## Lifecycle tools

```
extract ──► probe / distill / compound / formula ──► spectrometer ──► antidote
 (intake)              (build & assess)                  (monitor)        (close)
```

| Tool | Repo | One-liner |
|---|---|---|
| **extract** | [extract](https://github.com/Formulary-Labs/extract) | Screen a product onboarding kit against an OSCAL or gemara catalog — hard gate before program writes |
| **probe** | [probe](https://github.com/Formulary-Labs/probe) | Validate OSCAL and gemara artifacts; CI-friendly exit codes (`0` / `1` / `2`) |
| **distill** | [distill](https://github.com/Formulary-Labs/distill) | Legacy markdown → gemara `Policy` (packet docs routed, not emitted; pair with native OSCAL catalogs) |
| **compound** | [compound](https://github.com/Formulary-Labs/compound) | Annex SL security plan assembly (`--assemble-plan`) with hard packet boundary |
| **formula** | [formula](https://github.com/Formulary-Labs/formula) | Deterministic packet artifacts: SoA, risk CSVs, system card, impact |
| **assay** | [assay](https://github.com/Formulary-Labs/assay) | Resumable control assessment with 7-criterion validation |
| **challenge** | [challenge](https://github.com/Formulary-Labs/challenge) | Adversarial interrogation patterns against assessments and plans |
| **appraise** | [appraise](https://github.com/Formulary-Labs/appraise) | Framework-agnostic compliance narrative from OSCAL catalogs and gemara artifacts |
| **spectrometer** | [spectrometer](https://github.com/Formulary-Labs/spectrometer) | Cadence map, decision queue, overdue controls — model-free watch |
| **antidote** | [antidote](https://github.com/Formulary-Labs/antidote) | Post-audit feed-forward → `specimen` ingest |
| **substrate** | [substrate](https://github.com/Formulary-Labs/substrate) | Shared library: exit contract, flags, provenance, OSCAL and gemara loaders |

## Full tool table

| Tool | Description |
|---|---|
| [antidote](https://github.com/Formulary-Labs/antidote) | Post-audit feed-forward — corrective actions, lessons, specimen-ingest payload |
| [appraise](https://github.com/Formulary-Labs/appraise) | Framework-agnostic compliance narrative from OSCAL catalogs and gemara artifacts |
| [assay](https://github.com/Formulary-Labs/assay) | Control assessment engine (init → fill → validate → assemble) |
| [bind](https://github.com/Formulary-Labs/bind) | Cross-framework control mapping resolver |
| [challenge](https://github.com/Formulary-Labs/challenge) | Adversarial interrogation of compliance assessments |
| [compound](https://github.com/Formulary-Labs/compound) | Annex SL management system / security plan assembly (`--assemble-plan`) |
| [decay](https://github.com/Formulary-Labs/decay) | Longitudinal compliance drift detection |
| [distill](https://github.com/Formulary-Labs/distill) | Legacy docs → gemara Policy converter (pairs with native OSCAL catalogs) |
| [dose](https://github.com/Formulary-Labs/dose) | Evidence collection calendar (`.ics`) |
| [exhibit](https://github.com/Formulary-Labs/exhibit) | Read-only auditor HTML dashboard |
| [extract](https://github.com/Formulary-Labs/extract) | Product onboarding intake hard gate |
| [formula](https://github.com/Formulary-Labs/formula) | Deterministic SoA / risk / system-card / impact generation |
| [impact](https://github.com/Formulary-Labs/impact) | Impact assessment from a ControlCatalog |
| [probe](https://github.com/Formulary-Labs/probe) | OSCAL and gemara artifact validator |
| [scan](https://github.com/Formulary-Labs/scan) | External regulatory / threat intelligence monitoring |
| [specimen](https://github.com/Formulary-Labs/specimen) | Risk register (FAIR ALE) + feed-forward ingest |
| [spectrometer](https://github.com/Formulary-Labs/spectrometer) | Program monitoring — cadence, decisions, due dates |
| [substrate](https://github.com/Formulary-Labs/substrate) | Shared Go library |
| [titer](https://github.com/Formulary-Labs/titer) | Control coverage matrix and gap analysis |
| [vital](https://github.com/Formulary-Labs/vital) | Program health snapshot |

Artifact / catalog roles: [CATALOGS.md](CATALOGS.md).

## Installation

```sh
export GOTOOLCHAIN=auto   # modules declare go 1.26.6

go install github.com/Formulary-Labs/probe/cmd/probe@latest
go install github.com/Formulary-Labs/extract/cmd/extract@latest
go install github.com/Formulary-Labs/distill/cmd/distill@latest
go install github.com/Formulary-Labs/compound/cmd/compound@latest
go install github.com/Formulary-Labs/formula/cmd/formula@latest
go install github.com/Formulary-Labs/impact/cmd/impact@latest
go install github.com/Formulary-Labs/appraise/cmd/appraise@latest
go install github.com/Formulary-Labs/spectrometer/cmd/spectrometer@latest
go install github.com/Formulary-Labs/antidote/cmd/antidote@latest
```

Library:

```sh
go get github.com/Formulary-Labs/substrate@latest
```

Pre-built binaries (linux/darwin/windows × amd64/arm64) ship on each tool's GitHub Releases page with checksums and SPDX SBOMs.

## Quick start — security plan from OSCAL or gemara

```sh
# 1. Validate artifacts
probe --program my-isms data/oscal/*.yaml

# 2. Assemble Annex SL plan (packet materials cited, not inlined)
compound --assemble-plan \
  --program my-isms --standard iso27001 \
  --oscal data/oscal/ \
  --output out/security-plan.md \
  --report out/assembly-report.json

# 3. Interrogate structure
challenge out/security-plan.md --program my-isms --format md
```

`--gemara` is a deprecated alias for `--oscal`. Legacy markdown first? Run `distill --input docs/legacy --out data/oscal --program my-isms`, then probe.

## Design principles

1. **Single-purpose tools** — each CLI runs alone; composition is optional.
2. **Pick-and-choose** — install only the binaries your workflow needs.
3. **Determinism** — same inputs → same outputs; clock inputs are explicit (`--now`).
4. **Visible judgment boundary** — never silently invent compliance prose.
5. **Hard packet boundary** — SoA / risk / impact stay in the assessment packet; plans cite them.
6. **Shared foundation** — exit codes `0` / `1` / `2`, provenance JSONL, OSCAL and gemara loaders via `substrate`.

## Related

- Workspace monorepo overview (pipelines, Make targets): see the formulary workspace README
- Handoff contract for orchestrators: `docs/HANDOFF-CONTRACT.md` in the workspace
- Complytime integration design: [../complytime-integration-design.md](../complytime-integration-design.md)
