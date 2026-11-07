# FLEET-0001: One store, one vocabulary, and what each thing is called

- **Status:** accepted
- **Date:** 2026-11-07
- **Deciders:** owner (ianynchen), with analysis by Claude
- **Scope:** fleet-wide. Supersedes Meridian `DEC-0003` (separate custody, no
  projection) and Portolan decisions `0001` (federate, don't merge) and `0004`
  (correspondence table and shared conventions). Amends Binnacle `ADR 0002`
  (whole and partial replacement) by widening it beyond decisions.

## The question

Portolan was specified as a knowledge graph and Binnacle as a decision record,
kept deliberately separate — twice, in writing, with the separation defended on
custody grounds. Three things then undermined that position in the same week.

1. **Both need the same relationship vocabulary.** Replacement, supplementation,
   definition and mention mean the same thing whether an endpoint is a decision
   or a document section. Keeping them apart means defining them twice.
2. **The vocabularies are the same shape.** The defence of separation rested on
   Portolan needing an *open* catalog because relationships are discovered, and
   Binnacle needing a *closed* enum. That was wrong: Portolan's own decision
   0002 defines exactly four relationship types and requires extraction to map
   into them — a closed set, deliberately extended. The distinction did not
   exist.
3. **The twice-deferred question.** "What category does a `decision` node belong
   to?" was deferred by Portolan `0002` and deferred again by Binnacle
   `ADR 0002`, because neither system owned the vocabulary alone. A question
   that keeps being punted is waiting for an owner, not for information.

## The decision

### One store

Portolan's intended design is built **into Binnacle**, now, rather than built
separately and reconciled later.

Direction matters: Binnacle is released, tested and working, and absorbs a
design. Nothing is re-founded on anything. The earlier rejection of a merge was
substantially an argument about the opposite direction — re-basing Binnacle onto
an unbuilt Portolan — and it does not apply here.

Timing is the decisive argument. **Right now there is nothing to migrate**:
Binnacle holds no production data, Portolan is five scaffold files, the fleet has
17 decision records and zero ingested documents, and the only consumer —
Meridian — is unbuilt, so breaking an API costs nothing. This is the cheapest
this change will ever be.

Building Portolan separately would not have avoided the work; it would have
duplicated the half already done — migrations, schema namespacing, pool
ownership, the LLM port pattern, a human promotion gate, temporal validity and
as-of queries, embeddings with HNSW k-NN, a suggestion queue with dedup and
confidence floors, a sweep framework with cursors and caps, an audit trail, a
mountable router, and the test discipline.

### What the merge deletes

Several recent additions exist only to bridge the split, and go away with it:
Portolan `0004`'s `binnacle_ref_map` correspondence table (ids become native),
the dual embedding-convention question (768 vs 1536), the as-of contract written
once and implemented twice, and the three-way reference split
(`unknown-source` / `unloaded-source` / `dangling-reference`) collapsing toward
a foreign key. When a design's recent additions are mostly adapters for a
boundary, the boundary is what to question.

### What each thing is called

| Name | Role |
|---|---|
| **Portolan** | **the system.** A portolan chart is the whole picture — ports and the rhumb lines between them, which is what a graph of documents, entities, decisions and their relationships is. Architecture-level, not user-facing. |
| **Binnacle** | **the store.** Objects, relationships and decisions, with governed and derived write disciplines. The instrument housing at the helm: the authoritative thing you consult to know where you are and where you point. |
| **Meridian** | **the portal and the document pipeline.** The reference line you navigate by, and the only surface a user touches. |
| **Tradewind** | **agent and model propulsion.** Every LLM and agent invocation from any component. Emphatically **not** the storage engine. |
| **Sextant** | **observation of what was actually built.** Measures real position from the code rather than the intent. |
| **Waypoint** | **execution supervision.** The checkpoint that must be passed. |

This costs no refactoring: Binnacle, Tradewind and Meridian keep their package
names and import paths. Only Portolan changes role, from component to system,
which is free because it has almost no code. **The Portolan repository becomes
the fleet repository** — home to fleet-level decisions, the shared vocabulary and
cross-component architecture. Its knowledge-graph design documents migrate into
Binnacle, where that design now lives.

### One closed, governed vocabulary

Node kinds and relationship types are **declared as data** in a catalog, forming
a **closed set extended by deliberate act** — never discovered at runtime, and
never invented by an extractor. `decision` is a node kind among the others, which
is what makes one type system coherent rather than two systems sharing a schema.
That also answers the twice-deferred category question: it is answered in the
catalog, because there is now one catalog to answer it in.

Both replacement relations from `ADR 0002` apply to **every** kind, not only
decisions: a revised specification section partially replacing an older one is
exactly as real as `DEC-0003` partially replacing `DEC-0001`.

### Write discipline differs by kind, enforced from the catalog

Governed kinds (`decision`) are mutable only through the lifecycle engine, with
its authority checks, serialisation and audit. Derived kinds (`document`,
`section`, `entity`) are re-derivable by pipelines. The difference is enforced by
catalog-driven validation, which is Portolan's native idiom rather than a bolt-on.

This is an honest downgrade from Binnacle's current guarantee — "no method exists
that can mutate content" becomes "the catalog refuses writes to governed kinds."
Accepted: enforcing invariants with constraints rather than by withholding an API
is what databases do, it is declared in one place rather than remembered as a
predicate, and it is testable.

### A kind hierarchy exists from day one

`node_kind` carries a parent and a closure table even while the initial set is
flat, with the rule that **subtypes only add and never redefine** inherited
members (the mechanism DTDL demonstrates). Specialising `entity` into
`component` or `person` later must not invalidate stored instances, and
retrofitting a hierarchy after instances exist is the expensive path.

### Properties stay untyped for now

Objects carry `props` as JSON. A declared `property_def` catalog with datatypes
and constraints is **deferred**, and becomes warranted the first time a query
needs to filter or join on a property across kinds. Additive when it comes.

### The action framework is NOT generalised

Binnacle's lifecycle engine stays a hand-written state machine for decisions. A
declarative action-type framework is deferred until a **second** kind needs its
own governed acts — the two-instances-before-abstraction rule. A state machine
you can read beats a configurable one you cannot, and this one is correct
because it was attacked by hundreds of adversarial tests.

## Ontology scope

**In scope:** typed objects, typed relationships, declared constraints, governed
writes, temporal validity with as-of queries, provenance, and semantic search.
A knowledge graph with governance.

**Out of scope for now:** formal semantics and inference — OWL, SHACL, reasoners
deriving facts nobody asserted, consistency checking over axioms.

The fleet's hard queries do not need inference. Staleness propagation, impact
analysis and "rules governing anything reachable from here" are all transitive
closure, which a recursive query answers. Kind specialisation is a closure table,
not logical subsumption. And the genuinely hard question — *do these two
decisions contradict each other?* — **a reasoner cannot answer either**: formal
logic catches contradictions that were *declared*, not two policies in tension.
That needs a model to propose and a human to judge, which is what Binnacle's
conflict detection already does. The hard problems here are semantic, not logical.

### Evolving into inference later is explicitly allowed, and three measures keep it cheap

1. **Relationship semantics are declared as data even where nothing enforces
   them.** `rel_type` already carries `irreflexive`, `asymmetric` and
   `temporal`; `transitive` is declared too, read by nothing today. The axioms a
   reasoner would need are then already written down rather than buried in code.
2. **Asserted and derived facts stay distinguishable, permanently.** Any computed
   fact — a stored transitive closure, say — is marked derived with its
   derivation recorded. This is the one choice that would genuinely foreclose the
   option: if asserted and computed facts are indistinguishable, nothing can be
   safely recomputed and no reasoner can be adopted without untangling them by
   hand.
3. **The kind hierarchy with subtypes-only-add** is already what formal
   subsumption requires. No additional work.

**Caveat recorded rather than solved:** SQL is closed-world — a fact's absence
means false. Reasoners are open-world — absence means unknown. Adopting one is a
semantic shift to be reconciled, not a migration.

**Threshold for revisiting:** when the same recursive query is written repeatedly
to derive the same category of fact, and the rule is stable and declarative.
That is an axiom looking for a home. A one-off traversal is just a query.

## Consequences

- Meridian's portal story simplifies: one API to compose, not two, and no
  correspondence table between them.
- Meridian's in-flight decision lint is unaffected — it operates on front matter,
  not on either store — and its vocabulary (whole and partial replacement) is
  the same one defined here.
- Binnacle's `links.kind` moves from a closed enum to a catalog reference, and
  its endpoints widen from decisions to nodes. Feasible precisely because the
  table holds no data.
- Binnacle stops being a small library embeddable for decisions alone. Accepted:
  no consumer outside this fleet exists.
- Ingesting the fleet's own 17 decision records remains blocked on `ADR 0002`'s
  partial-replacement implementation, which this decision widens to all kinds.

## Revisit trigger

If a governed kind and a derived kind ever need genuinely incompatible storage —
not merely different write rules, but different physical shapes — the single
store is the wrong answer and the boundary returns. Nothing observed so far
suggests that; the difference has been in discipline, which the catalog expresses.
