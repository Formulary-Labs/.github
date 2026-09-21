# Contributing to Formulary

Each Formulary tool lives in its own repository and ships as an independently distributable binary. This document covers the conventions every tool must follow. Per-tool guides live in each repository's own `CONTRIBUTING.md`.

---

## Repository structure

Each Formulary tool is an independently distributable Go binary. Tools compose at runtime via pipes and files — not at compile time. No tool imports another tool. Every tool imports `substrate`.

Each tool lives in its own repository under the [Formulary-Labs](https://github.com/Formulary-Labs) org. Separate repositories enforce the independence contract: a tool that cannot be built without another tool has violated it.

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

## Go implementation patterns

These patterns are required in all Formulary tool code. They are checked by the shared `.golangci.yml`. See [STYLE.md](STYLE.md) for output format and flag conventions.

### Error handling

Errors that reach `main` go to stderr as structured JSON, then exit with the appropriate code:

```go
if err != nil {
    fmt.Fprintf(os.Stderr, `{"error": %q, "code": 2}`+"\n", err.Error())
    os.Exit(exit.ToolError)
}
```

Within library code, always wrap with context:

```go
return nil, fmt.Errorf("loading catalog from %q: %w", path, err)
```

Never swallow errors silently. If an error is intentionally ignored, document why with a comment and suppress the lint warning with a reason:

```go
_ = enc.Encode(result) //nolint:errcheck // write to stdout; if it fails, the process dies anyway
```

### No logging framework

Do not use the `log` package, `slog`, or any third-party logger. All output follows the stdout/stderr split:

```go
// progress → stderr
fmt.Fprintf(os.Stderr, "[%s] %s\n", toolName, message)

// errors → stderr
fmt.Fprintf(os.Stderr, `{"error": %q, "code": 2}`+"\n", err.Error())

// data → stdout
enc := json.NewEncoder(os.Stdout)
enc.SetIndent("", "  ")
_ = enc.Encode(result)
```

### Exit code contract

Always use `substrate/exit` constants. Never use raw integers.

```go
import "github.com/Formulary-Labs/substrate/exit"

os.Exit(exit.OK)         // 0 — clean
os.Exit(exit.Validation) // 1 — validation failure
os.Exit(exit.ToolError)  // 2 — unexpected failure
```

### gemara artifact loading

Always load gemara artifacts through `substrate/artifact`. Never import `go-gemara` directly from a tool — the substrate shim is the insulation layer that absorbs upstream API changes.

```go
import "github.com/Formulary-Labs/substrate/artifact"

catalog, err := artifact.LoadControlCatalog(catalogPath)
if err != nil {
    fmt.Fprintf(os.Stderr, `{"error": %q, "code": 2}`+"\n", err.Error())
    os.Exit(exit.ToolError)
}
```

Loaders available in `substrate/artifact`:

| Function | Gemara layer | Type |
|---|---|---|
| `LoadControlCatalog(path)` | Layer 2 | `*ControlCatalog` |
| `LoadGuidanceCatalog(path)` | Layer 1 | `*GuidanceCatalog` |
| `LoadRiskCatalog(path)` | Layer 3 | `*RiskCatalog` |
| `LoadPolicy(path)` | Layer 3 | `*Policy` |
| `LoadEvaluationLog(path)` | Layer 5 | `*EvaluationLog` |
| `LoadAuditLog(path)` | Layer 7 | `*AuditLog` |
| `LoadMappingDocument(path)` | Layer 3 | `*MappingDocument` |
| `DetectType(path)` | — | `ArtifactType` |

### Framework identity

Never hardcode framework names in tool logic. Framework identity comes from the gemara artifact's `metadata.id` field at runtime.

```go
// Correct: read identity from the artifact
framework := catalog.Metadata.Id  // "iso42001", "iso27001", "iec62443", etc.

// Wrong: hardcode framework logic
if framework == "iso42001" {
    // framework-specific behavior
}
```

Framework-specific behavior is delivered by providing different catalog files, not by branching on framework names.

### Control lifecycle filtering

The `ControlCatalog.Control.State` field carries lifecycle state (Active, Draft, Deprecated, Retired). Tools that iterate controls must skip Deprecated and Retired entries unless the caller opts in:

```go
for _, ctrl := range catalog.Controls {
    if ctrl.State == "Deprecated" || ctrl.State == "Retired" {
        continue
    }
    // process active controls
}
```

### Comment style

Package-level doc comments follow Go standard:

```go
// Package coverage computes control coverage metrics from a gemara ControlCatalog.
package coverage
```

Section dividers inside long files use em-dash separators:

```go
// ── flag parsing ────────────────────────────────────────────────
```

Inline comments explain non-obvious behavior only. Never narrate what the code does — the code is the narration.

### `//nolint` directives

Every `//nolint` directive must include a reason:

```go
_ = enc.Encode(result) //nolint:errcheck // write to stdout; fatal if fails
```

`//nolint` without a reason is rejected by the `nolintlint` linter.

---

## Repository governance

- Issues and PRs for cross-tool concerns (substrate changes, shared convention updates) belong in this `.github` repository
- Tool-specific issues and PRs belong in the individual tool repository
- Schema or enum changes that affect gemara compatibility belong upstream in [gemaraproj/gemara](https://github.com/gemaraproj/gemara) — open an issue there first

---

## License

All Formulary tools are published under the Apache License 2.0.
