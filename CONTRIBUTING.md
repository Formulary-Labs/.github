# Contributing to Formulary

Formulary is a collection of composable Go CLI tools for compliance program management. Each tool lives in its own repository under the [Formulary-Labs](https://github.com/Formulary-Labs) org.

This document covers the shared conventions every tool must follow. Per-tool contribution guides live in each repository's own `CONTRIBUTING.md`.

---

## The Charm pattern

Each Formulary tool is an independently distributable Go binary. Tools compose at runtime via pipes and files — not at compile time. No tool imports another tool. Every tool imports `substrate`.

This mirrors how [Charm](https://github.com/charmbracelet) organizes their ecosystem: separate repositories, shared conventions, composable at the shell.

No tool is a runtime dependency of another — composition is pipes and files only. [Regimen](https://github.com/Formulary-Labs/regimen) is an optional consumer of any subset of tools; no tool requires it to operate.

---

## Shared conventions every tool must follow

### 1. Machine-readable output first

Every tool must support `--format json` as the default output mode. Human-readable output is `--format md`. Tools that produce tabular data also support `--format csv`. Tools that produce documents support `--format html`.

The contract: any tool's JSON stdout is a valid input to the next tool in a pipeline.

### 2. Meaningful exit codes

| Code | Meaning |
|---|---|
| `0` | Clean — all checks passed, output is valid |
| `1` | Validation failure — input or output did not meet schema requirements |
| `2` | Tool error — unexpected failure (missing file, network error, etc.) |

Use the exit code constants from `substrate/exit`.

### 3. Standard flags

Every tool must accept these flags, defined in `substrate/flags`:

| Flag | Type | Description |
|---|---|---|
| `--format` | string | Output format: `json` (default), `yaml`, `md`, `csv`, `html` (where applicable) |
| `--program` | string | Program slug — used for provenance logging and file resolution |
| `--dry-run` | bool | Print what would be written without writing it |
| `--interactive` | bool | Enable interactive prompts (default: off — no prompts without this flag) |
| `--quiet` | bool | Suppress progress output; only emit the final result |

### 4. No interactive prompts by default

Tools must not emit interactive prompts unless `--interactive` is explicitly set. The default behavior must be scriptable without human input. This is required for CI and agent-orchestrated usage.

### 5. Structured stderr, structured stdout

Progress output, warnings, and errors go to stderr. The tool's data output goes to stdout. This makes pipes predictable: `tool --format json | next-tool` works cleanly.

Error output must be structured JSON when `--format json` is set:
```json
{"error": "description", "code": 1, "details": {...}}
```

### 6. `--dry-run` on all write operations

Any operation that writes to disk, logs provenance, or modifies a state file must respect `--dry-run`. In dry-run mode, the tool prints what it would write to stdout without touching the filesystem.

### 7. Provenance logging

All writes must log provenance by calling `substrate/provenance.Write(...)`. This appends a structured entry to `provenance.jsonl` in the program's log directory. Never write to the provenance log directly — always use the substrate function.

The provenance entry must include: spec reference, output path, program slug, one-sentence purpose, reusability class, and quality gate result.

### 8. gemara-compatible output

Tool outputs must be valid gemara artifacts where applicable:
- Layer 2 inputs: read via `substrate/gemara.LoadLayer2(...)`
- Layer 5 outputs: write via `substrate/gemara.WriteLayer5(...)`
- Validation: run `probe` as a smoke check in CI

Never duplicate gemara validation logic — call the substrate wrapper, which calls `gemara-go` directly.

---

## Adding a new tool

New tool checklist:

- [ ] Create the repository at `github.com/Formulary-Labs/<toolname>`
- [ ] Go module path: `github.com/Formulary-Labs/<toolname>`
- [ ] Import `github.com/Formulary-Labs/substrate` — do not re-implement shared logic
- [ ] Implement all standard flags from `substrate/flags`
- [ ] Exit codes from `substrate/exit`
- [ ] Provenance logging via `substrate/provenance`
- [ ] `--format json` as default output
- [ ] `--dry-run` on all write operations
- [ ] No interactive prompts without `--interactive`
- [ ] `go test ./...` passes
- [ ] goreleaser config for cross-platform release (`linux/amd64`, `darwin/arm64`, `darwin/amd64`, `windows/amd64`)
- [ ] CI: lint, test, goreleaser dry-run on PR
- [ ] README: one-paragraph description, installation, usage examples including pipe usage, gemara artifact reference
- [ ] `probe` validation step in CI smoke test

Before submitting the first PR, open an issue in this `.github` repository describing the tool's purpose, the compliance function or workflow it addresses, and why it belongs in Formulary rather than the AI agent layer. This keeps the ecosystem focused on deterministic work.

---

## What belongs in Formulary vs. the agent layer

Formulary tools handle deterministic work: extraction, mapping, measurement, scheduling, and document generation from structured inputs.

The AI agent layer handles judgment work: reasoning about heterogeneous raw materials, calibrating tone, prioritizing across competing concerns, drafting stakeholder communications.

The reference implementation of the agent layer is [regimen](https://github.com/Formulary-Labs/regimen) — the principal-level compliance program management agent that orchestrates Formulary tools and governs everything that requires judgment.

If implementing a tool requires an LLM to produce correct output, it belongs in the agent layer — not here.

When in doubt: can you write a unit test that deterministically verifies the output for a given input? If yes, it belongs in Formulary. If no, it belongs in the agent layer (regimen).

---

## Deferred tools

Two tools are architecturally planned but not yet built due to limited production grounding:

- **bind** — cross-framework control mapping (deferred: limited multi-framework program usage)
- **transfer** — OSCAL ↔ gemara interop bridge (deferred: no OSCAL consumers yet)

If you have a production use case for either, open an issue.

---

## Repository governance

- Issues and PRs for cross-tool concerns (substrate changes, shared convention updates) belong in this `.github` repository
- Tool-specific issues and PRs belong in the individual tool repository
- Schema or enum changes that affect gemara compatibility belong upstream in [gemaraproj/gemara](https://github.com/gemaraproj/gemara) — open an issue there first

---

## License

All Formulary tools are published under the Apache License 2.0.
