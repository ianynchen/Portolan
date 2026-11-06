# Decision 0002: v1 knowledge vocabulary — three kinds, four relationships, structure as columns

- **Status:** accepted
- **Date:** 2026-09-10
- **Deciders:** owner (ianynchen), with analysis by Claude
- **Scope:** Portolan knowledge model — node kinds, relationship types, structural
  storage, catalog constraint mechanism. Consumes decision 0001's federation ruling.

## The question

The seeded vocabulary in ARCHITECTURE.md was illustrative, not decided. Two
concrete failures forced the issue:

1. `SUPERSEDES` and `REPLACES` were both declared with `src_kinds`/`dst_kinds`
   NULL, so the trigger could not tell them apart and the distinction rested
   entirely on prose the extractor may not honour.
2. Decision 0001 listed `Claim` as a v1 object and required the cross-link
   `claim → decision CONFLICTS_WITH`. But `claim` is not a node kind in
   ARCHITECTURE.md — a claim becomes an `edge` on promotion, and `edge.src`
   references `node(id)`, so a fact cannot be the endpoint of another fact.

Both are symptoms of the same gap: the vocabulary was never defined with enough
rigour to be enforced, and extraction quality depends directly on those
definitions (FR-1.4, §14.4).

## Why "claim" fails as a type

Entities are what documents are **about**. Claims are what documents **say**.
The graph already stores both, differently: aboutness as edges pointing at
entity nodes, saying as edges carrying assertions with span and verbatim.
"Claim" tries to be a node for the second thing, which is why it swallows the
document — a decision is a claim, and so is a definition, and so is every
sentence. Speech-act categories are not disjoint, so they make unusable type
systems, and no extraction instruction repairs that.

Dropping `Claim` removes an import from decision 0001 rather than a design:
ARCHITECTURE.md never had it.

## The decision

### Three node kinds, in two categories

| Kind | Category | Written by | Reviewed |
|---|---|---|---|
| `document` | artifact | parser, deterministic | never (FR-5.6) |
| `section` | artifact | parser, deterministic | never (FR-5.6) |
| `entity` | subject | extraction proposes, curator promotes | always |

`entity` is deliberately **one opaque kind** in v1. Specialisation into
`component`, `person`, and the like happens through §14.4's four-outcome loop
when a query needs to pivot on the distinction — not in advance. The previously
seeded `component`, `assembly`, `delivery`, and `supersession` kinds were
illustrative and are not v1 vocabulary.

### Four relationship types

| Type | src | dst | same_category | temporal | irreflexive | asymmetric |
|---|---|---|---|---|---|---|
| `SUPERSEDES` | document, section, entity | document, section, entity | yes | **required** | yes | yes |
| `SUPPLEMENTS` | document, section, entity | document, section, entity | yes | no | yes | yes |
| `DEFINES` | document, section | entity | no | no | (automatic) | (automatic) |
| `MENTIONS` | document, section | document, section, entity | no | no | yes | **no** |

`SUPERSEDES` and `SUPPLEMENTS` are declared disjoint: one pair cannot both
displace and add to.

`MENTIONS` is deliberately not asymmetric — mutual cross-references between
sections are normal and must not be rejected.

### The category mechanism

`src_kinds`/`dst_kinds` compute a cartesian product, so they cannot express
"same category only". Two additions close that gap:

```sql
ALTER TABLE node_kind ADD COLUMN category text NOT NULL;      -- 'artifact' | 'subject'
ALTER TABLE rel_type  ADD COLUMN same_category boolean NOT NULL DEFAULT false;
```

with a check in `edge_validate` rejecting cross-category endpoints when the flag
is set. This reduces SUPERSEDES from nine permitted pairings to five: the four
artifact combinations, plus entity-to-entity.

Category is **semantic** (what the thing is). It is a separate axis from write
discipline (who may change it), which decision 0001 introduced as the
external-authority kind. The two correlate for these three kinds and come apart
when Binnacle's `Decision` projection arrives.

### Document structure is columns, not edges

`CONTAINS` is **removed entirely**. The document/section hierarchy is intra-document,
tree-shaped, and produced deterministically by the parser, so it becomes columns
on the section node: `document_id`, `parent_section_id`, `ordinal`, `path`.

This resolves §19.6's standing note that `document_id` on section nodes and
`path`/`ordinal` on CONTAINS edges were "already overdue" for graduation to real
columns.

Entity-to-entity containment does not exist in the initial corpus. If it appears,
it is one INSERT into `rel_type`, because the catalog is schema-as-data.

**Path ordering:** store the sort key as `int[]` of ordinals (`{3,2,1}`), which
sorts correctly without an extension, and keep the human-readable `"3.2.1"` as a
display string. Plain text paths sort lexicographically and break past nine
siblings.

### Standards: which one does what

| Standard | Layer | Role |
|---|---|---|
| SHACL | validation | What the catalog already is. Each `rel_type` row is a shape, closed-world, enforced by `edge_validate`. |
| OWL | vocabulary | Stable IRIs for export, plus the three characteristics that constrain rather than infer. |
| PROV-O | provenance | Sits *beneath* the relationships, describing how each edge came to exist. Does not compete with them. |

**OWL is not adopted as a reasoning layer.** The W3C OWL 2 Primer states that a
domain or range statement "is not a constraint on the knowledge, but allows a
reasoner to infer further knowledge," and calls the opposite belief a common
modeling error. Under OWL semantics, `SUPERSEDES rdfs:domain Document` plus an
extracted `entity SUPERSEDES entity` infers that the entity *is* a Document
rather than rejecting the edge — the precise failure this vocabulary exists to
prevent. Only `Irreflexive`, `Asymmetric`, and `DisjointObjectProperties`
constrain; `Functional`, `InverseFunctional`, `Transitive`, `Symmetric`, and
`Reflexive` all derive, and are not used.

## Binnacle alignment (decision 0001)

Portolan's `SUPPLEMENT_OF` is renamed **`SUPPLEMENTS`** to match Binnacle's
shipped `LinkKind`. `SUPERSEDES` already agreed. `CONFLICTS_WITH` is reserved
for the cross-plane link and takes Binnacle's name. `PROMOTED_FROM` stays
Binnacle-only: Portolan's staging-to-trusted promotion is a different concept
from tier promotion.

Two structural differences the projection must reconcile, neither a flaw:
Binnacle stores validity as two nullable timestamps on the decision, where
Portolan uses a `daterange` on the edge (which is what makes the
no-overlapping-supersession exclusion constraint expressible); and Binnacle's
`links` table carries no validity of its own, placing the date on the decision
instead of the link.

## Why (ranked)

1. **Enforceability is the point.** `src_kinds`/`dst_kinds` plus the category
   flag are the only part of a relationship definition the database checks.
   Prose descriptions guide the extractor; constraints stop it being wrong.
2. **The extractor cannot predict resolution.** §14.4 requires it to emit
   mentions, never node ids, so it cannot know whether a target resolves to a
   document or an entity. One `SUPERSEDES` lets the model report supersession
   and lets resolution plus the trigger determine legality. Splitting into
   `SUPERSEDES`/`REPLACES` would require the model to predict its own
   resolution outcome.
3. **Smaller is defensible.** §19.7 already pre-commits "fewer rel_types;
   SUPERSEDES-first" as the response to curation-capacity pressure. Four
   well-defined types beat twelve vague ones, and adding a type is one INSERT.
4. **The edge table becomes knowledge only.** With structure in columns, every
   `edge` row is an asserted fact that a human approved or a threshold
   auto-approved. §6's covering indexes stop carrying one structural row per
   section (10⁶–10⁷ rows at NFR-2.1 scale), and `kb_search`'s untyped expansion
   stops surfacing sibling sections as though they were findings.

## Consequences and risks

- **`DESCRIBES` is dropped** in favour of `DEFINES`. Accepted cost: FR-8.9 wants
  "substantially about" ranked above "mentioned in passing", so that ranking
  input is deferred until DESCRIBES is reinstated or MENTIONS is split.
- **`REPLACES` is dropped**, folded into same-category `SUPERSEDES`.
- **`MENTIONS` carries two jobs** — artifact-to-artifact cross-reference and
  artifact-to-entity aboutness. Both mean "refers to without asserting status",
  but the description's negative half must be sharp or it becomes a dumping
  ground.
- **Two traversal mechanisms.** Section subtrees are a recursive CTE over
  `parent_section_id`; the graph is edge traversal. §12's closure-table
  optimisation still applies to the former.
- **`kb_descendants` loses its `CONTAINS` default** and the §14.2 rollup view
  rewrites to join `section.document_id` directly, which shortens it.
- **Hub nodes remain a documented mechanism** (§9, FR-1.6) with no v1 instances.
  N-ary relations are unused in this vocabulary, not unsupported.
- **`decision`'s category is not settled.** If the Binnacle projection joins the
  artifact category, `decision SUPERSEDES document` becomes legal while Binnacle
  itself permits only decision-to-decision. Deferred to the projection design.
- **One-current-definition-per-entity is not enforced.** An appealing invariant
  for `DEFINES`, but it needs a computed predicate over the defining section's
  supersession status rather than a plain constraint. Deferred.

## Supersedes / superseded by

Supersedes the illustrative vocabulary seeded in ARCHITECTURE.md §2, §14.1 and
§14.3 (`component`, `assembly`, `delivery`, `supersession`, `CONTAINS`,
`SUPPLEMENT_OF`, `DESCRIBES`, `REPLACES`). Supersedes decision 0001's inclusion
of `Claim` in the v1 object set; the remaining 0001 objects (Document, Section,
Entity, the Decision projection) stand. Nothing supersedes this decision yet.
