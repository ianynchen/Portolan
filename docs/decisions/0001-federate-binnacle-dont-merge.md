# Decision 0001: Federate Binnacle and Portolan under one ontology — do not merge them

- **Status:** superseded by [FLEET-0001](FLEET-0001-one-store-and-the-naming-scheme.md)
- **Superseded:** 2026-11-07 — the federation ruling is reversed. Portolan's design is
  built into Binnacle as one store; see FLEET-0001 for why, including that this
  record's own defence of separation rested on a distinction (open vs closed
  relationship vocabularies) that does not exist.
- **Date:** 2026-09-05
- **Deciders:** owner (ianynchen), with analysis by Claude
- **Scope:** fleet architecture — Portolan, Binnacle; forward-looking implications for
  Sextant, Waypoint, Meridian

## The question

Should Binnacle (the decision record, built, v0.2.0) be combined with Portolan (the
document-relationship graph, in design)? Two intuitions pulled in opposite directions:

1. *Keep them separate* — a decision is well-structured (scenario, outcome, assumptions,
   alternatives); a document is free-form with no guaranteed structure, so they need
   different treatment.
2. *Merge them* — a document is a generalized form of a decision; a decision is "a
   document plus more properties and allowed actions."

## First principles: what each thing fundamentally is

Strip both to their essence and the real distinction is not **structured vs free-form**.
It is **system of record vs system of reference** — a difference in *write discipline*.

A **Binnacle decision** is a *governed fact*. Binnacle owns the truth of it: authority
rules on every mutation, a state machine, one transaction per lifecycle act, an audit
trail the status must fold from, a human gate on all long-term change. Its entire test
edifice exists to guarantee one property: *the record cannot lie*.

A **Portolan document** is an *observed artifact*. Ground truth lives elsewhere (a file,
a repo, a wiki page). Portolan derives structure from it — entities, relationships,
claims — probabilistically, with confidence scores. Its natural operation is
*re-derivation*: re-ingest, re-extract, recompute the graph. Nothing Portolan holds is
authoritative; it is an index over things whose authors change them without asking.

A further distinction (owner's observation): a decision **defines action** — when it
applies, when it does not, what should be done within its scope. A document is
information that may or may not ever be acted on. Decisions carry applicability
conditions and normative force; documents carry content. See "Where a decision sits in
an ontology" below.

## Adversarial review of both pure positions

**Against "a decision is a specialized document" (the merge position).** The subtype
claim holds at the data level and breaks at the process level. Test with
substitutability:

- Treat a Decision like any Document? Documents are re-ingested, re-extracted, deleted
  and rebuilt when the pipeline improves. Do that to a decision row and the audit
  guarantee is destroyed.
- Treat a Document like a Decision? Then every extracted entity needs a lifecycle, an
  authority check, and a human gate — meaningless for a paragraph a parser found.

The operations differ in *kind*, not degree. A merged store where some rows are mutated
under row locks with invariants and other rows are bulk-rebuilt by pipelines is a system
whose invariants are one migration script away from being fiction.

**Against "keep them fully separate forever" (the separation position).** The most
valuable edges are the *cross-plane* ones, and full separation leaves them homeless:

- **Staleness propagation** — a decision is superseded; which design docs cite it and
  are now stale? This closes Meridian's regeneration loop and is arguably the
  highest-value query in the fleet.
- **Impact analysis** — proposed change → precedent decisions → documents implementing
  them → (via Sextant, later) code affected. Blast radius across planes.
- **Context assembly** — Meridian building an agent's working context wants governed
  decisions and relevant document sections *ranked together*, in one query.
- **Decision mining** — meeting notes and RFCs contain decision-shaped claims.
  Extracting them is Portolan's job; governing them is Binnacle's. This inverts the
  generalization intuition: documents are not generalized decisions — *documents contain
  latent decisions; Portolan discovers them; Binnacle governs them* (they enter as
  short-term candidates through the queue, human gate intact).
- **Cross-plane conflicts** — a document asserts something contradicting a current
  decision. Binnacle's conflict machinery (queue item, human resolution) already has the
  right shape; it needs document-claim evidence as an input.

Also real: fully separate systems duplicate retrieval infrastructure (two embedders, two
k-NN paths, two graph walkers).

Merge destroys Binnacle's guarantees; separation orphans the compounding value. Both
pure positions lose.

## The Palantir ontology lens

Palantir Foundry's ontology solves exactly this tension. It has **one semantic plane** —
object types (properties + links) — but distinguishes objects by **write disposition**:

- *Pipeline-backed* objects are hydrated from datasets and rebuilt when pipelines rerun.
- *Action-backed* objects are mutated only through **action types**: governed operations
  with parameters, validation rules, permissions, and audit.

Foundry did not ship two products. It shipped one ontology with two write disciplines
side by side, plus a kinetic layer (actions/functions) over the semantic one.

Mapped onto this fleet:

| Palantir concept | Fleet realization |
|---|---|
| Semantic plane (object + link types) | **Portolan** — the graph; documents, sections, entities, claims as pipeline-backed objects |
| Action-backed object family | **Binnacle** — `Decision` as an object type; `record`, `promote`, `supersede`, `resolve_conflict` as its action types; Binnacle remains the sole writer |
| Cross-type links | Live in **Portolan** (decision → doc-section, doc-claim → conflicts-with → decision): links between an authoritative object and a derived one are themselves derived |
| Kinetic layer | **Meridian** — orchestrates actions across the fleet |
| Dynamic layer (agents) | **Tradewind**-powered agents |
| Future object families | Sextant components, Waypoint workflows project in the same way |

## Where a decision sits in an ontology

An ontology entity has **properties** and **actions**. A decision is not an action — so
where does it fit?

A decision is an **object** in the ontology: it has properties (scenario, outcome,
assumptions, alternatives, validity window, subject scope) and its *own* action types
(promote, supersede, resolve-conflict). But its **semantic role** is special: its
content is a **reified norm** — a rule stated as data.

Classical ontology layers cover two modalities:

- **Properties** describe what *is* (state).
- **Actions** define what *can be done* (operations, with validation and permissions).

Decisions add the third modality: what *should* be done — the normative ("deontic")
layer. A current decision functions as a constraint and guide on how future actions —
by humans and agents — are chosen within its applicability conditions; a superseded one
stops functioning, and the audit trail proves when and why.

Concretely, decisions relate to actions in three ways:

1. **Action selection** — before acting, an agent (dynamic layer) queries current
   decisions scoped to its situation (`relevant`, `precedent`). This is policy
   resolution: the decision does not execute anything; it informs which action is
   appropriate.
2. **Action justification** — an executed action (an event in the kinetic layer) can
   cite the decision that justified it: a provenance link from action instance to
   decision object.
3. **Action validation (mature state)** — an action type's guard can literally consult
   Binnacle: "refuse this deploy action unless a current decision covers it." Palantir
   buries such rules in action-type validation code written by engineers; Binnacle's
   contribution is making the rules *data* — versioned, superseded, audited, queryable.

So the governance loop across the three layers: **decisions govern action selection;
actions produce outcomes and documents; documents are mined for candidate decisions;
the human gate turns candidates into governed policy.** The decision object is the
bridge between the semantic plane and the kinetic plane.

## The decision

**Federate, do not merge.** Portolan becomes the ontology plane; Binnacle remains the
system of record for the Decision object family and publishes into the plane.

Contract sketch (binding direction for Portolan's requirements):

1. **External-authority object kind.** Portolan defines an object kind whose instances
   are *projections* of an external system of record. Pipelines never rewrite these
   rows; they are updated only by sync from the authority. Each carries provenance:
   `authority: binnacle`.
2. **Decision projection.** Binnacle already exposes everything a projection needs:
   stable UUIDs, `changes(since)` for incremental sync, `export()` for full sync,
   statuses to mirror. Sync is eventually consistent — acceptable for queries, never for
   authority checks.
3. **Authority always via Binnacle.** Any mutation of a decision goes through Binnacle's
   API and its gates. Portolan may deep-link or delegate; it never writes.
4. **Cross-link types** (decision → document-section "derives-from"/"implemented-by",
   claim → decision "contradicts") are Portolan link types, derived and re-derivable.
5. **Shared plumbing, never shared authority.** The embedder port and possibly a common
   k-NN/graph-walk helper may be factored into a small shared library later. Domains,
   schemas, and write paths stay separate.

## Why (ranked)

1. **The governance boundary is the product.** Binnacle's value *is* that its record
   cannot lie. Any merge that puts decision rows within reach of a re-derivation
   pipeline converts a guarantee into a convention.
2. **The cross-plane value needs a shared plane, not a merged store.** Every use case
   above works with a read-only projection plus cross-links.
3. **Practicalities.** Binnacle shipped with hard-won invariants; Portolan is unbuilt.
   Merging couples the most-tested asset to the least-built one and forces one release
   cadence on both.

## Consequences and risks

- Portolan's v1 scope grows by one concept (the external-authority object kind) and
  shrinks by an entire domain (no decision lifecycle to build — it exists).
- **Sync lag** is accepted: the projection may trail the record by a sync interval.
  Anything that needs current truth asks Binnacle.
- **Ontology sprawl** is the failure mode to guard: add object types only when a real
  query needs them. v1: Document, Section, Entity/Claim, the Decision projection, and
  the cross-link types above. Sextant/Waypoint families come later.
- Meridian consumes two APIs short-term (Portolan for the plane, Binnacle for actions);
  a unified query facade can come later without moving authority.

## Supersedes / superseded by

Supersedes the earlier sketch (2026-09-01) of building Binnacle *on* semantica/AGE and
the implicit assumption that Portolan and Binnacle might share a store. Nothing
supersedes this decision yet.
