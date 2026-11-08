---
id: "0003"
title: "markitdown normalises input formats; PDF structure is not trusted"
status: accepted
date: 2026-10-22
deciders: [human:ianynchen]
---

# Decision 0003: markitdown normalises input formats; PDF structure is not trusted

- **Status:** accepted
- **Date:** 2026-10-22
- **Deciders:** owner (ianynchen), with analysis by Claude
- **Scope:** Portolan ingestion — the format-normalisation boundary in front of the
  deterministic section parser. Consumes decision 0002's parser/curator split.

## The question

Decision 0002 makes `document` and `section` nodes **parser-produced, deterministic,
and never reviewed** (FR-5.6). The parser therefore needs input whose heading
structure it can trust, because no curator is downstream to catch a bad hierarchy.

Portolan's own corpus is markdown, but the wider requirement (REQUIREMENTS §0.1) is to
ingest "a body of documents" — in practice Word, HTML, slides, spreadsheets and PDF.
Writing one parser per format contradicts "the parser is deterministic and simple".
The question is whether to adopt a normaliser, which one, and how far to trust it.

## What was evaluated

`markitdown` (Microsoft, MIT) converts many formats to markdown. It was installed and
run in an isolated environment on Python 3.12, against generated test files in each
format. Findings (full detail and pasted outputs in the research notes):

| Format | Heading fidelity | Verdict |
|---|---|---|
| HTML | `h1`–`h6` map to exact `#` levels | trust |
| DOCX | Heading 1–3 correct; Word **Title** style degrades to plain text | trust, with the Title caveat |
| PPTX | slide title becomes `#`; bullets flat; slide breaks are HTML comments | synthetic structure only |
| XLSX | no headings; each sheet becomes `##` plus a table | synthetic structure only |
| PDF | **no headings at all** — flat text with hard line wraps, even for a cleanly styled PDF | do not trust |

Other verified properties:

- Python library, not CLI-only: `from markitdown import MarkItDown, StreamInfo`, then
  `MarkItDown(enable_plugins=False).convert(path_or_BytesIO)`. The result exposes
  `.markdown` (alias `.text_content`) and `.title`; `.title` was populated for HTML
  only. There is no streaming API.
- Version 0.1.9, MIT, Python >=3.10,<3.15.
- Local conversion needs no API key and no LLM: it ran with the proxy pointed at a
  dead port. LLM image description and Azure Document Intelligence are optional paths
  that portolan does not take.
- **Output is deterministic** — byte-identical across repeated runs and across
  processes. This is a precondition for section-level change detection (§19).
- Extras are mandatory per format: the bare package raises `MissingDependencyException`
  for docx, pptx, xlsx and pdf.
- Maintenance: roughly monthly releases, last push 2026-10-21, 739 open issues,
  still 0.x.

## The decision

### 1. Adopt markitdown as the normalisation step, behind a port

Portolan declares a `DocumentExtractor` port. markitdown is one adapter behind it, so a
layout-aware PDF extractor (for example PageIndex's tree extractor) can be substituted
per format without touching the parser. The port carries a structure-quality flag so
the parser knows whether the heading hierarchy is real or synthetic.

### 2. Install exactly four extras, pinned

```
markitdown[docx,pptx,xlsx,pdf]==0.1.9
```

`[all]` is rejected: it adds Azure, audio and YouTube support that portolan does not
use. Cost of the four extras is about 247 MB (base 135 MB — `magika` pulls
`onnxruntime` and `numpy`; the xlsx extra pulls `pandas`). No conflict was observed
with the pydantic/psycopg stack.

### 3. Trust is per format, and PDF structure is not trusted

- **HTML, DOCX** — the parser builds the section hierarchy from the emitted headings.
- **PPTX, XLSX** — synthetic structure: one slide or one sheet becomes one section.
  The parser must not infer depth from these.
- **PDF** — flat-document mode: one `document` node, no `section` children, until a
  layout-aware adapter exists. A fabricated hierarchy is worse than none, because
  FR-5.6 sends it into the trusted graph unreviewed.

### 4. The version is pinned, and upgrades are an ingestion event

Normaliser output is the input to `section_hash` and `structure_hash` (§19). A
markitdown upgrade can change the output of every file and so invalidate every hash,
which triggers corpus-wide re-extraction. Upgrades are therefore deliberate, planned
for cost, and never incidental dependency bumps.

## Consequences

- The parser stays single-format and deterministic, as decision 0002 requires.
- `document` and `section` nodes keep their never-reviewed status honestly: no format
  is allowed to assert a hierarchy it cannot support.
- `.title` is unreliable except for HTML, so the document node's name comes from the
  source URI or the first heading, not from the normaliser's title.
- The Word **Title** style arrives as body text; a document whose only title uses that
  style gets its name from the fallback above.
- A 0.x dependency with 739 open issues sits on the ingestion path. The port is what
  makes that reversible; the pin is what makes it predictable.
- Re-hashing cost on upgrade is accepted and must appear in the operational notes.

## Supersedes / superseded by

Supersedes nothing. Records the ingestion-normalisation choice left open by decision
0002 and REQUIREMENTS §0.1. Nothing supersedes this decision yet.
