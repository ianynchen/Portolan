---
id: "0004"
title: "Portolan holds the Binnacle correspondence, and the conventions both sides implement"
status: superseded
date: 2026-11-06
deciders: [human:ianynchen]
supersedes:
  - {id: "0001", part: "the projection expectation"}
  - {id: "0002", part: "the deferred decision-category item, now closed"}
superseded_by: ["FLEET-0001"]
---

# Decision 0004: Portolan holds the Binnacle correspondence, and the conventions both sides implement

- **Status:** superseded by [FLEET-0001](FLEET-0001-one-store-and-the-naming-scheme.md)
- **Superseded:** 2026-11-07 — the correspondence table existed only to bridge two
  stores. With one store, ids are native and `binnacle_ref_map` is unnecessary, as
  are the shared embedding and as-of conventions it carried.
- **Date:** 2026-11-06
- **Deciders:** owner (ianynchen), with analysis by Claude
- **Scope:** Portolan ↔ Binnacle identity and shared conventions. Consumes
  decision 0001's federation ruling and closes decision 0002's open `decision`
  category item. Mirrors Meridian
  `docs/decisions/DEC-0003-separate-custody-no-projection.md`.

## The question

Decision 0001 federated Portolan and Binnacle. Meridian then ruled that
promoted decisions are **not** projected into Portolan — each system serves a
different piece. That leaves one problem unanswered: a query spanning both
("decisions about the entities this blueprint touches") needs a join key, and
Binnacle subject refs are open strings `(role, kind, identifier)` while
Portolan has node ids, accumulated aliases, and entity merges.

Separation left unaddressed, four costs compound with use: queries that can
only fan out, identity drift as the graph curates beneath recorded refs, two
supersession/temporal models drifting apart, and two incommensurable embedding
spaces.

## Why not node ids inside Binnacle refs

The obvious fix — Binnacle refs carry Portolan node ids — was considered and
**rejected**:

1. It makes Portolan the identity authority for another system's data.
2. It imposes a hard ordering: no decision is recordable until its entity
   exists in Portolan, so every Binnacle write acquires a Portolan
   prerequisite.
3. It ends Binnacle's standalone usability. Binnacle is a library with no
   knowledge of Portolan, which is why it shipped first.
4. Decisively, it points **authoritative data at derived data**. Portolan's
   defining operation is re-derivation; a governed decision holding a foreign
   key into a regenerable graph develops dangling references the first time an
   entity is retired or re-extracted. Dependencies must point toward the
   stable component, never away from it.

## The decision

**Portolan holds a correspondence; Binnacle does not know Portolan exists.**

```sql
-- Portolan's schema. Derived, re-derivable, and owned here precisely because a
-- correspondence between a governed record and a regenerable graph is itself
-- regenerable.
CREATE TABLE binnacle_ref_map (
    ref_kind        text   NOT NULL,     -- Binnacle ref kind, e.g. 'component'
    ref_identifier  text   NOT NULL,     -- as recorded in Binnacle, verbatim
    node_id         bigint NOT NULL REFERENCES node(id) ON DELETE CASCADE,
    resolved_by     text   NOT NULL,     -- resolver version or 'curator'
    confidence      real,
    resolved_at     timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (ref_kind, ref_identifier, node_id)
);
CREATE INDEX binnacle_ref_map_node_idx ON binnacle_ref_map (node_id);
```

- The resolver writes it, using the same identification machinery and alias
  accumulation it already applies to document mentions (FR-4.9, FR-4.11).
- `kb_merge_nodes` cascades here, so entity curation repoints the mapping and
  **leaves the decision record saying exactly what it said** — the better audit
  answer as well as the cheaper one.
- Resolution is asynchronous and optional. An unresolved ref is a valid resting
  state, handled exactly as FR-4.6 handles unresolvable mentions; such a
  decision stays fully valid and reachable by text and precedent search.
- The composite primary key permits more than one node per ref deliberately: an
  ambiguous ref is a curation item, not a constraint violation.

### The spanning query

Traverse in Portolan → node ids → `binnacle_ref_map` → ref strings → one
Binnacle `relevant(subjects=[...])` call. No string matching, no alias
guessing, no per-entity fan-out. The mapping supplies what shared ids would
have supplied, without the dependency.

## Reasoning

- **A spanning query needs a join key.** Binnacle subject refs are open strings while
  Portolan has node ids, accumulated aliases, and entity merges. Separation left
  unaddressed, four costs compound with use: queries that can only fan out, identity
  drift, two supersession/temporal models drifting apart, and two incommensurable
  embedding spaces.
- **Node ids inside Binnacle refs point the wrong way.** Decisively, that fix points
  authoritative data at derived data. Dependencies must point toward the stable
  component, never away from it.
- **The correspondence belongs in Portolan.** A correspondence between a governed
  record and a regenerable graph is itself regenerable. Entity curation repoints the
  mapping and leaves the decision record saying exactly what it said — the better
  audit answer as well as the cheaper one.
- **Binnacle stays standalone.** Binnacle does not know Portolan exists and needs no
  code change. The mapping supplies what shared ids would have supplied, without the
  dependency.

## Conventions both sides implement

- **One embedding model and dimension.** Portolan's `node.embedding` is
  currently declared `vector(1536)` while Binnacle runs nomic-embed-text-v1.5
  at 768. Settle this before either corpus has volume: changing it later means
  re-embedding everything. The choice is **driven by Portolan's language
  requirements** — multilingual retrieval with canonical-English summaries may
  demand a different model than Binnacle's English-centric default — not by
  defaulting to whichever shipped first. Binnacle templates `embedding_dim`
  into its migration, so on its side this is a config value, not a code change.
- **One as-of contract**, stated once and implemented by both: what "valid at
  T" means for a fact, a decision, and a superseded thing. Binnacle's
  implementation is the reference, being built and tested; Portolan conforms.

## Consequences

- Closes decision 0002's open item: with no projection there is no `decision`
  node kind, so `decision SUPERSEDES document` never becomes legal and the
  category question is moot. Hub nodes remain documented with no v1 instance.
- Binnacle needs **no code change** for any of this. An optional multi-subject
  `relevant(subjects=[...])` form is an optimisation, deferrable until
  per-entity fan-out measures as the bottleneck; N single-subject calls are
  correct meanwhile.
- Meridian's authoring UI should autocomplete subject refs from Portolan
  entities, so the recorded string is usually already canonical — a
  convenience that raises hit rates without creating a dependency.
- The pre-Portolan decision backlog resolves at lower hit rates than
  live-authored refs. That is accepted; Meridian keeps the backlog enumerable
  so it is reviewable rather than invisible.

## Supersedes / superseded by

Supersedes decision 0001's projection expectation and closes decision 0002's
deferred `decision` category item. Nothing supersedes this decision yet.

## Revisit trigger

When decision or rule predicates must participate *inside* multi-hop
traversals and the application-level fan-out is measured as the bottleneck. The
move then is re-basing Binnacle's storage on this graph behind Binnacle's
unchanged client API — not a product merge — with kind-predicated triggers
carrying its write discipline. Note that the fleet stays federated regardless:
Sextant's facts live in DuckDB.
