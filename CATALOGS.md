# Catalogs

Several Formulary tools require gemara `ControlCatalog` or `MappingDocument` YAML files as inputs. This document lists known upstream sources.

---

## What is a gemara ControlCatalog?

A `ControlCatalog` is a gemara Layer 1 artifact that represents a compliance framework's control set in a structured, machine-readable format. Tools like [`titer`](https://github.com/Formulary-Labs/titer), [`assay`](https://github.com/Formulary-Labs/assay), [`challenge`](https://github.com/Formulary-Labs/challenge), and [`bind`](https://github.com/Formulary-Labs/bind) use it to resolve control IDs, titles, families, and metadata.

A `MappingDocument` is a gemara Layer 1 artifact that maps controls from one framework to another. [`bind`](https://github.com/Formulary-Labs/bind) uses it to resolve cross-framework control equivalences.

Validate any catalog or mapping document before use:

```bash
probe catalog.yaml
probe mapping.yaml
```

---

## Known catalog sources

### FINOS Common Cloud Controls (CCC)

The FINOS Common Cloud Controls project publishes control catalogs in gemara-compatible format. These are community-maintained and cover cloud security controls aligned with major frameworks.

- **Repository:** [github.com/finos/common-cloud-controls](https://github.com/finos/common-cloud-controls)
- **Artifact type:** `ControlCatalog`
- **Coverage:** Cloud security controls, cross-framework mappings

### gemara project upstream catalogs

The gemara project maintains example catalogs and test fixtures as part of the `go-gemara` SDK:

- **Repository:** [github.com/gemaraproj/go-gemara](https://github.com/gemaraproj/go-gemara)
- **Artifact type:** `ControlCatalog`, `MappingDocument`
- **Coverage:** Schema-valid examples for testing and tooling integration

---

## Authoring your own catalog

If no upstream catalog exists for your framework, you can author one. The gemara schema defines the `ControlCatalog` type — use the [`go-gemara`](https://github.com/gemaraproj/go-gemara) SDK or [`gemara-mcp`](https://github.com/gemaraproj/gemara-mcp) to validate the output.

Use [`probe`](https://github.com/Formulary-Labs/probe) to validate before use:

```bash
probe my-catalog.yaml
```

A minimal `ControlCatalog` YAML skeleton:

```yaml
metadata:
  type: ControlCatalog
  id: my-framework
  title: My Framework Control Catalog
  version: "1.0"
controls:
  - id: CTRL-001
    title: Access Control Policy
    family: Access Control
    description: Establish and maintain access control policies.
```

---

## Tools that require a catalog

| Tool | Catalog flag | Notes |
|------|-------------|-------|
| [`assay`](https://github.com/Formulary-Labs/assay) | `--catalog` | Source catalog for assessment |
| [`titer`](https://github.com/Formulary-Labs/titer) | `--catalog` | Coverage computation |
| [`challenge`](https://github.com/Formulary-Labs/challenge) | `--catalog` | Inheritance validation in interrogation |
| [`bind`](https://github.com/Formulary-Labs/bind) | `--source-catalog`, `--target-catalog` | Title resolution and source-ID validation |

## Tools that require a MappingDocument

| Tool | Flag | Notes |
|------|------|-------|
| [`bind`](https://github.com/Formulary-Labs/bind) | `--document` | Cross-framework control mapping resolution |

---

## Contributing a catalog

If you have authored a gemara `ControlCatalog` or `MappingDocument` for a publicly available framework and would like it listed here, open a pull request against this file with:

- A link to the source repository
- The framework(s) covered
- The artifact type (`ControlCatalog`, `MappingDocument`, or both)
- Confirmation that it passes `probe` validation

See [CONTRIBUTING.md](CONTRIBUTING.md) for the contribution process.
