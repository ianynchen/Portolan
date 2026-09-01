# Portolan — Requirements

## 0. What Portolan is

Portolan is a knowledge graph / knowledge base system that turns a body of
documents into a living, queryable graph of knowledge — and keeps it current,
cited, and trustworthy as the documents evolve. It serves two kinds of
consumers with equal priority: **humans**, who browse and search, and **AI
agents**, which query it programmatically as their source of truth.

### 0.1 What Portolan does

- **Ingests documents and understands their structure.** Every document is
  decomposed into its section hierarchy; documents and sections are first-class
  entities in the graph, so knowledge can be asserted, superseded, and retrieved
  at the granularity the sources actually use.
- **Extracts knowledge, probabilistically but accountably.** An LLM extracts
  entities and relationships per section against a governed, self-describing
  vocabulary. Every extracted claim carries its verbatim source text, location,
  and confidence. Nothing enters the trusted graph without passing a promotion
  gate; humans curate where confidence is insufficient.
- **Understands time.** Facts carry validity: timeless, open-ended (started, end
  unknown), or closed. Supersession — of whole documents or single sections — is
  modeled natively; every query can be answered "as of" any date, and superseded
  knowledge is flagged, never hidden or lost.
- **Tracks provenance end to end.** Every fact is traceable to the document(s),
  location, extractor, and reviewer that produced it; every answer can carry
  citations.
- **Stays current at low cost.** Sources are monitored continuously; change
  detection and re-processing operate at section granularity, so an unchanged
  document set costs nothing and a one-section edit re-processes one section.
- **Retrieves by meaning, keyword, and structure together.** Hybrid search
  (semantic embeddings at coarse and fine granularity, weighted full-text over
  names and full body content, fuzzy matching) resolves to graph entities, from
  which bounded, temporally-filtered graph traversal assembles the answer —
  across languages.
- **Captures decisions from conversations — safely.** Meetings, email, and chat
  are never ingested raw. An agent detects decisions and commitments, drafts
  decision-shaped documents with provenance into the source, and a human with
  authority ratifies before ingestion. The graph carries a single, uniform trust
  level.
- **Serves agents natively.** MCP and REST interfaces expose a small,
  task-oriented operation surface; the knowledge vocabulary is itself queryable,
  so agents discover what Portolan knows before asking.

### 0.2 Knowledge structures supported

Portolan supports the full spectrum of graph knowledge structures (the eight types
popularized by extraction frameworks such as Hyper-Extract), expressed through
two structural primitives — binary relationships and n-ary relations — plus two
qualifier dimensions, time and space:

| # | Structure | Portolan capability |
|---|-----------|-------------------|
| 1 | Model (typed entity) | Typed entities with attributes under a governed, runtime-extensible vocabulary |
| 2 | List | Ordered collections of entities |
| 3 | Set | Uniqueness-constrained collections |
| 4 | Graph | Typed, attributed binary relationships; multiple relationships per entity pair; nested graphs (an entity that is itself a graph with internal structure) |
| 5 | Hypergraph | N-ary relations with named participant roles, preserved as single relations; relations can participate in further relations |
| 6 | Temporal graph | Validity periods on facts, open-ended supersession, as-of retrieval, declarative temporal integrity |
| 7 | Spatial graph | Spatial attributes on entities and relationships (when the domain requires) |
| 8 | Spatio-temporal graph | Combined spatial + temporal qualification on the same facts |

### 0.3 How Portolan should evolve

Portolan grows in deliberate stages, each gated on evidence from the previous one
rather than speculation:

1. **Foundation (now):** the authoritative document set — supersession-heavy,
   sectioned documents — with extraction, curation, temporal integrity, hybrid
   search, and the MCP/REST surface. Success measure: humans and agents answer
   "what is currently true, and why" with citations, faster and more reliably
   than document search.
2. **Conversational capture:** the formalize→ratify→ingest loop over meetings,
   email, and chat, capturing decisions and commitments at low human cost.
   Gated on the foundation being trusted — ratification only works when the
   destination is worth ratifying into.
3. **Scale and platform maturity:** growth from the initial document set toward
   [10⁶ sections / 10⁸ relationships], the managed-database scaling path
   (Cloud SQL → AlloyDB), and evidence-driven relaxation of curation thresholds
   as measured extraction precision allows.
4. **Guarded expansions (each a deliberate decision, not a default):** new
   domains admitted only where relationships are the payload, correctness has
   consequences, and the document set cross-references itself; expertise location
   (who-knows-what) as a separate, lower-trust mechanism if demanded; epistemic
   tiering of unratified knowledge only if constraint C-7 is deliberately
   revisited; graph analytics served by export, never in-system.

The evolutionary principle throughout: **extraction is commodity; the governed,
temporal, curated substrate is the scarce asset.** Portolan expands its sources
and scale, never its promiscuity about truth status.

---

This document states **what** the system must do and the qualities it must have.
It deliberately contains no design or implementation decisions; those live in the
accompanying schema/design document.

Requirement keywords follow RFC 2119 (MUST / SHOULD / MAY).

---

## 1. Functional requirements

### 1.1 Knowledge model

- **FR-1.1** The system MUST represent knowledge as typed entities and typed
  relationships, where both entities and relationships can carry arbitrary
  attributes.
- **FR-1.2** The vocabulary of entity types and relationship types MUST be
  open-ended and extensible at runtime, without schema migrations or downtime.
- **FR-1.3** The vocabulary MUST nevertheless be governed: every entity and
  relationship instance MUST conform to a declared type, and each type MAY
  declare constraints (permitted endpoint types, required attributes, whether the
  relationship is temporal).
- **FR-1.4** The vocabulary MUST be self-describing: each type MUST carry a
  human/agent-readable description, and the full vocabulary MUST be queryable so
  that agents can discover it before composing queries. Descriptions MUST state
  both what a type covers and how it differs from adjacent types it could be
  confused with (positive and negative definition), since extraction guidance
  is generated from them.
- **FR-1.5** The system MUST support binary relationships (exactly two
  participants) with attributes on the relationship itself.
- **FR-1.6** The system MUST support n-ary relationships (three or more
  participants, each with a named role), preserving the fact that the
  participants belong to a single relation (e.g., one delivery event linking
  supplier, product, destination, and contract).
- **FR-1.6a** The shape of each n-ary relation type — its permitted roles and
  per-role cardinalities — MUST be declarable in the governed vocabulary,
  enforced when instances enter the knowledge base, and discoverable by agents
  alongside the rest of the vocabulary.
- **FR-1.7** Relationships MUST be able to participate in further relationships
  (relations about relations, e.g., one event superseding another event).
- **FR-1.8** The system MUST support nested graphs: an entity that is itself a
  graph, owning both member entities and internal relationships among them, while
  the entity remains addressable as a single endpoint from outside.
- **FR-1.9** The system MUST support multiple relationships between the same pair
  of entities (different types and/or different validity periods), and MUST
  support entities related to themselves only where explicitly permitted.
- **FR-1.10** The system MUST support ordered collections and uniqueness-
  constrained collections of entities.
- **FR-1.11** The system SHOULD accommodate spatial attributes on entities and
  relationships, and combined spatio-temporal qualification, when the domain
  requires it.

### 1.2 Temporal model

- **FR-2.1** Relationships (and entities where relevant) MUST support a validity
  period expressing when the fact holds true in the world.
- **FR-2.2** The temporal model MUST distinguish three states: (a) not temporal /
  timeless, (b) started with an unknown end (open-ended), and (c) a closed
  interval. "End unknown" and "never ends" MUST NOT be conflated.
- **FR-2.3** It MUST be possible to close an open-ended fact when its end becomes
  known (e.g., when a supersession takes effect), and closing the old fact and
  asserting its successor MUST be a single atomic operation.
- **FR-2.4** The system MUST enforce temporal integrity rules declaratively where
  expressible — e.g., no two overlapping supersessions of the same target.
- **FR-2.5** All retrieval MUST support "as-of" queries: answers computed against
  the facts valid on a caller-specified date, with the current date as default.
- **FR-2.6** Superseded information MUST remain retrievable and MUST be
  identifiable as superseded (never silently deleted or hidden).

### 1.3 Document ingestion & structure

- **FR-3.1** The system MUST ingest source documents and register each with a
  stable identity, content fingerprint, and ingestion timestamp.
- **FR-3.2** The system MUST decompose documents into their section structure,
  including arbitrarily nested sections, and represent documents and sections as
  first-class entities in the graph with their containment hierarchy, ordering,
  section paths/numbers, and headings.
- **FR-3.3** Structural decomposition MUST be deterministic (parser-based, not
  model-based) and MUST NOT require human review.
- **FR-3.4** Relationships MUST be assertable at either document level or section
  level, recorded at the level the source actually states them.
- **FR-3.5** The system MUST provide a document-level rollup of section-level
  relationships for display and querying (e.g., "D2 supersedes D1, partially:
  §3.2"), derived from — and never able to contradict — the section-level facts.
  The rollup MUST distinguish whole-document from partial relationships.
- **FR-3.6** The system MUST extract images embedded in documents (figures,
  diagrams, photos, tables rendered as images), preserve them as retrievable
  assets anchored to their owning section and position, and retain captions
  where present. Images are retrieval assets, never knowledge entities in the
  graph.
- **FR-3.7** For each extracted image, the system MUST generate searchable text
  derivatives: a description of the image's content and any text appearing
  within it (OCR). These derivatives are retrieval apparatus and MUST NOT
  require human curation.
- **FR-3.8** Image re-processing MUST be change-driven: an unchanged image
  (by content fingerprint) MUST NOT be re-described when its containing
  section or document changes.
- **FR-3.9** The system MUST support document version semantics as a per-source
  policy: (a) in-place revision of a mutable document, and (b) a new revision as
  a new document that supersedes its predecessor, with the predecessor's content
  preserved and temporally closed.

### 1.4 Knowledge extraction

- **FR-4.1** The system MUST extract entities and relationships from document
  text using an LLM, with probabilistic output accepted as a design premise.
- **FR-4.2** Extraction MUST operate at section granularity.
- **FR-4.3** The extraction vocabulary MUST be generated from the governed type
  vocabulary (FR-1.3/1.4) so extractor and knowledge base cannot drift.
- **FR-4.3a** Extraction MUST NOT invent or improvise type names: extracted
  entities and relationships are restricted to the governed vocabulary.
  Relationships stated in sources but not covered by the vocabulary MUST be
  capturable in a designated unclassified overflow (with verbatim text and
  location), which MUST never be promoted to the knowledge base; recurring
  unclassified patterns MUST be surfaced to curators as candidate vocabulary
  additions (evidence-driven vocabulary evolution).
- **FR-4.3b** Unclassified relationships MUST carry the extractor's structured
  type suggestion (proposed name and inferred shape: direction, endpoint
  types, temporality, rationale) as input to the human decision — never as
  schema by default. Curators MUST be able to resolve each recurring pattern
  via exactly one of four outcomes: accept the suggestion (new type,
  agent-shaped), map to a new type (human-authored shape), map to an existing
  type, or dismiss with reason. Type creation follows the vocabulary-change
  discipline (NFR-7.4); affected content MUST be re-extracted under the
  updated vocabulary, and resolved overflow rows MUST retain the link to the
  type that resolved them.
- **FR-4.3d** Improvement of extraction guidance from review outcomes MUST be
  tiered: data-level learning (aliases, thresholds, curated examples) MAY be
  automated; changes to vocabulary descriptions — which are the source of
  extraction guidance — MUST follow the vocabulary-change discipline with a
  human promoting each amendment; automatic rewriting of extraction
  instructions MUST NOT be promoted without human approval. Every proposed
  amendment MUST carry measured evidence (precision deltas against a golden
  set and the retained review set), descriptions MUST be versioned with the
  version recorded on extractions performed under them, and materially
  amended descriptions MUST trigger re-extraction of affected content.
- **FR-4.3c** The vocabulary MUST record provenance for each type: its origin
  (seeded, human-authored, agent-suggested, human-remapped) and the evidence
  that motivated it, so the vocabulary is auditable with the same discipline
  as the facts it governs. Outcome distributions (accepted vs remapped vs
  mapped-to-existing vs dismissed) MUST be measurable as vocabulary-quality
  and extractor-quality metrics.
- **FR-4.4** Every extracted claim MUST carry: the verbatim source text
  supporting it, its character span within the source, and an extraction
  confidence score.
- **FR-4.5** Extracted references to other documents, sections, or entities MUST
  be captured as mentions (as written in the source), never as guessed internal
  identifiers; a deterministic resolution step MUST map mentions to known
  entities, recording the resolution method and confidence.
- **FR-4.6** Mentions that cannot be resolved (e.g., references to documents not
  yet ingested) MUST be retained as first-class pending items and re-resolved
  automatically as new content arrives; ingestion order MUST NOT affect the final
  graph.
- **FR-4.7** The system MUST perform entity identity resolution (deduplication)
  across documents, including alias accumulation; newly auto-created entities
  MUST be flagged as provisional until confirmed.
- **FR-4.8** The system MUST support merging duplicate entities without loss of
  relationships, provenance, or history.
- **FR-4.9** Extracted temporal statements MUST preserve open-endedness: a stated
  start with no stated end MUST be recorded as open-ended, not defaulted to a
  closed interval.
- **FR-4.10** Extraction MUST produce, per section, a canonical-language summary
  suitable for display and semantic indexing.
- **FR-4.10a** Confidence-gated promotion (FR-5.3) MUST enforce a
  non-configurable floor: relationships that mutate an entity's lifecycle
  (supersession, replacement, or any fact that closes another fact's
  validity) MUST require human review whenever either endpoint was resolved
  by probabilistic means (similarity or model-ranked matching) rather than
  exact or alias identification — regardless of confidence score. Curator
  confirmation of a vague reference MUST enrich the target entity's aliases
  so subsequent occurrences resolve deterministically; references that remain
  ambiguous even under review MUST persist as unresolved (asserting against
  no entity) and be reportable through gap-aware answers.
- **FR-4.11** Where images contain knowledge (e.g., supersession tables or
  labeled diagrams rendered as images), extraction MUST be able to derive
  candidate facts from image content; such claims MUST flow through the same
  staging, resolution, and curation gates as text-derived claims, with
  provenance identifying the source image, and MUST be subject to stricter
  review thresholds than text-derived claims (transcription-error risk).

### 1.5 Curation & quality control

- **FR-5.1** Probabilistic (extracted) facts MUST be quarantined in a staging
  area and MUST NOT enter the trusted knowledge base without passing a
  promotion gate.
- **FR-5.2** The promotion gate MUST validate candidate facts against the
  governed vocabulary and all integrity constraints; violations MUST be routed
  to human review, not silently dropped or force-inserted.
- **FR-5.3** Promotion MUST be confidence-gated with configurable thresholds per
  relationship type and per resolution method; the system MUST support a mode
  where all extracted facts require human review.
- **FR-5.4** Human reviewers MUST be able to approve, reject, or redirect each
  candidate fact, seeing the verbatim supporting text highlighted in its section
  context and the proposed change to the graph.
- **FR-5.5** Review decisions, including rejection reasons, MUST be retained (for
  precision measurement, extraction regression testing, and future training).
- **FR-5.6** Deterministic structural facts (documents, sections, containment)
  MUST bypass staging and review.

### 1.6 Provenance

- **FR-6.1** Every fact in the knowledge base MUST be traceable to its source(s):
  document, location (span) within the document, extraction/assertion method, and
  timestamp.
- **FR-6.2** A single fact MUST be able to carry multiple supporting sources.
- **FR-6.3** Retrieval responses MUST be able to include citations for every
  returned fact.
- **FR-6.4** When a source no longer supports a previously extracted fact (e.g.,
  after revision), the fact MUST NOT be auto-deleted; it MUST be flagged as
  potentially stale and queued for human review.

### 1.7 Change detection & incremental ingestion

- **FR-7.1** The system MUST monitor sources for changes, via storage-event
  triggering and/or polling, and process changes without manual initiation.
- **FR-7.2** Re-processing an unchanged document MUST be near-zero cost;
  detection MUST use content fingerprints over normalized text so that byte-level
  differences without content changes (e.g., re-exported files) are no-ops.
- **FR-7.3** Change processing MUST operate at section granularity: only changed
  sections are re-extracted, re-summarized, and re-indexed.
- **FR-7.4** Section reconciliation MUST recognize moved/renumbered sections
  (identical content at a different position) and preserve their identity,
  relationships, provenance, and indexes without re-extraction.
- **FR-7.5** Added and removed sections MUST be handled; removed sections MUST be
  retired (retrievable, marked, history preserved), never hard-deleted.
- **FR-7.6** A change to a section MUST NOT invalidate relationships asserted
  *about* that section from other documents; such inbound relationships MUST
  instead generate an advisory review item.
- **FR-7.7** Every ingestion run MUST be recorded in an auditable ledger with
  outcome and change statistics; anomalous statistics (e.g., mass section
  churn on a minor revision) MUST be surfaceable as alerts before promotion.
- **FR-7.8** Ingestion MUST be idempotent and safe under concurrent execution
  (the same document MUST NOT be processed by two workers simultaneously).
- **FR-7.9** Partial failure MUST be contained: a failed re-extraction of one
  section MUST NOT roll back or block the successful processing of others, and
  failed units MUST be independently retryable.

### 1.8 Search & retrieval

- **FR-8.1** The system MUST support retrieval by natural-language query,
  combining semantic (embedding) similarity and keyword search, with fused
  ranking.
- **FR-8.2** Keyword search MUST cover entity names, summaries, AND full document
  body content; matches in names/headings/summaries MUST rank above passing
  mentions in body text.
- **FR-8.3** The system MUST support fuzzy and partial matching on names and
  identifiers (typos, prefixes) suitable for autocomplete.
- **FR-8.4** Semantic search MUST operate at both a coarse level (what a section
  is about) and a fine level (content within long sections), with fine-grained
  hits attributed to their owning section.
- **FR-8.5** Search results MUST resolve to graph entities (typically sections or
  domain entities), from which callers can traverse the graph; search and
  browse MUST converge on the same "focus entity + neighborhood" experience.
- **FR-8.6** Retrieval MUST support bounded multi-hop graph expansion from entry
  points, with temporal (as-of) filtering applied at every hop.
- **FR-8.7** Retrieval MUST support direct graph queries: traversal by
  relationship type, transitive containment expansion, materialization of a
  nested graph's internal structure, n-ary relation queries by participant and
  role, and reverse traversal ("what refers to X").
- **FR-8.8** Search results whose subject is superseded as of the query date MUST
  be shown with their supersession context rather than hidden.
- **FR-8.9** Retrieval relevance SHOULD weight "substantially about" links above
  "mentioned in passing" links.
- **FR-8.10** Search MUST cover image-derived text (captions, descriptions, OCR)
  as a retrieval signal; image matches MUST resolve to their owning section,
  with the matching image identifiable in the result. Image-similarity search
  (query by image) MAY be added when demanded; it is not required initially.
- **FR-8.11** Retrieval MUST offer an explain mode reporting, per result, which
  retrieval signals surfaced it, their relative contributions, and why it
  matched (evidence tag); a diagnostic mode MUST trace why a specified expected
  result did or did not surface for a query. Resolution results MUST expose
  their resolution tier/confidence to consumers.
- **FR-8.12** Answers SHOULD be gap-aware: where the knowledge base holds
  pending signals relevant to a query (unresolved references, unclassified
  relationships, facts awaiting review), responses SHOULD be able to state
  specifically what is not yet known and why, rather than omitting or
  synthesizing around the gap.
- **FR-8.13** The system MUST provide, for every entity and document, a derived
  dossier view: its compiled current state as-of a given date (facts,
  relationships, summary) together with an append-only evidence timeline
  (assertions with citations, supersessions, merges, retirements). Dossiers are
  projections of the knowledge base and MUST NOT be independently editable
  stores of truth.
- **FR-8.14** The system SHOULD support exporting a tenant's derived knowledge
  (dossiers, links, properties) as a portable plain-file collection in an
  interlinked markdown format, as a read-only projection: it serves as a
  zero-cost browsing interface (especially for the personal profile) and as a
  portability guarantee. Exports MUST respect the public-surface egress policy
  and MUST NOT be treated as an editable store of truth.

### 1.9 Multilingual support

- **FR-9.1** The system MUST handle content in multiple languages (initially
  English and French), with language detected automatically per section, stored
  with a confidence score; low-confidence detections MUST be routed to review.
- **FR-9.2** Keyword search MUST apply correct per-language linguistic processing
  (stemming) and MUST be accent-insensitive; content of unknown language MUST be
  searchable without incorrect linguistic processing.
- **FR-9.3** Queries MUST NOT require the user to declare a query language;
  cross-language querying MUST work for the supported language set.
- **FR-9.4** Semantic search MUST be cross-lingual: a query in one supported
  language MUST retrieve relevant content in another.
- **FR-9.5** Entity/section summaries MUST be maintained in one canonical
  language for uniform display and indexing, while original-language text remains
  stored, searchable, and displayable; verbatim provenance text MUST remain in
  the source language.
- **FR-9.6** Adding a new language MUST be a configuration-level change, not a
  redesign.

### 1.10 Interfaces

- **FR-10.1** The system MUST expose its retrieval and query capabilities to AI
  agents via an MCP server.
- **FR-10.2** The system MUST expose equivalent capabilities via a RESTful API
  for programmatic and human-facing clients.
- **FR-10.3** Both interfaces MUST expose a small, stable, task-oriented
  operation surface (search, expand, traverse, materialize, supersede, merge,
  vocabulary discovery) rather than raw query access; internal storage MUST NOT
  be part of the public contract.
- **FR-10.4** The vocabulary discovery operation MUST return type names and
  descriptions sufficient for an agent to compose correct queries (and for
  agent system prompts to be generated from it).
- **FR-10.5** A human-facing experience MUST support graph-first browsing
  (render a focus entity and its neighborhood; traverse interactively) with
  search as an alternate entry path, including document-level relationship
  views with drill-down to section level.
- **FR-10.6** All read operations MUST accept an as-of date parameter.
- **FR-10.7** Extraction components MUST interact with the knowledge base
  exclusively through a service-level ingestion API — catalog reads and batch
  submissions — never through direct database access. The unit of submission
  is a self-contained, transport-independent batch (structure and/or
  candidates) with idempotent semantics: resubmitting a batch MUST be safe.
- **FR-10.8** Submission MUST validate against the governed vocabulary and
  return violations to the submitter; the catalog endpoint MUST support
  versioned caching so clients can pre-validate locally and detect vocabulary
  changes cheaply.
- **FR-10.9** Command-line adapters MUST be provided as a third face of the
  operation surface (alongside REST and MCP) for agentic-tool, peripheral
  (skills, commands, scripts), and human shell use. They MUST follow
  agent-ergonomic conventions (structured output mode, meaningful exit codes,
  safe retries via idempotent submission). In deployment profiles with a
  service tier, the CLI MUST operate as a thin client of the service —
  inheriting authentication and tenant binding from credentials, never flags —
  and direct library or database access MUST be disabled by configuration; a
  direct in-process backend is permitted only in single-user profiles where
  the operating-system user is the trust boundary.

### 1.11 Conversational-source formalization

- **FR-11.1** Conversational sources (email threads, chat/Slack conversations,
  meeting recordings/transcripts) MUST NOT be ingested directly as knowledge;
  they enter the knowledge base only via formalized, human-ratified documents.
- **FR-11.2** The system MUST monitor configured conversational sources and
  detect formalization-worthy signals — decisions, commitments, and material
  fact changes — via automated classification; most conversations are expected
  to produce no signal and no artifact.
- **FR-11.3** Signal-detection thresholds MUST be configurable and tunable from
  ratification outcomes.
- **FR-11.4** For each detected signal, the system MUST generate a draft formal
  document (LLM-drafted) that is decision-shaped — one document per decision or
  commitment cluster, not per conversation — including context, the decision or
  commitment, effective dates, and owners. The document template MUST be
  configurable per source (different conversation domains warrant different
  memo shapes), versioned and reviewed as configuration rather than embedded
  in code or workflow definitions.
- **FR-11.5** Drafts MUST carry provenance spans linking each claim to its
  location in the source conversation (transcript timestamp, message reference),
  and the formalized document MUST record a derived-from link to the source
  artifact(s), which are retained in artifact storage.
- **FR-11.6** Draft generation MUST deduplicate across channels: a decision
  discussed in multiple sources MUST yield one document citing all sources; the
  drafting process MUST consult the existing knowledge base for open decisions
  on the same subject.
- **FR-11.7** Drafts MUST be ratified before ingestion by a human with standing
  (decision owner or participant), not merely a knowledge-base curator; the
  ratifier's verified identity **and the hash of the exact artifact approved**
  MUST be recorded in provenance, so ratification constitutes evidence that a
  specific person approved a specific document rather than a record that an
  approval action occurred.
- **FR-11.8** The drafting process MUST emit the document together with its
  structured facts (relationships, participants, validity), and a single
  ratification approval MUST cover both — formalized documents MUST NOT require
  a second extraction-curation review; documents from outside the formalization
  loop retain the standard two-step flow.
- **FR-11.9** Ratified formalized documents MUST flow through the standard
  ingestion pipeline and receive full treatment (structure, temporality,
  supersession, provenance, search); a later memo reversing an earlier decision
  MUST supersede it under the standard version semantics.
- **FR-11.10** Source transcripts, threads, and recordings MUST NOT be
  searchable knowledge-base content or graph facts; they serve only as audit anchors
  reachable from provenance.
- **FR-11.11** People (ratifiers, decision owners, commitment participants) MUST
  be representable as first-class entities, with identity resolution across
  addresses/handles and role changes handled temporally, never by deletion.
- **FR-11.12** Monitoring of conversational sources MUST be governed by
  per-source consent/enablement configuration; visibility of
  conversation-derived content MUST be controllable. (Expertise location —
  aggregating who-discussed-what across conversations — is explicitly OUT OF
  SCOPE for this mechanism and requires a separate decision.)

### 1.12 Tenancy & content egress

- **FR-12.1** The core MUST be multi-tenant: disjoint sets of documents,
  vocabulary, and knowledge hosted for different tenants, with every core
  operation tenant-scoped. Cross-tenant relationships and cross-tenant query
  results MUST be structurally impossible, not merely unrequested.
- **FR-12.2** Each tenant MUST have its own governed vocabulary, seeded from a
  common kernel at tenant creation and evolving independently through the
  standard vocabulary-evolution loop; one tenant's vocabulary changes MUST NOT
  affect another tenant's extraction or querying.
- **FR-12.3** Tenant isolation MUST NOT degrade under load from other tenants
  (per-tenant index locality), and a migration path to hard isolation
  (schema- or database-per-tenant) MUST be preserved behind the same
  operation surface.
- **FR-12.4** The public retrieval surface (REST and MCP) MUST NOT return raw
  stored content — full section text, chunk text, full image-derived text, or
  verbatim extraction quotes. It MAY return: metadata (names, kinds, paths,
  headings), canonical summaries, facts with temporal validity, citations as
  pointers (document reference, section path, span offsets, source URI), and
  bounded match snippets (highlights) capped in length. Consumers requiring
  full text follow the source URI to the system of record, under that
  system's own access controls.
- **FR-12.5** Raw content MUST nevertheless be retained internally (it is
  required for snippet generation, re-extraction, re-embedding, and
  reconciliation) behind an internal accessor not exposed on any public
  surface; chunk text MUST NOT be stored redundantly (chunks are span
  references into section content).
- **FR-12.6** Curation and ratification surfaces are the explicit exception to
  FR-12.4: reviewing a claim requires its verbatim source text in section
  context, so content-bearing responses exist only on governance endpoints,
  which MUST be segregated from the public surface from the outset and
  role-gated when authorization is implemented.

---

## 2. Non-functional requirements

*(Quantitative targets below are initial proposals to be validated with
stakeholders; treat bracketed values as placeholders until confirmed.)*

### 2.1 Performance

- **NFR-1.1** Interactive retrieval (hybrid search + bounded expansion) SHOULD
  return in ≤ [1s] at p95 under normal load; simple lookups and single-hop
  traversals in ≤ [200ms] at p95.
- **NFR-1.2** Autocomplete/fuzzy name lookup SHOULD return in ≤ [100ms] at p95.
- **NFR-1.3** Unchanged-document change checks MUST be near-zero cost
  (fingerprint comparison only); a single-section change SHOULD be fully
  processed (excluding human review latency) within [minutes].
- **NFR-1.4** Read performance MUST NOT degrade materially during ingestion or
  re-indexing.

### 2.2 Scalability

- **NFR-2.1** The system SHOULD scale to [10⁵–10⁶] documents, [10⁶–10⁷] sections
  and entities, and [10⁷–10⁸] relationships without architectural change.
- **NFR-2.2** Read workload (agent-driven) is expected to dominate writes; the
  system MUST scale reads independently of ingestion throughput.
- **NFR-2.3** Vector search MUST remain viable as the document set grows (index build
  time, memory footprint, and recall targets maintained at scale).
- **NFR-2.4** Known scale limits (very deep unbounded traversals, whole-graph
  analytics) MUST be documented, with the escalation path identified.

### 2.3 Consistency & integrity

- **NFR-3.1** A retrieval response MUST be internally consistent: facts,
  supersession status, and citations returned as of one snapshot.
- **NFR-3.2** Integrity rules (type constraints, temporal non-overlap,
  referential integrity) MUST be enforced by the system of record itself, not
  solely by application code.
- **NFR-3.3** Derived representations (document-level rollups, indexes, summary
  fields) MUST NOT be able to contradict the underlying facts; where derived data
  is materialized for performance, staleness bounds MUST be defined.
- **NFR-3.4** Multi-step semantic operations (close-and-supersede, entity merge,
  structural reconciliation) MUST be atomic.

### 2.4 Trustworthiness & auditability

- **NFR-4.1** Every answer MUST be explainable: which facts, from which sources,
  valid over which period, promoted by which mechanism (auto vs. human).
- **NFR-4.2** The system MUST retain enough history to audit how the knowledge
  base evolved: ingestion runs, review decisions, entity merges, retirements.
- **NFR-4.3** Extraction precision MUST be measurable per relationship type and
  resolution method from review outcomes, and auto-approval thresholds MUST be
  adjustable based on that evidence.
- **NFR-4.4** The system SHOULD preserve an upgrade path to full bitemporal
  auditing ("what did the KB believe as of date T") without redesign.

### 2.5 Availability & reliability

- **NFR-5.1** Retrieval availability target: [99.9%] monthly. Ingestion MAY have
  a lower target; ingestion outages MUST NOT affect retrieval.
- **NFR-5.2** All ingestion operations MUST be idempotent and retry-safe.
- **NFR-5.3** Data MUST be backed up with defined RPO/RTO targets of
  [RPO ≤ 24h / RTO ≤ 4h]; restore procedures MUST be tested.
- **NFR-5.4** LLM-provider failures or rate limits MUST degrade extraction
  gracefully (queue and retry) without data loss or corruption.

### 2.6 Security & access control

- **NFR-6.1** All interfaces (MCP, REST, human UI) MUST require authentication;
  agent access MUST be attributable to a principal. *(MVP posture: the core is
  authorization-ignorant beyond tenancy; authentication and endpoint-class
  gating live in the service layer around it. Fine-grained document-level
  authorization and verified-principal agent identity are deferred, with the
  required seams — caller-context parameters, public/governance endpoint
  segregation, single-writer custody — preserved from the start.)*
- **NFR-6.2** Authorization MUST distinguish at minimum: read, curate
  (review/approve extraction), ratify (approve formalized documents — restricted
  to principals with standing on the decision), and administer (vocabulary and
  source configuration).
- **NFR-6.3** Write paths that bypass the promotion gate MUST NOT be exposed
  through public interfaces.
- **NFR-6.4** Data MUST be encrypted in transit and at rest; secrets (LLM API
  keys, DB credentials) MUST be managed via a secret store, never in code or
  configuration files.
- **NFR-6.5** Document content sent to external LLM/embedding services MUST
  comply with the organization's data-handling policy; the set of external
  services used MUST be explicit and reviewable. Conversational content
  (meetings, chat, email) is the most sensitive class and MUST additionally
  respect per-source consent configuration (FR-11.12) before any external
  processing.

### 2.7 Operability & observability

- **NFR-7.1** The system MUST expose health, throughput, latency, queue-depth
  (staging/review backlog), and error metrics; ingestion anomalies (NFR/FR-7.7)
  MUST alert.
- **NFR-7.2** Extraction and promotion MUST be traceable end-to-end (per
  document, per section, per fact) for debugging.
- **NFR-7.3** LLM and embedding costs MUST be measurable per document/section and
  attributable to ingestion runs.
- **NFR-7.4** Vocabulary (type catalog) changes MUST be version-controlled and
  reviewed with the same discipline as schema migrations.
- **NFR-7.5** Prompt and extraction-model changes MUST be testable against the
  retained review set (regression testing) before rollout.
- **NFR-7.6** Deterministic resolution tiers MUST improve from operational
  evidence: fallbacks to probabilistic resolution and unresolved mentions MUST
  be logged and periodically mined for new normalization rules and aliases;
  the distribution of resolution across tiers MUST be trackable over time.
- **NFR-7.7** Routine maintenance (re-resolution, duplicate-candidate
  detection, derived-content refresh, integrity audits, vocabulary-candidate
  reporting) MUST run as a scheduled, ledgered consolidation cycle whose
  outputs flow through the standard staging/curation gates — self-maintenance
  MUST NOT write directly to the trusted knowledge base.

### 2.8 Portability & evolvability

- **NFR-8.1** The system MUST run on managed PostgreSQL — Google Cloud SQL
  initially — using only extensions available there, and MUST be migratable to
  AlloyDB with no application-visible change beyond re-indexing; embeddings MUST
  survive the migration without regeneration.
- **NFR-8.2** Components with anticipated replacement (lexical ranking, ANN
  index type, embedding generation, extraction method/model, traversal
  implementation) MUST sit behind stable internal interfaces so they can be
  upgraded without changes to callers.
- **NFR-8.2a** Exactly one component (the core service) MUST hold database
  custody: it owns the schema and its migrations, and all reads and writes by
  any other component — extraction, UIs, agents — flow through its service
  interfaces. All knowledge-base writes MUST go through the core's operation
  surface (a precondition for the planned bitemporal migration remaining
  additive).
- **NFR-8.3** New entity types, relationship types, and languages MUST be
  addable as configuration/data changes, not redesigns.
- **NFR-8.4** The extraction pipeline MUST NOT be coupled to a single LLM
  provider or model version.
- **NFR-8.5** The system is implemented in a single language ecosystem
  (Python) across core, extraction, formalization, service, and CLI tiers, so
  that the personal deployment profile is a single-runtime install and
  component boundaries can collapse to in-process calls where the profile
  permits. Invariants that a static type system would otherwise enforce
  (content non-egress, context threading, exhaustive case handling) MUST be
  compensated by explicit mechanisms: boundary models that structurally cannot
  carry restricted data, strict static checking in CI, and an integration test
  suite against a real database that is treated as load-bearing rather than
  optional.
- **NFR-8.5a** Component boundaries MUST be enforced mechanically, not by
  convention: components that submit content (extraction, conversational
  formalization) MUST NOT be able to link the core library or reach the
  database, and the permitted dependency graph MUST be declared and verified
  in continuous integration. Shared data contracts MUST live in a dedicated
  contract package so boundary compliance is type-checked without exposing
  implementation.
- **NFR-8.6** The system MUST support two deployment profiles over the same
  core, contracts, and tests — enterprise/cloud (service topology, managed
  storage, conservative curation defaults) and personal/local (single-process,
  local artifact storage, filesystem-event triggering, permissive curation
  defaults with the lifecycle review floor retained, single implicit tenant) —
  differing only in port bindings and configuration, never in forked logic.
- **NFR-8.7** Intermediate ingestion artifacts (content-addressed section
  objects) MUST have configurable retention per source/profile: retained as a
  normalized-form provenance mirror, or deleted after ingestion is durably
  confirmed, to bound storage cost. Deletion MUST never precede durable
  confirmation of ingestion outcome.

### 2.9 Cost

- **NFR-9.1** Steady-state cost MUST be dominated by change volume, not document-set
  size: no full re-extraction or re-embedding of unchanged content.
- **NFR-9.2** LLM usage MUST be reserved for semantic tasks; structural tasks
  (parsing, hashing, resolution, validation) MUST be deterministic and cheap.
- **NFR-9.2a** Derivation investment (summaries, embeddings refresh, dossier
  pre-rendering) SHOULD be tiered by evidence of attention — mention frequency,
  connectivity, retrieval hits — rather than spent uniformly across all
  entities.
- **NFR-9.3** Infrastructure SHOULD start on the lower-cost managed tier (Cloud
  SQL) with defined, measurable triggers for scaling up (AlloyDB): vector-index
  scale/rebuild pain, read QPS beyond replica capacity, or need for in-database
  ML functions.

---

## 3. Constraints & assumptions

- **C-1** System of record: managed PostgreSQL on Google Cloud (Cloud SQL now,
  AlloyDB as the scaling path).
- **C-2** LLM-based extraction is accepted as probabilistic; human curation
  capacity exists and is part of the operating model.
- **C-3** Initial content languages: English and French; actual distribution to
  be verified from the first ingestion batches.
- **C-4** Source documents are predominantly sectioned documents in a
  supersession-heavy domain (documents/bulletins that supersede or supplement
  each other, in whole or by section). Source systems include SharePoint,
  Confluence, Coda, Google Drive (Docs, Slides; Sheets opt-in per folder),
  and local directories.
- **C-5** Primary consumers: AI agents (via MCP/REST) and human users (via a
  browse/search UI); both require citations and as-of correctness.
- **C-6** Graph analytics workloads (PageRank, community detection, etc.) are
  out of scope; if they arise, they are served by exporting to a dedicated
  engine, not by this system.
- **C-7** Conversational knowledge capture follows the formalize→ratify→ingest
  model: LLM drafts, humans with authority ratify, and only ratified documents
  become knowledge. The graph carries a single, uniform trust level (ratified);
  epistemic tiering of unratified claims is out of scope unless this constraint
  is revisited.
- **C-8** *(enterprise profile)* No knowledge-base content leaves the GCP trust
  boundary: every external-model or external-service integration that
  processes document content (embedding, extraction, VLM, structure recovery,
  drafting) MUST run on Vertex AI or inside the VPC. Third-party cloud
  processing services are excluded by default; component selection applies
  this constraint before feature evaluation, so it is not re-litigated per
  tool. In the personal/local profile this constraint is replaced by
  user-consented provider choice through the same integration seams.
