# Formulary Style Guide

One set of rules. Every tool follows them. No exceptions that don't have a documented reason.

---

## Vocabulary

Formulary tool names come from a shared register: laboratory procedures, chemical operations, and clinical measurement. Every name in this register is short, concrete, and unambiguous about what the tool does. The name is the interface.

Current register: `assay`, `bind`, `challenge`, `compound`, `decay`, `dose`, `exhibit`, `formula`, `probe`, `scan`, `specimen`, `titer`, `vital`.

New tools must fit this register. If a proposed name requires explanation to fit, the name is wrong. Generic names (`run`, `check`, `generate`, `process`) are never acceptable.

---

## Output Conventions

### Format hierarchy

| Format | Flag | Use case |
|---|---|---|
| JSON | `--format json` (default) | Machine consumption, pipelines, agent input |
| Markdown | `--format md` | Human review, CI logs, terminal inspection |
| CSV | `--format csv` | Spreadsheet handoff, audit packages |
| HTML | `--format html` | Rendered dashboard, browser delivery |

Default is always JSON. If a tool supports only one format, that format is JSON.

The contract: any tool's JSON stdout is a valid input to the next tool in a pipeline.

### stdout / stderr split

Primary output (data) → stdout. Everything else → stderr.

```
stdout:  {"controls": [...]}          ← the tool's result
stderr:  [titer] computing 47 controls  ← progress note (stripped by pipes)
stderr:  {"error": "...", "code": 2}  ← tool error
```

This split is non-negotiable. It is what makes pipes work.

### JSON output

Two spaces of indentation. `encoding/json.Encoder` with `SetIndent("", "  ")`. Keys are `snake_case`. No trailing newlines after the closing brace (the encoder adds one automatically — don't add another).

```go
enc := json.NewEncoder(os.Stdout)
enc.SetIndent("", "  ")
_ = enc.Encode(result)
```

### Markdown output

Use `✓` and `✗` only. No other Unicode symbols in structured output. No ANSI escape codes. No emoji. Tables must have a header row and a separator row.

```
| Control | Status | Owner |
|---|---|---|
| AC-1 | ✓ | sec-team |
| AC-2 | ✗ | [OWNER NEEDED] |
```

### Progress messages

One line per meaningful step. Prefix with the tool name in brackets. Lowercase. No trailing punctuation.

```
[titer] loading catalog: iso42001-controls.yaml
[titer] computing coverage for 93 controls
[titer] coverage: 71% (66 evidenced, 27 gap)
```

No spinners, no progress bars, no color. A clean log is more useful than a decorated one.

### Error messages

Structured JSON to stderr, always:

```
{"error": "catalog not found: iso42001-controls.yaml", "code": 2}
```

The `code` field matches the exit code. `1` for validation failures, `2` for tool errors.

No exclamation marks. No apologies. No "Oops" or "Uh oh." State the problem and the value that caused it.

---

## Exit Codes

Three values. No others.

| Code | Constant | Meaning |
|---|---|---|
| `0` | `exit.OK` | Clean run — output is valid |
| `1` | `exit.Validation` | Input or output failed validation |
| `2` | `exit.ToolError` | Unexpected failure — missing file, parse error, etc. |

Always use the constants from `substrate/exit`. Never use raw integers.

Exit codes are the interface for CI and agent orchestration. `probe artifact.yaml && proceed` must work without reading output.

---

## Flag Conventions

Use the stdlib `flag` package. No third-party flag libraries.

`--kebab-case` for all flags. No single-character short flags in main commands.

Declare flags in a grouped `var (...)` block at the top of `main()`:

```go
var (
    catalogFlag = flag.String("catalog", "", "Path to gemara ControlCatalog YAML")
    formatFlag  = flag.String("format", "json", "Output format: json, md, csv")
    quietFlag   = flag.Bool("quiet", false, "Suppress progress output")
    versionFlag = flag.Bool("version", false, "Print version and exit")
)
```

### Canonical flags

These flags mean the same thing in every tool that uses them. Use the exact same descriptions.

| Flag | Type | Default | Description |
|---|---|---|---|
| `--catalog` | string | `""` | Path to gemara ControlCatalog YAML |
| `--format` | string | `"json"` | Output format: json, md, csv, html (where applicable) |
| `--out` | string | `""` | Write output to file instead of stdout |
| `--program` | string | `""` | Program slug for provenance logging |
| `--quiet` | bool | `false` | Suppress progress output; emit result only |
| `--dry-run` | bool | `false` | Print what would be written without writing it |
| `--version` | bool | `false` | Print version and exit |

---

## The `[DATA NEEDED]` Contract

`[DATA NEEDED: <type>]` is a defined output — not a bug and not a gap. It marks the boundary where deterministic generation stops and judgment begins.

```
[DATA NEEDED: narrative]   ← prose that requires reasoning about context
[DATA NEEDED: owner]       ← requires a human decision
[DATA NEEDED: date]        ← a deadline that must be set by the program
```

The agent layer (regimen) fills these placeholders. The CLI tool makes its scope explicit so the division of labor is auditable. Every `[DATA NEEDED]` in a tool's output is a defined handoff point, not a failure.

Never suppress a `[DATA NEEDED]`. Never fill it with a guess. The placeholder is the correct output.

---

## README Structure

Every tool README follows this order. No sections before the name and one-liner. No motivation paragraphs.

```markdown
# <tool>

One sentence. What it takes. What it produces. No adjectives.

```bash
go install github.com/Formulary-Labs/<tool>/cmd/<tool>@latest
```

## What it does

Two to four sentences. Implementation, not marketing. What the tool does with its input. Where the output goes.

## Usage

```bash
<tool> [flags] <input>
```

### Flags

| Flag | Default | Description |
|---|---|---|

### Examples

Concrete commands. Show the pipe pattern if the tool composes.

## Outputs

| Artifact | Filename | Format | Contents |
|---|---|---|---|

## Related tools

One-liner for each tool this one composes with.

## License

Apache License 2.0
```

Show before tell. If an example is clearer than a sentence, use the example.

---

## Writing Tone

The same register throughout: tools, docs, specs, and comments.

- Fragments are acceptable. "Pass it a catalog. Get a coverage matrix." is correct.
- State what the tool does, not why compliance matters. The reader already knows.
- No filler transitions: "Additionally," "Furthermore," "It is worth noting that" — cut them.
- Precise terms, used consistently. A "control" is a control everywhere. Do not rotate synonyms (safeguard, measure, mechanism) for variety.
- Active voice. "titer computes coverage" not "coverage is computed by titer."
- Numbers over approximations. "93 controls" not "roughly a hundred."

For compliance document output (anything a tool generates that goes to an auditor), the language standard is `regimen/engine/doc-style-guide.md`. Run the doc-style-guide checker on any narrative output before shipping.

---

## What Belongs in Formulary

Formulary tools handle deterministic work: reading structured inputs, applying a defined function, producing a structured output. If you can write a unit test that verifies the output for a given input without an LLM, it belongs in Formulary.

If the work requires judgment — reasoning across heterogeneous inputs, calibrating tone, prioritizing across competing concerns — it belongs in the agent layer (regimen).

Each tool is a precise instrument. Together they compound.
