# Formulary — Proposed Complytime Integration Design

**Status:** v0.1.0 — ready for submission  
**Date:** 2026-09-21  
**From:** Formulary-Labs (github.com/Formulary-Labs)

---

## Summary

[Formulary](https://github.com/Formulary-Labs) is a collection of composable Go CLI micro-tools for compliance program management built on the [gemara](https://github.com/gemaraproj/gemara) schema. This document proposes integration points between Formulary tools and complytime's continuous assessment pipeline.

Formulary and complytime are complementary, not competing. Formulary handles human-facing ad-hoc workflows and audit prep; complytime handles continuous automated assessment. The overlap is the gemara Layer 5 artifact format both produce and consume.

---

## Formulary's Relationship to Complytime's Problem Documents

The [complytime problem docs](https://github.com/complytime/complytime/tree/main/docs/problems) identify four core problems. Formulary addresses three of them directly.

### Requirement Fidelity Problem

**Complytime's problem:** Continuous assessment tools often degrade requirement fidelity by over-simplifying framework requirements into binary pass/fail checks that lose the nuance of the original requirement language.

**Formulary's contribution:** `assay` addresses this by treating control assessment as a structured analytical procedure (the "assay" metaphor is literal: a systematic test with defined criteria). The tool's 7-criterion batch validation enforces:
- Citation quality classification (direct assertion vs. topical reference vs. adjacent capability vs. not found)
- Satisfaction determination with explicit evidence traceability
- Rejection of restatements as implementations (see `challenge`'s Restatement Test)

The output is a gemara Layer 5 artifact with full requirement fidelity. This artifact can feed complytime's continuous pipeline as a validated baseline.

**Proposed integration:** `assay --output-format gemara-layer5` → complytime ingest

### Evaluator Coupling Problem

**Complytime's problem:** Assessment tools are tightly coupled to specific evaluators (humans, specific LLMs), making results non-reproducible and non-comparable.

**Formulary's contribution:** Formulary's approach is AI-optional and deterministic. The deterministic layers of `assay` (citation extraction, schema validation, quality gate) produce the same output regardless of evaluator. The narrative generation layer (which requires judgment) is explicitly separated — it is not part of the tool and is flagged as `[DATA NEEDED: narrative]` in the output.

**Proposed integration:** Submit `assay`'s 7-criterion validation schema as a candidate for complytime's `Evaluator Protocol` specification.

### Cross-Framework Mapping Problem

**Complytime's problem:** Organizations operating under multiple frameworks must manually duplicate assessment work across frameworks.

**Formulary's primary contribution:** `bind` reads a gemara `MappingDocument` (Layer 1 artifact) and resolves cross-framework control mappings deterministically. Given a `MappingDocument` for ISO 27001 → NIST SP 800-53 and the corresponding `ControlCatalog` files, `bind` produces a structured mapping table — which source controls correspond to which target controls, which are unmapped (no-match relationship), and which source IDs are absent from the catalog (data quality flags).

This is a direct answer to the cross-framework duplication problem: instead of re-assessing against every framework independently, organizations maintain a gemara `MappingDocument` and use `bind` to derive coverage overlap from existing assessments.

`titer` addresses a complementary angle: it computes coverage matrices from a single framework's control catalog. For single-framework coverage analysis, `titer` is the right tool. For cross-framework equivalence resolution, `bind` is more directly relevant.

**Proposed integration:** 
- `bind --document mapping.yaml --format json` → complytime ingest for cross-framework control equivalences
- Unmapped controls from `bind --filter unmapped` feed into complytime as coverage gaps requiring independent assessment

**Proposed schema contribution:** The gemara `MappingDocument` artifact type (already in the gemara schema) is the natural schema for cross-framework control mappings in complytime. We propose complytime adopt `MappingDocument` as the standard input format for cross-framework mapping workflows rather than developing a parallel format.

### Evidence Integration Problem

**Complytime's problem:** Evidence collection is disconnected from the continuous assessment pipeline — evidence exists in scattered locations and is not systematically linked to control assessments.

**Formulary's contribution:** `dose` (evidence calendar) and `specimen` (risk/POA&M register) address the evidence management layer:
- `dose` produces shift-left-scheduled RFC 5545 iCalendar events and markdown event lists, ensuring evidence collection is planned before deadlines
- `specimen` tracks risk status and links audit findings to corrective actions through the `ingest` subcommand

**Proposed integration:** `dose` output format extended to emit gemara-compatible evidence attestation records that complytime's pipeline can consume as evidence provenance.

---

## Artifact Compatibility

All Formulary tools produce gemara-compatible artifacts where applicable:

| Tool | gemara Layer | Artifact |
|---|---|---|
| `probe` | validation | validates Layer 2 and Layer 5 artifacts |
| `assay` | Layer 5 | assessment result artifact |
| `titer` | reads Layer 2 | produces coverage JSON (candidate for gemara Layer 3) |
| `specimen` | reads/writes | risk catalog entries (compatible with gemara RiskCatalog) |
| `formula` | produces | SOA CSV, risk CSV — candidates for gemara schema additions |

---

## Proposed Schema Contributions to gemara

Based on Formulary's production usage, we propose the following additions to the gemara schema:

1. **Layer 3: Coverage Matrix** — a schema for control coverage snapshots produced by `titer`. Structure: `{framework, timestamp, families: [{id, total, evidenced, gap, owner_gaps}]}`.

2. **Layer 5 Extensions: Citation Quality** — adding `citation_quality` to assessment requirement entries (direct_assertion | topical_reference | adjacent_capability | architectural_description | citation_not_found). This is already in `assay`'s state model and is a direct improvement to requirement fidelity.

3. **Post-Audit Feed-Forward** — a schema for audit-to-improvement loop artifacts consumed by `specimen ingest`. Currently an informal JSON format; should be standardized.

---

## Proposed Contribution Timeline

| Milestone | Target | Artifact |
|---|---|---|
| Formulary v0.1.0 (all 13 tools released) | 2026-09-21 ✓ | This document — initial submission |
| gemara schema proposals | 2026-Q4 | PRs to gemaraproj/gemara for Layer 3 and citation quality extensions |
| complytime problem doc alignment | 2026-Q4 | Follow-up discussion on `MappingDocument` adoption for cross-framework workflows |
| assay evaluator protocol | 2027-Q1 | Draft evaluator protocol spec for complytime review |

---

## Contact

Issues and discussion: [github.com/Formulary-Labs/.github](https://github.com/Formulary-Labs/.github)
