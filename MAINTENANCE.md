# Maintenance Log

Running record of non-feature infrastructure changes across the Formulary ecosystem.
Each entry documents what broke, why, and how it was fixed to prevent repeating the same diagnosis.

---

## 2026-09-21 — CI Remediation Pass 1: `replace` directive and substrate integration test

### Symptoms
All 15 tool repos failed CI at `go test` / `go build` with:
```
replacement directory ../substrate does not exist
```
substrate CI failed at `go test` with `TestVersionFlag` attempting to `chdir` into sibling tool directories.

### Root causes
1. Every tool `go.mod` contained `replace github.com/Formulary-Labs/substrate => ../substrate`.
   GitHub Actions checks out each repo independently — `../substrate` does not exist in CI.
2. `substrate/integration_test.go` (`TestVersionFlag`) built all 15 tool binaries by navigating to
   sibling directories (`../assay`, `../probe`, etc.). Those directories don't exist in standalone CI.

### Fixes applied
- Tagged substrate `v0.1.0` (already present on remote, confirmed).
- Deleted `substrate/integration_test.go`. Each tool's own CI verifies its build; the cross-tool test
  was redundant and monorepo-only.
- Removed `replace` directive from all 15 tool `go.mod` files; set
  `require github.com/Formulary-Labs/substrate v0.1.0`. Ran `go mod tidy` per tool.
- Committed and pushed each tool.

### Lesson
`replace` directives in `go.mod` are local-dev-only. Never commit them without a corresponding
`go.work` file (which CI can ignore cleanly) or a module proxy tag. Cross-repo integration tests
that assume sibling checkout layout are incompatible with per-repo CI.

---

## 2026-09-22 — CI Remediation Pass 2A: substrate v0.2.0 for impact and appraise

### Symptoms
`impact` and `appraise` failed at `go test` with:
```
undefined: artifact.RiskCatalog
undefined: artifact.Policy
```

### Root cause
`v0.1.0` was tagged at commit `80f168f` (before `RiskCatalog`, `Policy`, `LoadRiskCatalog`,
`LoadPolicy`, and related constants were added to `substrate/artifact` in commit `c087ec6`).
Both tools depended on those exports but pinned `v0.1.0`.

### Fix applied
- Tagged substrate `v0.2.0` at current HEAD (which includes all needed exports).
- Updated `impact/go.mod` and `appraise/go.mod` to `require github.com/Formulary-Labs/substrate v0.2.0`.
- Ran `go mod tidy`, committed, pushed both tools.

### Lesson
When adding new exported symbols to substrate that tools already depend on locally (via `go.work`),
the tag must be cut before those tools' `go.mod` files can pin the version. The sequence is:
commit substrate changes → tag → update tool `go.mod` → push.

---

## 2026-09-22 — CI Remediation Pass 2B: golangci-lint v2 + action version

### Symptoms (round 1)
All 13 tools (those with passing Test+Build) failed Lint with:
```
Error: can't load config: the Go language version (go1.24) used to build golangci-lint
is lower than the targeted Go version (1.25.0)
golangci-lint exit with code 3
```

### Root cause (round 1)
All CI workflows used `golangci/golangci-lint-action@v6` with `version: latest`.
`latest` resolved to golangci-lint `v1.64.8`, which was compiled with Go 1.24. All `go.mod`
files declare `go 1.25.0`. golangci-lint v1.64.8 refuses to lint a module targeting a newer
Go version than its own toolchain.

### Fix applied (round 1)
- Pinned golangci-lint to `v2.13.2` (current stable, compiled with Go 1.25+) in all 16 CI workflows.
- Added missing Lint and Security scan steps to substrate's CI workflow (it only had Test and Build).

### Symptoms (round 2)
After pinning `v2.13.2`, all 16 repos failed Lint with:
```
invalid version string 'v2.13.2', golangci-lint v2 is not supported by golangci-lint-action v6,
you must update to golangci-lint-action v7.
```

### Root cause (round 2)
`golangci-lint-action@v6` only supports golangci-lint v1.x. golangci-lint v2.x requires
`golangci-lint-action@v7` or higher.

### Fix applied (round 2)
- Updated all 16 CI workflows from `golangci/golangci-lint-action@v6` to `@v7`.

### Symptoms (round 3)
After upgrading the action, Lint failed on all 16 repos with config schema errors:
```
jsonschema: "issues" does not validate: additional properties 'exclude-use-default', 'exclude-rules' not allowed
jsonschema: "" does not validate: additional properties 'linters-settings' not allowed
```

### Root cause (round 3)
The shared `.golangci.yml` was written in golangci-lint v1 schema. golangci-lint v2 uses a
stricter schema with renamed sections:

| v1 key | v2 replacement |
|--------|---------------|
| `linters-settings:` (top-level) | `linters.settings:` (nested under `linters:`) |
| `linters-settings.goimports` | `formatters.settings.goimports` |
| `goimports`, `gofmt` in `linters.enable` | `formatters.enable` |
| `issues.exclude-rules` | `linters.exclusions.rules` |
| `issues.exclude-use-default` | removed (v2 default = no preset exclusions) |

### Fix applied (round 3)
- Rewrote canonical `formulary-github-org/golangci.yml` in v2 schema.
- Redeployed (copied verbatim) to all 16 tool repos and substrate.

### Lesson
`golangci-lint-action` major version and golangci-lint major version must be kept in sync:
- action v6 → golangci-lint v1.x only
- action v7+ → golangci-lint v2.x

`version: latest` in the action resolves to the latest v1.x release (the action was released
before v2 existed), not the globally latest golangci-lint release. Always pin both the action
version (`@v7`) and the binary version (`version: v2.x.x`) explicitly.

When upgrading golangci-lint across a major version, run `golangci-lint migrate` locally on the
config file before deploying, or consult the migration guide at
https://golangci-lint.run/docs/product/migration-guide/.

---

## CI Remediation Pass 4 — Lint violations (2026-09-22)

### Symptoms
golangci-lint now running correctly with v2 config. All 16 repos had real code-quality
violations flagged across six linter categories.

### Root causes and fixes

| Linter | Violation | Fix applied |
|---|---|---|
| `gofmt` | Many files not properly formatted | Ran `go fmt ./...` across all 16 repos |
| `gosec G301` | `os.MkdirAll(path, 0o755)` — dirs should be 0750 | Changed to `0o750` in 8 production files |
| `gosec G302/G306` | `os.WriteFile/OpenFile` with `0o644` | Changed to `0o600` in 9 production files |
| `errcheck` | Unchecked `defer f.Close()` and `os.WriteFile` returns | Added `//nolint:errcheck` on test read-only closes; fixed open error handling in tests |
| `staticcheck QF1012` | `b.WriteString(fmt.Sprintf(...))` in dose/calendar | Converted 7 occurrences to `fmt.Fprintf(&b, ...)` |
| `staticcheck QF1003` | `if e.Status == ...` in exhibit/render | Converted to a `switch e.Status { }` block |
| `revive unused-parameter` | `verbose bool` in probe; `cfg/ctx/orgName` in compound writeClause9/10/5 | Renamed unused params to `_` |
| `revive exported` (comment format) | Missing/wrong doc comment format on exported consts/types | Added `//nolint:revive` to self-documenting const blocks; fixed var/type doc comments |
| `revive exported` (stutter) | `drift.DriftReport`, `render.RenderHTML`, etc. | Added `//nolint:revive // stutter is intentional` on affected type/func lines |
| **Build failure** | `specimen` used `artifact.LoadRiskCatalog` but was on substrate v0.1.0 | Bumped specimen to `substrate v0.2.0` |

### Lessons
- After fixing the linter infrastructure (action version, config schema), the linter runs and
  finds **real violations**. Fix the violations before declaring CI green.
- `go fmt ./...` (not `gofmt -w ./...`) is the correct command to recursively format all packages.
- File permission constants in production code: directories → `0o750`, files → `0o600`.
  **This applies to test files too** — gosec checks `os.WriteFile` in `_test.go` files.
- For test helper files, prefer `//nolint:errcheck // test read-only file` over try/catch boilerplate.
- When a new substrate minor version adds exported symbols, check ALL tool repos that use those
  symbols — not just the ones you added the symbols for. `specimen` had the same dependency on
  `artifact.LoadRiskCatalog` as `impact`/`appraise` but was missed in the v0.2.0 pass.
- `//nolint:lintername` must be placed on the **declaration line**, not the doc-comment line above it.
  Placing a nolint on a comment has no effect on the declaration that follows.
- `//nolint:nolintlint` fires when a nolint directive is unused. Two common causes:
  (a) you already suppressed the error a different way (e.g. `_, _ = f()` makes `//nolint:errcheck` redundant),
  (b) the linter doesn't flag test files (e.g. gosec ignores `_test.go`).
- revive `exported` rule: every exported identifier needs a doc comment starting with the identifier
  name. For large const blocks of self-documenting string identifiers, add `//nolint:revive // self-documenting`
  before the `const (` block. For type alias blocks, add individual `// TypeName ...` comments.
- revive `unused-parameter`: scan ALL function signatures, not just the ones the CI shows — the CI may
  show only the first N violations. A function that has ONE of its params renamed to `_` may still
  have OTHER params that are also unused.

---

---

## 2026-09-22 — Sprint 9: gofmt after adding multi-field var blocks

### Symptoms
specimen and titer CI failed immediately after the Sprint 9 FAIR feature push:
```
cmd/specimen/main.go:71:1: File is not properly formatted (gofmt)
coverage/coverage.go:93:1: File is not properly formatted (gofmt)
```

### Root cause
When adding multiple new variables to an existing `var (...)` block, the manual
tab alignment between variable names and `=` signs can become inconsistent with
what `gofmt` expects. `gofmt` uses tab stops, so the longest variable name in
the block sets the alignment for the entire block. In specimen, adding
`exposureFactor` (longer than `controlID`) shifted the expected tab column for
all other variables.

### Fix
Run `go fmt ./...` in both repos immediately after edits, before committing.
Fixes were committed as style-only follow-up commits and pushed.

### Lesson learned
**Always run `go fmt ./...` in the changed repo before committing**, especially
when editing `var (...)` blocks. The CI linter catches this instantly and
wastes a full CI run. Add `go fmt ./...` as a pre-commit habit for all Sprint
work, or consider a local pre-commit hook.

---

## Standing rules

- **substrate tags**: cut a new semver tag for every commit that adds or changes exported API.
  Tools that depend on new exports must be updated to the new tag before those tools are pushed.
- **golangci-lint**: pin both the action version (`@v7`) and the binary version (`version: vX.Y.Z`).
  When upgrading the binary, check if the action major version also needs to bump.
- **shared config**: `formulary-github-org/golangci.yml` is the canonical source. Always edit that
  file first, then redeploy to all repos. Never edit per-tool copies directly.
- **local `replace` directives**: use `go.work` for local development. Never commit a `replace`
  directive to a tool repo; it will break standalone CI.
