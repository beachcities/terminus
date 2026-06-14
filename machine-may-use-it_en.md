<!-- 日本語版: machine-may-use-it_ja.md -->
*Japanese version: [machine-may-use-it_ja.md](./machine-may-use-it_ja.md)*

# Beyond Machine-Readable and Machine-Understandable: May the Machine *Use* It?

**Version**: 2026-06-13
**Status**: A technical concept note on data governance in the age of AI — how to attach "rules for what may be done" to a document, and how to enforce them, in an environment where data, code, and models are tightly coupled. Concrete implementation and application are covered in a separate document.

---

## 1. Background: Beyond Machine-Readable Lies "May the Machine Use It?"

Data description formats have progressively acquired two properties: expressive power for structure, and readability and safety for both humans and machines.

- **CSV** — A flat record. Cannot express hierarchy or context.
- **JSON** — Machine-oriented hierarchical structure. Expressive, but hard to quality-check by eye.
- **YAML (especially Strict YAML)** — Reconciles structure with human readability, suppressing ambiguity of interpretation to achieve deterministic handling.

All of this concerns how to *describe* data correctly and safely — the question of machine-readability and, beyond it, machine-understandability. But now that data, code, and models are tightly coupled, a different axis is needed: a layer of rules for **how a machine (AI) may use that data** — governance.

From machine-readable (can be read), to machine-understandable (the meaning is clear), and then beyond, to "may it be used?" (permissions and policy). This note addresses that final axis.

**DocLang** (LF AI & Data Foundation; a specification working group was launched in June 2026) is an AI-native document format for unstructured content that, in addition to preserving structure, semantics, and layout, integrates this governance at the specification level.

Note that the generational account above is an organization of "properties acquired in sequence," not a technical lineage. **DocLang is not a successor to YAML; it is XML-based** (`.dclg.xml`, validated with XSD plus Schematron, `pip install doclang`). Nor is it a replacement for structured-data description — its target is unstructured documents.

### 1.1 A Fit-for-Purpose Architecture (Guiding Principle)

This concept assumes a division of roles according to the nature of the data.

- **Structured data** (API integration, agent state management, etc.) → **Agentic Data (YAML, etc.)**
- **Unstructured documents** (documents shared between humans and AI) → **DocLang**

The two do not compete; their targets differ. Fixing this premise up front lets us answer the question "isn't YAML enough?" correctly.

---

## 2. The Problem: Why Prompting an LLM Is Not Enough

Delegating data processing to an AI agent while relying solely on natural-language prompt instructions has architectural limits.

1. **Destruction of structure** — LLMs tend to treat a document as a flat string, unintentionally breaking the original hierarchy or schema.
2. **Forgetting governance** — Even if the system prompt says "exclude personal information," hallucination or prompt injection can break through that constraint.
3. **Loss of provenance** — Wholesale rewriting loses the traceability of why and where something was changed.

### 2.1 What Existing Means Reach — and What They Do Not

The data-quality world already has established means. Laying out what each one covers reveals the missing layer.

- **Rule-based validation** (deterministically checking structure, types, and consistency) covers **the correctness of the data itself**. Valid and necessary, but the question of permitted use is out of scope.
- **Data catalog description standards** (such as DCAT) cover **the description of what the data is**. General-purpose, but again not a layer that *enforces*, before an operation, what may be done with that data.

So the means for "correctness of the data" and "catalog description" are in place, but the layer that mechanically enforces **who may handle the data, through which operation, and how far** is missing. And this layer cannot be substituted by writing it into a prompt. A prompt is an input to probabilistic inference, and is therefore exposed to the three limits above (destruction, forgetting, loss). This underscores why relying solely on prompts remains architecturally insufficient.

---

## 3. DocLang's Governance Metadata (Specification v0.4.0)

DocLang holds "how the data may be used" as machine-readable **declarations** in the document's `<head>`. The main per-operation controls defined in the specification are as follows.

| Metadata | What it controls |
|---|---|
| `extraction_permitted` | Whether extraction is allowed |
| `pii_extraction_allowed` | Even if the whole document is extractable, extraction of personal-information parts (names, addresses, etc.) alone can be blocked |
| `rag_indexing_allowed` | Whether the document may be taken into a RAG index (i.e., whether it may be ingested as RAG answer material) |
| `training_permitted` | Whether use for model training is allowed |

In addition, Licensing / Data classification / Retention / Compliance frameworks, and detailed PII attributes (`pii_status`, `ai_use_restriction`, etc., with mappings to GDPR, ISO 27701, and the like) are defined.

Two properties matter.

- **Component-level override** — Declared at the document level, it can be overridden per component (per part). This enables fine-grained control such as "this table is extractable; the PII in this paragraph is not."
- **Complementary to DCAT** — Whereas DCAT describes "what the data is," DocLang describes "what may be done with that data." Not a replacement, but an additional axis.

### 3.1 Separating Declaration from Enforcement (The Crux of the Design)

What DocLang defines is a machine-readable **declaration**, not an **enforcement mechanism** that controls execution. Merely pasting a declaration into a prompt leaves it as a *request*.

Therefore, a layer that reliably reads the declaration and performs a deterministic check and block **before** the LLM's probabilistic inference must be implemented on our side. This is the crux of the skills in the next section, and the keystone of this concept as a whole.

---

## 4. Agent Skills for Data Stewards

We combine DocLang's deterministic metadata with the agent's probabilistic inference. As "skills" for an AI agent, we envisage the following three.

1. **Structural and semantic inconsistency detection** — Deterministically validate the structural layer, detecting deviations from the expected data model (e.g., open-data specifications) and proposing fixes without breaking the structure.
2. **Governance and usage-policy auditing** — Mechanically interpret the governance declarations in `<head>` and decide permissibility before an operation. Not a prompt-based "request," but enforcement as a rule bound to the data.
3. **Context-preserving normalization** — Keep the multi-layer intact while applying non-destructive patches only to specific parts (such as notational inconsistencies), preserving context and the provenance of edits.

All three share a common shape: "read the declaration bound to the document, and decide behavior deterministically before the operation." If such operational skills are published with their rules and circulated as cross-cutting, reusable parts, they can prevent the reinvention of the wheel while serving as a shared digital public good that fosters collective intelligence. A taxonomy of such skills is a matter for future work.

---

## 5. A Reference Implementation: The Governance Core (Sphragis)

The core of the skills above — the part that reads a governance declaration and returns permissibility before an operation — is released as an independent, task-agnostic, framework-agnostic minimal open-source implementation.

- **Repository**: https://github.com/beachcities/sphragis (Apache-2.0; the core has zero dependencies)
- **What it does**: Interprets a DocLang `<head>` and returns, for an operation (extract / rag_index / rag_retrieve / train / share_downstream), a deterministic `allow` / `allow_with_obligations` / `deny`. Obligations (required transformation, audit, human-in-the-loop) are surfaced alongside the verdict.
- **Default posture**: strict by default (any operation not explicitly permitted is denied). This is a design meant to close, at the code level, the class of leak that happens "because it wasn't forbidden."
- **Demo**: An interactive simulator is bundled in `demo/`. You can feel what "read the declaration and mechanically block before the operation" means by trying it (trying a PII-containing extraction under strict posture is the clearest case).

This small open-source project itself embodies the concept's principle of "publish the rules and circulate them in a cross-cutting, reusable form." At its core lies a simple premise: a declaration is not an enforcement mechanism, and a prompt is merely a request—not a rule.

---

## 6. The Next Document

This note is kept at the level of a general concept. Concrete application — into which workflow, in what order, against which metrics it is embedded and its effect verified — is treated in a following document. In particular, taking natural-language database querying (Text-to-SQL) as the subject, it will work out the design of embedding the governance core as a guard layer, together with the verification (PoC) of its accuracy and enforcement strength.

---

## License

This document is provided under the **Creative Commons Attribution 4.0 International License (CC BY 4.0)**.
Full license: https://creativecommons.org/licenses/by/4.0/

- **Author**: beachcities
- **Title**: Beyond Machine-Readable and Machine-Understandable: May the Machine *Use* It?
- **Year**: 2026

Reproduction, modification, redistribution, and commercial use are permitted. The only condition is appropriate credit (attribution).

Suggested attribution:

> beachcities, "Beyond Machine-Readable and Machine-Understandable: May the Machine Use It?" (2026), CC BY 4.0.

Note that the software Sphragis ( https://github.com/beachcities/sphragis ) referenced in this document is provided separately under the **Apache License 2.0**. The document (CC BY 4.0) and the software (Apache-2.0) are under different licenses.
