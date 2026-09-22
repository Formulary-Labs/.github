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

## Standing rules

- **substrate tags**: cut a new semver tag for every commit that adds or changes exported API.
  Tools that depend on new exports must be updated to the new tag before those tools are pushed.
- **golangci-lint**: pin both the action version (`@v7`) and the binary version (`version: vX.Y.Z`).
  When upgrading the binary, check if the action major version also needs to bump.
- **shared config**: `formulary-github-org/golangci.yml` is the canonical source. Always edit that
  file first, then redeploy to all repos. Never edit per-tool copies directly.
- **local `replace` directives**: use `go.work` for local development. Never commit a `replace`
  directive to a tool repo; it will break standalone CI.
