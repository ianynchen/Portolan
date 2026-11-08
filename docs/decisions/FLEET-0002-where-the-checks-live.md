---
id: "FLEET-0002"
title: "Where the coherence checks live once the store exists"
status: accepted
date: 2026-11-07
deciders: [human:ianynchen]
---

# FLEET-0002: Where the coherence checks live once the store exists

- **Status:** accepted
- **Date:** 2026-11-07
- **Deciders:** owner (ianynchen), with analysis by Claude
- **Scope:** fleet-wide. Builds on [FLEET-0001](FLEET-0001-one-store-and-the-naming-scheme.md)
  (one store) and Binnacle `ADR 0002` (whole and partial replacement). Governs
  Meridian's decision lint and the Binnacle store that will hold decisions, entities
  and documents.

## The question

Meridian's decision lint was built against **files**: markdown records discovered
through a registry of repository paths, parsed from front matter, assembled into an
in-memory corpus, and checked in Python. Twenty-seven finding codes, 286 tests.

Tomorrow decisions, entities and documents live in Binnacle's tables. A lint that
loads an entire graph into Python to check it is the wrong shape at that point. So:
what of this survives, what becomes schema, and when does the file-based version go
away?

## What survives

**The semantics, which are the expensive part.** Eight tasks produced a precise,
executable specification of what coherence and staleness *mean*: the head-set walk,
covering acknowledgement, the date rule, no-double-reporting, whole-versus-partial
status invariants, self-partial refusal. Three fix rounds and two corrections to the
specification itself went into getting those right.

Translating a check into a recursive CTE is mechanical **once you know exactly what it
must do**. Knowing exactly what it must do was the work.

## The decision

### 1. Most checks become schema, not checks

Classified by what each finding code becomes once records are rows:

| Becomes | Codes | Mechanism |
|---|---|---|
| **Vanishes — an artifact of files** | `no-front-matter`, `unterminated-front-matter`, `invalid-yaml`, `front-matter-not-mapping`, `invalid-record` | There is no document to be malformed. These exist only while a file is the carrier. |
| **Vanishes — an artifact of discovery** | `source-shrunk`, `unregistered-source`, `unloaded-source`, `invalid-registry` | Discovery becomes a query. The registry's role shrinks to ingestion, per [DEC-0008](../../meridian/docs/decisions/DEC-0008-bootstrap-and-cli.md), and these become ingestion-health concerns rather than corpus findings. |
| **Column type or constraint** | `missing-field` → `NOT NULL`; `invalid-field-type` → column types; `invalid-status` → enum/`CHECK`; `invalid-date` → `date`; `invalid-decider`, `unknown-decider-kind` → `CHECK`; `invalid-id`, `id-pattern-mismatch` → `CHECK` against the kind's pattern; `unknown-field` → rejected by the schema itself | Unrepresentable rather than detected. |
| **Key or foreign key** | `duplicate-id` → primary key; `dangling-reference`, `invalid-reference`, `unknown-source` → foreign keys | Unrepresentable rather than detected. |
| **Catalog-driven trigger** | `asymmetric-supersession`, `status-inconsistent`, `supersession-cycle` | Portolan's `edge_validate` idiom. Cycle detection may be moved to a sweep if the write-time cost proves real; the other two are cheap. |
| **Sweep — the only genuine remainder** | `stale-citation`, `partially-superseded-citation` | Time-dependent: a citation goes stale when *something else* is replaced later, so nothing at insert time can catch it. |

**Twenty-five of twenty-seven either vanish or become schema. Exactly two need a sweep.**

### 2. The sweep already exists

The two remaining checks are a candidate query feeding a human review queue — which is
Binnacle's discovery sweep, already built: cursor-driven, batched, capped, writing queue
items with rationale and confidence, resolved by a human through apply or dismiss, with
auto-voiding when the underlying thing changes.

A stale-citation finding **is** a queue item. `reviewed_against` — the acknowledgement
mechanism — **is** a queue resolution. No new subsystem; a new candidate query against
existing machinery.

### 3. One shape note, because it is cheaper to know now

`reviewed_against` is a **set**, so it becomes a join table (`citation`,
`citation_acknowledgement`) rather than a column. The front-matter shape otherwise maps
one-to-one onto rows.

### 4. The oracle rule

When the store's version is built, both implementations run over the same corpus and
**must produce identical findings**. The Python implementation is the reference oracle —
its 286 tests are the specification, and a disagreement means the SQL is wrong until
proven otherwise.

### 5. While both exist, the store is authoritative

If the file lint and the store disagree outside of a deliberate oracle comparison, **the
store wins and the file lint is wrong.** Everything downstream reads the store; a
standoff between two definitions is worse than either being wrong.

## The narrowing and retirement conditions

The file lint does two jobs, and they end at different times.

**Job A — schema validation:** is this *file* a well-formed decision record? Front matter
present and parseable, fields present and well-typed, id matching its pattern.

**Job B — corpus coherence:** symmetry, cycles, status consistency, staleness.

### Narrowing: Job B ends when the store enforces it

**Condition:** the store's constraints and triggers are live, the staleness sweep is
live, and the oracle comparison (§4) produces identical findings over the fleet's real
corpus.

At that point the file lint **drops the graph checks** and keeps only schema validation.
Running them over files becomes duplicated work whose sole remaining value is failing
earlier with a friendlier message — and whose cost is the drift risk §5 exists to
arbitrate. This is the retirement that matters, because it eliminates having two
definitions of staleness.

### Full retirement: Job A ends only if no record is authored as a file

**Condition:** no decision record is created as a file — every one is authored through
Meridian directly into the store.

This is **not planned and may never happen**, and that is a deliberate position rather
than an omission. Two authoring paths will coexist:

- **Decisions captured during Meridian's gates** go straight to the store and never touch
  a file (Meridian FR-2.6). Job A never applies to them.
- **Architectural decisions like this one** are authored as markdown in a repository,
  reviewed in a pull request, and ratified at merge (DEC-0008). That workflow is *good* —
  the pull request is a better review gate for a consequential ruling than a form in a
  UI — and there is no reason to remove it.

So the honest expectation is that the file lint **narrows and then persists**, small, as
pre-ingestion validation of the git-authored path. It should not be maintained as a
parallel implementation of coherence; it should be maintained as a parser.

## Consequences

- Whoever builds the store has a specification rather than a blank page: twenty-five
  codes name their own mechanism, two name a sweep that already exists.
- The staleness sweep's design is already decided — it is Binnacle's discovery sweep with
  a different candidate query.
- Meridian's Task 8 should record the narrowing condition in `RUNBOOK.md`, so the day the
  store's checks go live, dropping the graph checks is a known step rather than a
  rediscovery.
- The file lint's test suite outlives the file lint: it becomes the oracle's test suite.

## Revisit trigger

If write-time enforcement of `supersession-cycle` proves expensive on real volumes, move
it to the sweep and record that — it is the one classification above made on expectation
rather than measurement.
