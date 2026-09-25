# Architecture

How the Formulary ecosystem fits together — data flows, layer responsibilities, and how to add a new tool.

---

## Three-layer stack

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3 — Judgment (regimen)                                       │
│                                                                     │
│  AI compliance program management agent. Reads program memory,      │
│  decides what to do next, invokes Formulary CLIs for deterministic  │
│  work, fills [DATA NEEDED] sections with judgment and context.      │
│  Writes back to program memory. Governs intake → audit closure.     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ invokes CLIs, reads outputs
┌───────────────────────────────▼─────────────────────────────────────┐
│  Layer 2 — Deterministic execution (Formulary CLIs)                 │
│                                                                     │
│  probe · assay · titer · specimen · dose · exhibit · challenge      │
│  vital · decay · scan · compound · formula · bind                   │
│  impact · appraise · distill                                         │
│                                                                     │
│  Each tool does one thing. Reads gemara artifacts + program state.  │
│  Writes gemara artifacts or structured outputs. No inference,       │
│  no randomness, no content not traceable to the input.              │
│  Any subset can be used; no tool requires another at runtime.       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ reads/writes
┌───────────────────────────────▼─────────────────────────────────────┐
│  Layer 1 — Schema (gemara)                                          │
│                                                                     │
│  go-gemara SDK · gemara schema (CUE) · gemara-mcp (MCP server)     │
│                                                                     │
│  Defines all artifact types. Every Formulary tool imports           │
│  substrate, which wraps go-gemara. gemara-mcp gives AI agents       │
│  direct schema access in-context.                                   │
└─────────────────────────────────────────────────────────────────────┘
```

No connection between layers is mandatory. A practitioner can use any Formulary CLI directly without regimen. Regimen can execute manually using its function specs if no CLI is installed. gemara-mcp can be used independently of both.

---

## Artifact lifecycle

gemara defines five artifact layers. Formulary tools operate across all of them:

```
Layer 1 — Reference artifacts
  ControlCatalog      framework control set (input to assay, titer, challenge, bind)
  MappingDocument     cross-framework control mappings (input to bind)
  GuidanceCatalog     implementation guidance

Layer 2 — Assessment inputs
  Policy              organizational policy document
  PrincipleCatalog    design principles

Layer 3 — Assessment state  [proposed addition: CoverageMatrix from titer]
  (no stable Formulary artifact yet)

Layer 4 — Evidence
  AuditLog            audit activity log
  EnforcementLog      enforcement events

Layer 5 — Assessment results
  EvaluationLog       control assessment output (produced by assay)
  RiskCatalog         risk register entries (input to specimen catalog; source for impact)
```

### Typical data flow

```
ControlCatalog (L1) ──► assay ──► EvaluationLog (L5)
                                        │
                    ┌───────────────────┤
                    │                   │
                    ▼                   ▼
              titer (coverage)    challenge (interrogation)
                    │
                    ▼
              specimen (risks) ──► RiskCatalog (L5)
                    │
                    ▼
           dose (calendar) · exhibit (auditor view) · vital (health)
                    │
                    ▼
           formula (artifact pipeline) · compound (management system doc)
```

`probe` validates any artifact at any layer before it enters a pipeline step.

`bind` operates at Layer 1 — it reads `MappingDocument` artifacts and resolves cross-framework control equivalences without producing a new gemara layer artifact (output is JSON/MD/CSV for downstream use).

`impact` operates at Layer 1–3 — reads a `ControlCatalog` and optional `RiskCatalog`; produces `impact-assessment.csv` mapping controls to potential harms and linked risk severity. Framework identity comes from catalog metadata.

`appraise` operates across all available layers — accepts any gemara artifacts as input, auto-detects their type, and generates a framework-agnostic compliance narrative Markdown document with `[DATA NEEDED]` placeholders for sections requiring AI judgment.

`scan` and `decay` operate on program state snapshots, not gemara artifacts directly.

---

## Layer responsibilities

| Layer | Owns | Does not own |
|---|---|---|
| gemara (L1) | Artifact schema and validation | Tool behavior, workflow logic |
| Formulary CLIs (L2) | Deterministic computation, structured output | Inference, judgment, narrative prose |
| regimen (L3) | Judgment, context, program memory, [DATA NEEDED] completion | Deterministic computation (delegates to CLIs) |

The boundary is explicit and auditable: every `[DATA NEEDED: narrative]` placeholder in a Formulary output is a defined handoff from Layer 2 to Layer 3.

---

## Data stores

```
runs/[program]/latest.json      program run state — primary shared artifact
logs/provenance.jsonl           append-only provenance log (all tools write here)
data/[program]/kanban.yaml      kanban board state (regimen)
memory/[program]-memory.md      program memory (regimen)
```

Formulary CLIs read from and write to `runs/[program]/latest.json` and append to `logs/provenance.jsonl`. They do not write to regimen's memory or kanban directly.

---

## Adding a new Formulary tool

A new tool must follow all conventions in [CONTRIBUTING.md](CONTRIBUTING.md). The technical checklist:

### 1. Create the repo

```
github.com/Formulary-Labs/<toolname>
```

Go module path: `github.com/Formulary-Labs/<toolname>`

### 2. Import substrate

```go
import (
    "github.com/Formulary-Labs/substrate/exit"
    "github.com/Formulary-Labs/substrate/flags"
    "github.com/Formulary-Labs/substrate/format"
    "github.com/Formulary-Labs/substrate/provenance"
)
```

Use `flags.RegisterOn` for shared CLI flags (`--format`, `--program`, `--dry-run`). Use `exit.OK` / `exit.ValidationFailure` / `exit.ToolError` for exit codes.

### 3. Write provenance on every run

```go
provenance.Write("logs/provenance.jsonl", provenance.Entry{
    Spec:        "functions/<tool>-spec.md",
    Output:      outputPath,
    OutputType:  "other",
    Program:     programSlug,
    Purpose:     "...",
    Reusability: provenance.Instance,
    QualityGate: provenance.Pass,
    Tool:        "<toolname>",
    ToolVersion: version,
})
```

### 4. Add release infrastructure

Copy `.goreleaser.yaml` and `.github/workflows/ci.yaml` from an existing tool (e.g. `probe`). Update `project_name` and binary paths.

### 5. Add a function spec to regimen

Create `regimen/functions/<toolname>-spec.md` following the structure of an existing spec (e.g. `functions/control-coverage-spec.md`). The spec describes:
- When to invoke the tool
- Required inputs and flags
- Output format and schema
- Downstream routing (what consumes the output)
- Provenance entry format

### 6. Wire into regimen coordinator

Add a row to the Owned Specs table in `regimen/agents/coordinator.md` and, if the tool handles a portfolio-level concern, add a routing function section.

### 7. Update FORMULARY.md in regimen

Add a row to the spec → CLI map table in `regimen/FORMULARY.md` with the function spec, CLI name, repo link, what the CLI handles, and what the spec adds.

### 8. Add to org README

Add a row to the Tools table in the [org profile README](profile/README.md). Add a `go install` line to the Installation section.

### 9. Add to CATALOGS.md if the tool uses a catalog

If the tool reads a `ControlCatalog` or `MappingDocument`, add it to the tools table in [CATALOGS.md](CATALOGS.md).

### 10. Write tests

- Unit tests for the core computation package
- `TestVersionFlag` in `substrate/integration_test.go` is run against all tools — your binary will be included automatically when added to the workspace `Makefile`.

---

## Relationship to gemara-mcp

`gemara-mcp` and Formulary CLIs are parallel consumers of the gemara schema layer — not competing.

| | gemara-mcp | Formulary CLIs |
|---|---|---|
| **Callsite** | AI agents in context (MCP tools) | Terminals, CI pipelines |
| **Input** | Natural language + gemara artifacts | Structured flags + gemara artifacts |
| **Output** | In-context schema responses | Files, stdout, exit codes |
| **Token cost** | Schema loaded into agent context | Zero — computation outside context window |

In a regimen session: regimen calls `gemara-mcp` for schema questions and Formulary CLIs for computation. The two work together; neither replaces the other.

---

## Relationship to complytime

[complytime](https://github.com/complytime/complytime) is a continuous automated compliance assessment system. Formulary tools produce gemara Layer 5 artifacts that complytime's pipeline can ingest — the integration point is the shared gemara schema, not a direct dependency.

See [complytime-integration-design.md](complytime-integration-design.md) for the proposed integration design.
