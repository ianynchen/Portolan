# PostgreSQL Knowledge Graph / Knowledge Base — Complete Schema Design

A single-database design for a knowledge base that:

- Stores **typed entities and relationships** with an open-ended, catalog-governed vocabulary (no table-per-relationship)
- Covers all 8 Hyper-Extract knowledge structures (model, list, set, graph, hypergraph, temporal, spatial, spatio-temporal) with **two structural primitives**: binary edges and hub nodes
- Supports **temporal validity** including open-ended ("started, no end yet") facts, with declarative integrity (no overlapping supersessions)
- Supports **nested graphs** (a node that *is* a graph, with its own internal edges — the RDF named-graph analog)
- Carries **provenance** on every fact (source document, span, confidence, review status) so both humans and agents get citable answers
- Serves **hybrid retrieval** (vector similarity + full-text + graph traversal) in one engine, one transaction — the GraphRAG pattern
- Exposes a small **function surface** for agents, so they never compose raw SQL and can never forget the temporal filter

## Design principles (the rules everything follows)

1. **A relationship that needs attributes, identity, or lifecycle is an entity in disguise.**
   - 2 participants → binary edge with properties (`edge` row)
   - 3+ participants, or a relation that participates in further relations → **hub node** (a `node` whose kind is flagged `is_relation`)
2. **Schema-as-data.** Relationship and node vocabularies live in catalog tables (`rel_type`, `node_kind`), not DDL. New type = one INSERT, no migration. Catalogs carry natural-language descriptions so agent prompts can be generated from them and never drift from the database.
3. **Graph-ness is orthogonal to kind.** "B is an assembly" (domain type) and "B contains a subgraph" (structure) are separate facts — `kind` stays a domain type; subgraph scoping is expressed by `edge.graph_id`. Document structure is neither: it is a parser-produced tree living in columns on the section node (§3, decision 0002).
4. **Temporal states are explicit and distinct:**
   - `validity IS NULL` — fact is not temporal (timeless)
   - `daterange(start, NULL)` — started, end unknown (open-ended)
   - `daterange(start, end)` — closed interval
   - avoid `'infinity'` unless the domain genuinely asserts permanence
5. **Extraction output is untrusted.** LLM-extracted facts land in staging; catalog-driven validation is the quality gate; approval promotes rows into the real tables with provenance attached.
6. **Machines propose, humans promote, evidence accompanies both.** The single rule behind extraction gating (§14.6), the four-outcome vocabulary loop (§14.4), consolidation sweeps (§17.6), self-improvement amendments (§17.7), and conversational ratification (§16). Any future automation proposal is answered by it: automate the *labor* of proposing without limit, provided the proposal carries measured evidence and a human promotes it. Automating the judgment away is the one move the architecture does not permit — and not only for correctness: curation is also how the organization stays in contact with its own knowledge (*don't delegate understanding*).

---

## 1. Extensions

```sql
CREATE EXTENSION IF NOT EXISTS vector;      -- pgvector: embedding similarity
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- exclusion constraints mixing = and &&
-- CREATE EXTENSION IF NOT EXISTS postgis;  -- only when spatial data actually arrives
-- CREATE EXTENSION IF NOT EXISTS pg_jsonschema;  -- optional: props validation
```

---

## 2. Catalog tables (schema-as-data)

These are the "edge constraints table" — the schema of the graph, stored as data.
Agents can `SELECT` these to learn the vocabulary before composing queries.

```sql
CREATE TABLE node_kind (
    name         text PRIMARY KEY,
    category     text NOT NULL,                    -- 'artifact' | 'subject' (decision 0002)
    is_relation  boolean NOT NULL DEFAULT false,  -- true = hub node (n-ary relation/event)
    is_container boolean NOT NULL DEFAULT false,  -- true = may scope subgraphs (graph_id)
    props_schema jsonb,                           -- optional JSON Schema for node.props
    description  text NOT NULL                    -- FOR AGENTS: what this kind means
);

CREATE TABLE rel_type (
    name         text PRIMARY KEY,
    src_kinds    text[],                          -- allowed source kinds; NULL = any
    dst_kinds    text[],                          -- allowed destination kinds; NULL = any
    same_category boolean NOT NULL DEFAULT false, -- both endpoints must share node_kind.category
    is_temporal  boolean NOT NULL DEFAULT false,  -- if true, edges should carry validity
    is_irreflexive boolean NOT NULL DEFAULT true, -- reject src = dst
    is_asymmetric  boolean NOT NULL DEFAULT false,-- reject B->A when A->B exists
    props_schema jsonb,                           -- optional JSON Schema for edge.props
    description  text NOT NULL                    -- FOR AGENTS: semantics + direction convention
);
```

```sql
-- Declared shapes of n-ary relations: which roles a hub kind composes,
-- with cardinalities. Turns hub structure from convention into data.
CREATE TABLE hub_role (
    hub_kind  text NOT NULL REFERENCES node_kind(name),
    rel_type  text NOT NULL REFERENCES rel_type(name),
    min_count int  NOT NULL DEFAULT 1,
    max_count int,                        -- NULL = unbounded
    PRIMARY KEY (hub_kind, rel_type)
);

-- Mutually exclusive relationship types: one ordered pair of nodes may not
-- carry both. The OWL owl:propertyDisjointWith characteristic, enforced.
CREATE TABLE rel_disjoint (
    rel_type_a text NOT NULL REFERENCES rel_type(name),
    rel_type_b text NOT NULL REFERENCES rel_type(name),
    PRIMARY KEY (rel_type_a, rel_type_b),
    CHECK (rel_type_a < rel_type_b)
);
```

**Constraint columns come from OWL, deliberately narrowly.** `is_irreflexive`,
`is_asymmetric`, and `rel_disjoint` are the only three OWL object-property
characteristics that *constrain*; `Functional`, `InverseFunctional`,
`Transitive`, `Symmetric` and `Reflexive` all *infer* new facts and are not
used. `rdfs:domain`/`rdfs:range` are likewise inference axioms, not constraints
— under OWL semantics an out-of-domain edge reclassifies its endpoint rather
than being rejected. `src_kinds`/`dst_kinds`/`same_category` therefore carry
closed-world SHACL semantics, not OWL semantics. See decision 0002.

`hub_role` pays four times: (1) the validation trigger (§7) enforces hub
completeness at promotion — "a delivery has exactly one supplier" becomes a
declarative rule, not convention; (2) vocabulary discovery returns whole relation
*shapes* ("delivery = supplier + product(s) + destination [+ contract]") for
agents and for extraction-prompt generation; (3) n-ary extraction output is
validated structurally before human review; (4) the review UI renders a hub as a
form with labeled slots rather than a bag of edges.

**Description discipline (both catalogs):** every description answers two
questions — *what belongs here*, and *what does NOT belong here versus the
neighboring types it could be confused with*. Extraction prompts are generated
from these descriptions (§14.4), so the negative half directly reduces LLM
confusion between adjacent types. Example: `DEFINES: src establishes what dst
is. NOT for text that merely refers to the entity (use MENTIONS); NOT between
two artifacts.` This MECE-style discipline is the
single cheapest extraction-precision lever in the system.

**Property-name reuse across kinds.** When designing `props_schema`, prefer
reusing an existing property name over minting a synonym: one `genre` spanning
books, films, and podcasts makes "everything sci-fi" a natural cross-kind
query; three near-synonyms make it impossible. A standing check at every
low-level design review, alongside the §19.6 jsonb graduation test.

**The v1 vocabulary (decision 0002).** Three kinds in two categories, four
relationship types. `entity` is deliberately one opaque kind; specialisation
into `component`, `person` and the like happens through §14.4's four-outcome
loop when a query needs to pivot on the distinction, not in advance.

```sql
INSERT INTO node_kind (name, category, description) VALUES
  ('document', 'artifact',
   'A whole source document. NOT one of its sections (use section); NOT a thing the document is about (use entity).'),
  ('section',  'artifact',
   'A section of a document at any nesting depth; the unit relationships usually target. NOT the document as a whole (use document).'),
  ('entity',   'subject',
   'A thing the documents are about, identified across documents. NOT a document or section (those are artifacts); NOT a statement made about a thing.');

INSERT INTO rel_type (name, src_kinds, dst_kinds, same_category, is_temporal,
                      is_irreflexive, is_asymmetric, description) VALUES
  ('SUPERSEDES',
   ARRAY['document','section','entity'], ARRAY['document','section','entity'],
   true, true, true, true,
   'src displaces dst from a stated date. Direction: superseder -> superseded. Both endpoints share a category: an artifact displaces an artifact, an entity displaces an entity. At most one open supersession per dst (enforced). NOT for a document that adds to another without displacing it (use SUPPLEMENTS); NOT for a document reporting that one entity replaced another — that is an entity-to-entity SUPERSEDES whose source is recorded in provenance, not an edge from the document.'),

  ('SUPPLEMENTS',
   ARRAY['document','section','entity'], ARRAY['document','section','entity'],
   true, false, true, true,
   'src adds to dst without displacing it. Direction: supplement -> base. Both endpoints share a category. NOT for displacement (use SUPERSEDES); NOT for a passing reference (use MENTIONS).'),

  ('DEFINES',
   ARRAY['document','section'], ARRAY['entity'],
   false, false, true, true,
   'src establishes what dst is. Direction: defining text -> entity defined. NOT for text that merely refers to the entity (use MENTIONS); NOT between two artifacts.'),

  ('MENTIONS',
   ARRAY['document','section'], ARRAY['document','section','entity'],
   false, false, true, false,
   'src refers to dst without asserting anything about its status. Direction: referring text -> thing referred to. Covers cross-references between artifacts and passing references to entities. NOT for displacement (use SUPERSEDES); NOT for addition (use SUPPLEMENTS); NOT for establishing what something is (use DEFINES). Deliberately not asymmetric: mutual cross-references between sections are normal.');

INSERT INTO rel_disjoint (rel_type_a, rel_type_b) VALUES
  ('SUPERSEDES', 'SUPPLEMENTS');   -- one pair cannot both displace and add to
```

**No `CONTAINS`.** Document structure is intra-document, tree-shaped and
parser-produced, so it lives in columns on the section node (§3), not in edges.
Decision 0002 records why, and §19.6's standing note about `document_id`,
`path` and `ordinal` being overdue for graduation is thereby resolved.

**`hub_role` has no v1 instances.** N-ary relations remain supported (FR-1.6,
§9) and the mechanism is unchanged; this vocabulary simply does not use one yet.
The `delivery`/`supersession` hub kinds previously seeded here were
illustrative.

---

## 3. Core tables

```sql
CREATE TABLE node (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    kind       text NOT NULL REFERENCES node_kind(name),
    name       text,                    -- canonical display name
    props      jsonb NOT NULL DEFAULT '{}',
    validity   daterange,               -- entity-level temporality (NULL = timeless)
    -- geom    geometry,                -- uncomment with PostGIS when spatial arrives
    -- document structure (kind='section' only; NULL elsewhere) — decision 0002.
    -- The section tree is parser-produced, intra-document and tree-shaped, so it
    -- is columns rather than edges. This is §19.6's graduation, applied.
    document_id       bigint REFERENCES node(id),  -- owning document node
    parent_section_id bigint REFERENCES node(id),  -- NULL = top-level section
    ordinal           int,                         -- order among siblings
    path              int[],                       -- sort key, e.g. {3,2,1}
    path_label        text,                        -- display form, e.g. '3.2.1'
    heading           text,                        -- section heading as written
    -- retrieval columns:
    summary    text,                    -- 1-3 sentence description; the embedding source
    embedding  vector(1536),            -- pgvector; dimension matches your embedding model
    tsv        tsvector GENERATED ALWAYS AS
                 (to_tsvector('english',
                    coalesce(name,'') || ' ' || coalesce(summary,''))) STORED
);

CREATE TABLE edge (
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    src       bigint NOT NULL REFERENCES node(id),
    dst       bigint NOT NULL REFERENCES node(id),
    rel_type  text   NOT NULL REFERENCES rel_type(name),
    graph_id  bigint REFERENCES node(id),  -- named-graph scope; NULL = default graph
    validity  daterange,                   -- NULL = timeless; (start,NULL) = open-ended
    props     jsonb  NOT NULL DEFAULT '{}'
);
```

**Key points:**

- `edge.id` is the key, not `(src, dst)`: multiple edges between the same pair (different rel_types, different validity windows) are allowed by design.
- `graph_id` is the RDF named-graph analog: edges with `graph_id = B` are *inside* graph B; edges with `graph_id IS NULL` are top-level statements *about* things. "Materialize graph B" = `SELECT * FROM edge WHERE graph_id = :b_id`.
- `validity` is a real column (not buried in `props`) because range operators and GiST indexing need it.

---

## 4. Provenance

Every fact should be traceable to its source. One fact can be supported by multiple
documents, hence a separate assertion table rather than columns on node/edge.

```sql
CREATE TABLE document (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    uri          text NOT NULL,
    content_hash text NOT NULL,
    ingested_at  timestamptz NOT NULL DEFAULT now(),
    props        jsonb NOT NULL DEFAULT '{}'
);
CREATE UNIQUE INDEX document_hash_idx ON document (content_hash);

CREATE TABLE assertion (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    node_id     bigint REFERENCES node(id),
    edge_id     bigint REFERENCES edge(id),
    document_id bigint NOT NULL REFERENCES document(id),
    span        int4range,          -- character offsets in the source
    confidence  real,               -- extractor confidence, if any
    extractor   text,               -- e.g. 'hyper-extract:v0.1.2:template-x', 'manual'
    asserted_at timestamptz NOT NULL DEFAULT now(),
    status      text NOT NULL DEFAULT 'approved'
                CHECK (status IN ('pending','approved','rejected')),
    CHECK (num_nonnulls(node_id, edge_id) = 1)   -- exactly one target
);
CREATE INDEX assertion_node_idx ON assertion (node_id) WHERE node_id IS NOT NULL;
CREATE INDEX assertion_edge_idx ON assertion (edge_id) WHERE edge_id IS NOT NULL;
CREATE INDEX assertion_doc_idx  ON assertion (document_id);
```

The `asserted_at` / `ingested_at` timestamps double as a lightweight transaction-time
record ("what did we believe, when") — often enough audit trail without full
bitemporal modeling (see §10).

---

## 5. Staging (extraction landing zone)

LLM-extracted facts are probabilistic; the real tables are strict. Batches land here,
validation runs against the catalogs, violations route to human review, approval
promotes rows.

```sql
CREATE TABLE staging_node (
    batch_id    bigint NOT NULL,
    ext_id      text   NOT NULL,        -- extractor's local id, for edge resolution
    kind        text,                   -- unvalidated
    name        text,
    props       jsonb NOT NULL DEFAULT '{}',
    summary     text,
    validity    daterange,
    document_id bigint REFERENCES document(id),
    span        int4range,
    confidence  real,
    resolved_node_id bigint,            -- filled by entity resolution / promotion
    status      text NOT NULL DEFAULT 'pending',
    PRIMARY KEY (batch_id, ext_id)
);

CREATE TABLE staging_edge (
    batch_id    bigint NOT NULL,
    ext_id      text   NOT NULL,
    src_ext_id  text   NOT NULL,
    dst_ext_id  text   NOT NULL,
    rel_type    text,                   -- unvalidated
    graph_ext_id text,
    validity    daterange,
    props       jsonb NOT NULL DEFAULT '{}',
    document_id bigint REFERENCES document(id),
    span        int4range,
    confidence  real,
    status      text NOT NULL DEFAULT 'pending',
    violation   text,                   -- populated by validation pass
    PRIMARY KEY (batch_id, ext_id)
);
```

Promotion flow: validate kinds/rel_types against catalogs → entity-resolve
`staging_node` against existing `node` rows (name + embedding similarity) →
insert approved rows into `node`/`edge` → write `assertion` rows carrying
document, span, confidence, extractor.

---

## 6. Indexes — matched to the four query shapes

Efficiency is schema + indexes + the queries actually run. A knowledge base serving
humans and agents has four recurring query shapes; each gets its index.

### Shape 1 — Entry: "find nodes about X" (text in, node IDs out)

```sql
CREATE INDEX node_embedding_idx ON node USING hnsw (embedding vector_cosine_ops);
CREATE INDEX node_tsv_idx       ON node USING gin (tsv);
CREATE INDEX node_kind_name_idx ON node (kind, name);

-- Document structure (decision 0002): one index serves the whole-tree render
-- (`WHERE document_id = :d ORDER BY path`) and child lookup. Partial, so it
-- carries only section rows.
CREATE INDEX node_section_tree_idx ON node (document_id, path)
    WHERE document_id IS NOT NULL;
```

Hybrid entry = pgvector top-k UNION full-text match, fused (see `kb_search`, §8).
Keyword search catches exact identifiers embeddings miss ("part XJ-42"); embeddings
catch paraphrases keywords miss. **These node-level indexes cover name + summary
only — full document content, chunk-level vectors, and multilingual search are the
Search Subsystem, §13.**

### Shape 2 — Expansion: "from these nodes, walk out" (the workhorse)

```sql
CREATE INDEX edge_src_idx ON edge (src, rel_type) INCLUDE (dst, validity);
CREATE INDEX edge_dst_idx ON edge (dst, rel_type) INCLUDE (src, validity);
CREATE INDEX edge_graph_idx ON edge (graph_id, rel_type)
    WHERE graph_id IS NOT NULL;
```

Column order matters: `(src, rel_type)` makes "all outgoing SUPERSEDES edges from
node X" a single index range scan. `INCLUDE (dst, validity)` makes temporally
filtered traversal **index-only** — the hop and the as-of check never touch the heap.
This is the same adjacency-index structure a native graph store maintains internally.

### Shape 3 — Temporal: "as of date D"

```sql
CREATE INDEX edge_validity_idx ON edge USING gist (dst, validity)
    WHERE validity IS NOT NULL;
CREATE INDEX node_validity_idx ON node USING gist (validity)
    WHERE validity IS NOT NULL;
```

### Shape 4 — Type-scoped attribute lookups (add as patterns emerge, not speculatively)

```sql
-- example: only when 'reason' on SUPERSEDES edges becomes a hot filter
CREATE INDEX edge_supersedes_reason_idx ON edge ((props->>'reason'))
    WHERE rel_type = 'SUPERSEDES';
```

Partial indexes are what make single-table polymorphism perform like table-per-type:
each index is small, covers only relevant rows, and the planner uses it whenever the
query includes the `rel_type = ...` predicate (which these queries always do).

---

## 7. Constraints & validation

### Declarative (preferred wherever possible)

```sql
-- No two overlapping supersessions of the same target.
-- Note: an open-ended range [2024-01-01,) overlaps everything after its start,
-- so an open claim correctly blocks conflicting claims.
ALTER TABLE edge ADD CONSTRAINT no_overlapping_supersession
    EXCLUDE USING gist (dst WITH =, validity WITH &&)
    WHERE (rel_type = 'SUPERSEDES')
    DEFERRABLE INITIALLY IMMEDIATE;
-- DEFERRABLE allows SET CONSTRAINTS ... DEFERRED inside transactions that must
-- transiently overlap (close-then-assert patterns).

-- SUPPLEMENTS is at most one per (src, dst) pair
CREATE UNIQUE INDEX one_supplement ON edge (src, dst)
    WHERE rel_type = 'SUPPLEMENTS';

-- Self-loops are rejected per rel_type via rel_type.is_irreflexive (§7 trigger),
-- which defaults true. A blanket CHECK would prevent ever admitting a
-- legitimately reflexive type.
```

### Catalog-driven trigger (one trigger, data-driven — not per-type trigger sprawl)

```sql
CREATE FUNCTION edge_validate() RETURNS trigger AS $$
DECLARE
    rt rel_type%ROWTYPE;
    s_kind text; d_kind text; g_container boolean;
    s_cat  text; d_cat  text;
BEGIN
    SELECT * INTO rt FROM rel_type WHERE name = NEW.rel_type;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'unknown rel_type %', NEW.rel_type;
    END IF;

    SELECT n.kind, nk.category INTO s_kind, s_cat
      FROM node n JOIN node_kind nk ON nk.name = n.kind WHERE n.id = NEW.src;
    SELECT n.kind, nk.category INTO d_kind, d_cat
      FROM node n JOIN node_kind nk ON nk.name = n.kind WHERE n.id = NEW.dst;

    IF rt.src_kinds IS NOT NULL AND NOT (s_kind = ANY(rt.src_kinds)) THEN
        RAISE EXCEPTION 'rel_type % does not allow source kind %', NEW.rel_type, s_kind;
    END IF;
    IF rt.dst_kinds IS NOT NULL AND NOT (d_kind = ANY(rt.dst_kinds)) THEN
        RAISE EXCEPTION 'rel_type % does not allow destination kind %', NEW.rel_type, d_kind;
    END IF;

    -- Same-category pairing (decision 0002): src_kinds/dst_kinds compute a
    -- cartesian product and cannot express "artifact->artifact OR entity->entity".
    IF rt.same_category AND s_cat IS DISTINCT FROM d_cat THEN
        RAISE EXCEPTION 'rel_type % requires both endpoints in one category (got % and %)',
            NEW.rel_type, s_cat, d_cat;
    END IF;

    -- OWL Irreflexive / Asymmetric, the two per-edge constraint characteristics.
    IF rt.is_irreflexive AND NEW.src = NEW.dst THEN
        RAISE EXCEPTION 'rel_type % is irreflexive; src = dst = %', NEW.rel_type, NEW.src;
    END IF;

    IF rt.is_asymmetric AND EXISTS (
        SELECT 1 FROM edge e
        WHERE e.rel_type = NEW.rel_type AND e.src = NEW.dst AND e.dst = NEW.src
    ) THEN
        RAISE EXCEPTION 'rel_type % is asymmetric; the converse edge % -> % exists',
            NEW.rel_type, NEW.dst, NEW.src;
    END IF;

    -- OWL propertyDisjointWith: one ordered pair may not carry both types.
    IF EXISTS (
        SELECT 1 FROM edge e JOIN rel_disjoint d
          ON (d.rel_type_a = NEW.rel_type AND d.rel_type_b = e.rel_type)
          OR (d.rel_type_b = NEW.rel_type AND d.rel_type_a = e.rel_type)
        WHERE e.src = NEW.src AND e.dst = NEW.dst
    ) THEN
        RAISE EXCEPTION 'rel_type % is disjoint with an existing edge between % and %',
            NEW.rel_type, NEW.src, NEW.dst;
    END IF;

    IF rt.is_temporal AND NEW.validity IS NULL THEN
        RAISE EXCEPTION 'rel_type % requires validity', NEW.rel_type;
    END IF;

    IF NEW.graph_id IS NOT NULL THEN
        SELECT nk.is_container INTO g_container
        FROM node n JOIN node_kind nk ON nk.name = n.kind
        WHERE n.id = NEW.graph_id;
        IF NOT coalesce(g_container, false) THEN
            RAISE EXCEPTION 'node % is not a container kind; cannot scope a graph', NEW.graph_id;
        END IF;
    END IF;

    RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER edge_validate_trg
    BEFORE INSERT OR UPDATE ON edge
    FOR EACH ROW EXECUTE FUNCTION edge_validate();
```

### Hub-shape validation (declarative via hub_role)

Row-level triggers cannot check hub completeness (a hub's roles arrive as
multiple edge rows), so enforce it at **promotion time**: when a staged n-ary
relation is approved, the promotion transaction inserts the hub node and all its
role edges together, then validates the assembled shape against `hub_role`
(each role's count within [min_count, max_count]; no undeclared roles). A
statement-level constraint trigger (deferred to commit) or an explicit check
inside the promotion function both work; the promotion function is simpler and
sufficient, since hubs are only created through it.

### Accepted looseness (be deliberate about the bucket)

Graph-global invariants (e.g. "a transitive relationship must be acyclic, and
`parent_section_id` must not cycle") don't fit row-level
constraints. Options: deferred trigger running a recursive check, application-side
check, or a periodic integrity job. Choose per invariant and document the choice.

---

## 8. Functions — the agent-facing surface

**The single most important efficiency decision: agents call functions, not raw SQL.**
The function surface is the optimization boundary (swap CTEs for closure tables
without touching callers), the correctness boundary (the temporal filter cannot be
forgotten), and a far easier tool to hand an agent than a schema.

### 8.1 `kb_supersede` — close-and-assert as one semantic event

```sql
CREATE FUNCTION kb_supersede(
    new_superseder bigint,
    target         bigint,
    effective      date,
    reason         text DEFAULT NULL
) RETURNS bigint AS $$
DECLARE new_edge_id bigint;
BEGIN
    -- close any open supersession of the target
    UPDATE edge
    SET validity = daterange(lower(validity), effective)
    WHERE rel_type = 'SUPERSEDES'
      AND dst = target
      AND upper_inf(validity);

    -- assert the new open-ended supersession
    INSERT INTO edge (src, dst, rel_type, validity, props)
    VALUES (new_superseder, target, 'SUPERSEDES',
            daterange(effective, NULL),
            CASE WHEN reason IS NULL THEN '{}'::jsonb
                 ELSE jsonb_build_object('reason', reason) END)
    RETURNING id INTO new_edge_id;

    RETURN new_edge_id;
END $$ LANGUAGE plpgsql;
```

Ordering matters for the exclusion constraint: close first, then insert, in one
transaction.

### 8.2 `kb_descendants` — bounded transitive expansion

```sql
CREATE FUNCTION kb_descendants(
    root      bigint,
    rel       text,                   -- required; no default since CONTAINS was removed
    max_depth int  DEFAULT 10
) RETURNS TABLE (id bigint, depth int) AS $$
    WITH RECURSIVE members AS (
        SELECT e.dst AS id, 1 AS depth
        FROM edge e
        WHERE e.src = root AND e.rel_type = rel
        UNION ALL
        SELECT e.dst, m.depth + 1
        FROM edge e JOIN members m ON e.src = m.id
        WHERE e.rel_type = rel AND m.depth < max_depth
    )
    SELECT DISTINCT id, min(depth) OVER (PARTITION BY id) AS depth
    FROM members;
$$ LANGUAGE sql STABLE;
```

If this shows up hot in slow-query logs, replace the body with a trigger-maintained
closure table — callers never change (see §11).

### 8.3 `kb_graph` — materialize a named graph

```sql
CREATE FUNCTION kb_graph(g bigint)
RETURNS TABLE (edge_id bigint, src bigint, dst bigint, rel_type text,
               validity daterange, props jsonb) AS $$
    SELECT id, src, dst, rel_type, validity, props
    FROM edge WHERE graph_id = g;
$$ LANGUAGE sql STABLE;
```

### 8.4 `kb_search` — the canonical agent query (hybrid entry + scoped expansion + provenance)

```sql
CREATE FUNCTION kb_search(
    query_embedding vector(1536),
    query_text      text,
    as_of           date DEFAULT CURRENT_DATE,
    hops            int  DEFAULT 2,
    k               int  DEFAULT 10
) RETURNS TABLE (
    node_id    bigint,
    kind       text,
    name       text,
    summary    text,
    depth      int,           -- 0 = entry hit, 1..hops = expansion
    via_edge   bigint,        -- edge that led here (NULL for entry hits)
    via_rel    text,
    doc_uri    text,          -- best-supporting document, if any
    confidence real
) AS $$
    WITH vec_hits AS (                              -- Shape 1a: vector entry
        SELECT id, row_number() OVER (ORDER BY embedding <=> query_embedding) AS rnk
        FROM node
        WHERE embedding IS NOT NULL
        ORDER BY embedding <=> query_embedding
        LIMIT k
    ),
    txt_hits AS (                                   -- Shape 1b: full-text entry
        SELECT id, row_number() OVER
               (ORDER BY ts_rank(tsv, websearch_to_tsquery('english', query_text)) DESC) AS rnk
        FROM node
        WHERE tsv @@ websearch_to_tsquery('english', query_text)
        LIMIT k
    ),
    fused AS (                                      -- reciprocal rank fusion
        SELECT id, sum(1.0 / (60 + rnk)) AS score
        FROM (SELECT * FROM vec_hits UNION ALL SELECT * FROM txt_hits) u
        GROUP BY id
        ORDER BY score DESC
        LIMIT k
    ),
    expanded AS (                                   -- Shapes 2+3: temporal-filtered walk
        (SELECT id, 0 AS depth, NULL::bigint AS via_edge, NULL::text AS via_rel
         FROM fused)
        UNION
        (WITH RECURSIVE walk AS (
            SELECT f.id, 0 AS depth, NULL::bigint AS via_edge, NULL::text AS via_rel
            FROM fused f
            UNION
            SELECT e.dst, w.depth + 1, e.id, e.rel_type
            FROM walk w
            JOIN edge e ON e.src = w.id
            WHERE w.depth < hops
              AND (e.validity IS NULL OR e.validity @> as_of)
         )
         SELECT * FROM walk WHERE depth > 0)
    )
    SELECT n.id, n.kind, n.name, n.summary,
           x.depth, x.via_edge, x.via_rel,
           d.uri, a.confidence
    FROM expanded x
    JOIN node n ON n.id = x.id
    LEFT JOIN LATERAL (                             -- best-supporting assertion
        SELECT a.document_id, a.confidence
        FROM assertion a
        WHERE a.node_id = n.id AND a.status = 'approved'
        ORDER BY a.confidence DESC NULLS LAST
        LIMIT 1
    ) a ON true
    LEFT JOIN document d ON d.id = a.document_id
    ORDER BY x.depth, n.id;
$$ LANGUAGE sql STABLE;
```

Every returned fact carries a citation; the `validity @> as_of` filter is applied at
**every hop**, so agents cannot accidentally answer from superseded facts.

*Note:* with the Search Subsystem (§13) in place, the two entry CTEs here extend to
four fused legs (trigram, weighted lexical via `kb_lexical_candidates`,
section-vector, chunk-vector via `kb_chunk_candidates`) — same RRF pattern,
everything downstream unchanged.

### 8.5 Generating the agent's vocabulary prompt

```sql
-- feed this directly into the agent's system prompt / tool description
SELECT name, description FROM rel_type ORDER BY name;
SELECT name, is_relation, description FROM node_kind ORDER BY name;
```

---

## 9. Mapping the 8 knowledge structures

Two structural primitives + two qualifier dimensions cover all eight. Nothing else
is needed at the storage layer.

| Structure               | Representation                                                        |
|-------------------------|-----------------------------------------------------------------------|
| AutoModel               | `node` row (kind + props)                                             |
| AutoList                | `node` rows + ordering (`ordinal` column for sections; edge props otherwise) |
| AutoSet                 | `node` rows + partial unique index on the natural key                 |
| AutoGraph               | `node` + `edge`                                                       |
| AutoHypergraph          | **hub node** (`node_kind.is_relation = true`) + role-typed edges      |
| AutoTemporalGraph       | `validity daterange` on edges (and nodes)                             |
| AutoSpatialGraph        | `geom geometry` column (PostGIS), GiST index — add when needed        |
| AutoSpatioTemporalGraph | `geom` + `validity` on the same row (composes for free)               |

### Hyperedges via hub nodes — worked example

"Supplier S delivered Product P to Warehouse W under Contract C" is **one** relation
with four role-labeled participants. Pairwise edges lose which delivery goes with
which (the n-ary reconstruction problem); a hub node preserves it:

```sql
WITH d AS (
    INSERT INTO node (kind, name, props)
    VALUES ('delivery', 'delivery-2024-0042', '{"qty": 500}')
    RETURNING id
)
INSERT INTO edge (src, dst, rel_type)
SELECT d.id, x.dst, x.rel FROM d,
       (VALUES (:s_id, 'HAS_SUPPLIER'),
               (:p_id, 'HAS_PRODUCT'),
               (:w_id, 'HAS_DESTINATION'),
               (:c_id, 'UNDER_CONTRACT')) AS x(dst, rel);
```

Because the hub is a node, **relations about relations** work naturally:
`INSERT INTO edge (src, dst, rel_type) VALUES (:delivery_57, :delivery_42, 'SUPERSEDES' ...)`.

Rule of thumb: **binary edge with properties when the relation has exactly two
participants; hub node when it has three or more, or when the relation itself must
participate in further relations.**

### Nested graphs — worked example

*Illustrative: uses a hypothetical extended vocabulary (`component`, `assembly`,
`CONTAINS`, `CONNECTED_TO`) to show the mechanism. The v1 vocabulary (§2,
decision 0002) contains none of these — nested graphs remain supported (FR-1.8)
but are not exercised by v1's four relationship types.*

"A is a supplement of B; B is itself a graph containing B1 and B2 (which are
interconnected); A supersedes B1 from 2024-01-01, end unknown":

```sql
-- nodes
INSERT INTO node (kind, name) VALUES
    ('component', 'A'),    -- id 1
    ('assembly',  'B'),    -- id 2  (is_container kind)
    ('component', 'B1'),   -- id 3
    ('component', 'B2');   -- id 4

-- B's membership (topology)
INSERT INTO edge (src, dst, rel_type) VALUES
    (2, 3, 'CONTAINS'),
    (2, 4, 'CONTAINS');

-- B's INTERNAL wiring (edges scoped to graph B — the named-graph analog)
INSERT INTO edge (src, dst, rel_type, graph_id) VALUES
    (3, 4, 'CONNECTED_TO', 2);

-- top-level statements about B and its members
INSERT INTO edge (src, dst, rel_type) VALUES
    (1, 2, 'SUPPLEMENTS');
INSERT INTO edge (src, dst, rel_type, validity) VALUES
    (1, 3, 'SUPERSEDES', daterange('2024-01-01', NULL));  -- open-ended
```

---

## 10. Temporal modeling — usage notes

### The three states (keep them crisp)

| State                              | Meaning                                   |
|------------------------------------|-------------------------------------------|
| `validity IS NULL`                 | fact is not temporal (timeless)            |
| `daterange('2024-01-01', NULL)`    | started, **end unknown yet** (open-ended)  |
| `daterange('2024-01-01','2025-06-30')` | closed interval                        |

Avoid `'infinity'` as an upper bound unless the domain genuinely asserts permanence —
"unknown" and "forever" are different claims.

### Open-ended ranges behave correctly everywhere

```sql
-- effective today?  true for [2024-01-01,)
WHERE validity @> CURRENT_DATE

-- still open?
WHERE upper_inf(validity)

-- the exclusion constraint still works: an open range overlaps everything
-- after its start, so an open claim blocks conflicting claims — exactly right.
```

### Closing a range when the end arrives

Use `kb_supersede` (§8.1) — closing the old fact and asserting the new one is a
single semantic event and must be one transaction. Close first, then insert
(exclusion-constraint ordering).

### Bitemporal model — DELIBERATELY DEFERRED, planned for a later stage

**Decision record.** Portolan v1 tracks one time dimension: **valid time**
(`validity` — when a fact is true in the world). Full bitemporality adds a
second, independent dimension: **transaction time** (`recorded tstzrange` —
when the system believed each row). The two answer different questions:

- valid time: "what superseded B1 as of June 20?"
- transaction time: "what did Portolan *believe* superseded B1 as of June 20?" —
  point-in-time reconstruction of the graph any past answer was computed from.

Why it matters: today a correction is an `UPDATE`, which makes the graph right
about the world but **amnesiac about itself** — the six weeks during which the
KB held the wrong date, and every agent answer computed from it, become
unreconstructable. Bitemporality fixes this with append-never-update: a
correction closes the old row's `recorded` range and inserts the corrected row;
both persist forever. (Graphiti's edge model — valid_at/invalid_at +
created_at/expired_at — is the production reference implementation.)

Why we deliberately defer it: it roughly doubles the modeling surface — every
mutation becomes a paired close+insert, every read needs a `recorded` predicate
(forgetting it silently returns dead beliefs), integrity constraints must scope
to currently-believed rows, tables accrete belief history forever — and the
lightweight audit trail already in place (`assertion.asserted_at`,
`document.ingested_at`, review decisions, `ingestion_run`, the §17.1 evidence
timeline) answers "why did the KB say X" at human granularity. We pay the
write-path tax only when a real consumer of belief-history exists.

**This IS a useful feature and WILL be implemented at a later stage.** The
trigger: the first genuine requirement to audit a past answer against the exact
graph state that produced it (compliance review, dispute, regulator) —
NFR-4.4's upgrade path exercised.

**Migration plan — obligations on today's design so the upgrade stays additive:**

1. `ALTER TABLE edge ADD COLUMN recorded tstzrange NOT NULL
   DEFAULT tstzrange(now(), NULL)` — every existing row backfills as "believed
   since migration (or since a best-effort backfill from `assertion.asserted_at`),
   still believed." No data is invalidated; history before the migration is
   simply coarse.
2. Write-path discipline flips inside existing seams only: `kb_supersede`,
   promotion, `kb_merge_nodes`, and correction flows switch from UPDATE to
   close+insert pairs. **Callers never change — this is why the function
   surface (§8) must remain the only write path.**
3. Read-path: current-belief becomes `upper_inf(recorded)`; add it as a default
   predicate inside `kb_search`/`kb_descendants`/`kb_dossier`, with an optional
   `believed_at timestamptz` parameter exposing time travel. Again: seams only.
4. Constraints: existing exclusion constraints re-scope to current belief
   (`WHERE upper_inf(recorded)` added to their predicates) — verify each §7
   constraint individually during migration.
5. Indexes: the Shape-2 covering indexes gain a
   `WHERE upper_inf(recorded)` partial variant so hot current-belief traversal
   does not pay for history rows; history queries use separate GiST on
   `recorded`.
6. Storage: belief history accretes; plan retention/archival policy for closed
   `recorded` rows at migration time, not after.

**Standing constraint until then (the one rule that keeps the migration
cheap):** no application code writes to `edge`/`node` outside the §8 function
surface, and no consumer assumes a fact row is immutable. Every write path
added between now and the migration must go through a function, or the
migration stops being additive.

---

## 11. Query cookbook

### Q1 — Entry: what does the KB know about "hydraulic pump supersessions"?

```sql
SELECT * FROM kb_search(:query_embedding, 'hydraulic pump supersession',
                        as_of => CURRENT_DATE, hops => 2, k => 10);
```

### Q2 — Point lookup with temporal filter: what supersedes B1 today?

```sql
SELECT n.id, n.name, e.validity, e.props->>'reason' AS reason
FROM edge e JOIN node n ON n.id = e.src
WHERE e.rel_type = 'SUPERSEDES'
  AND e.dst = :b1_id
  AND e.validity @> CURRENT_DATE;
```

### Q3 — Time travel: what superseded B1 as of 2024-06-01?

```sql
-- identical query, different date — this is the payoff of range modeling
... AND e.validity @> DATE '2024-06-01';
```

### Q4 — Multi-relation join: who supersedes B1 AND supplements B1's container?

*Illustrative, using the same hypothetical `CONTAINS` vocabulary as §9.*

```sql
SELECT a.id, a.name, s.validity
FROM edge s
JOIN edge c   ON c.rel_type = 'CONTAINS'      AND c.dst = s.dst
JOIN edge sup ON sup.rel_type = 'SUPPLEMENTS' AND sup.src = s.src
             AND sup.dst = c.src
JOIN node a   ON a.id = s.src
WHERE s.rel_type = 'SUPERSEDES'
  AND s.dst = :b1_id
  AND s.validity @> CURRENT_DATE;
```

### Q5 — A document's section tree, any depth

Structure is columns (§3, decision 0002), so the whole tree is one indexed range
scan with no recursion — `path` already encodes the hierarchy:

```sql
SELECT id, path_label, heading, ordinal
FROM node
WHERE document_id = :doc_id
ORDER BY path;
```

A subtree below one section, when the whole document is too much:

```sql
WITH RECURSIVE sub AS (
    SELECT id, path, heading, 0 AS depth FROM node WHERE id = :section_id
    UNION ALL
    SELECT n.id, n.path, n.heading, sub.depth + 1
    FROM node n JOIN sub ON n.parent_section_id = sub.id
)
SELECT * FROM sub ORDER BY path;
```

`kb_descendants` remains available for transitive expansion over any *edge*
relationship, should the vocabulary ever admit one.

### Q6 — Materialize a named graph: B's internal structure

```sql
SELECT * FROM kb_graph(:b_id);
```

### Q7 — Hyperedge queries: all deliveries involving supplier S AND warehouse W

```sql
SELECT hub.id, hub.name, hub.props
FROM node hub
JOIN edge es ON es.src = hub.id AND es.rel_type = 'HAS_SUPPLIER'    AND es.dst = :s_id
JOIN edge ew ON ew.src = hub.id AND ew.rel_type = 'HAS_DESTINATION' AND ew.dst = :w_id
WHERE hub.kind = 'delivery';
```

### Q8 — Provenance: cite the sources behind an edge

```sql
SELECT d.uri, a.span, a.confidence, a.extractor, a.asserted_at
FROM assertion a JOIN document d ON d.id = a.document_id
WHERE a.edge_id = :edge_id AND a.status = 'approved'
ORDER BY a.confidence DESC NULLS LAST;
```

### Q9 — Vocabulary introspection (what agents run before composing queries)

```sql
SELECT name, description, src_kinds, dst_kinds, is_temporal
FROM rel_type ORDER BY name;
```

### Q10 — Reverse traversal: what does anything say about node X? (uses edge_dst_idx)

```sql
SELECT e.rel_type, n.id, n.name, e.validity
FROM edge e JOIN node n ON n.id = e.src
WHERE e.dst = :x_id
  AND (e.validity IS NULL OR e.validity @> CURRENT_DATE);
```

---

## 12. Operational notes & deliberate deferrals

**Do now:**
- The two covering edge indexes (Shape 2) — the traversal workhorse.
- Provenance from day one — retrofitting citations onto an existing KB is painful.
- Staging + catalog validation as the LLM-extraction quality gate.
- The function surface — agents call `kb_search`/`kb_descendants`/`kb_supersede`,
  never raw SQL.

**Defer until evidence demands:**
- **Closure table** for deep section or transitive hierarchies: trigger-maintained
  `(ancestor, descendant, depth)`; swap into `kb_descendants` body when recursive
  CTEs appear in slow-query logs. Callers unchanged.
- **PostGIS**: a nullable `geom` column is a cheap later migration.
- **Partitioning `edge` by rel_type**: only past ~10⁸ rows.
- **Bitemporal `recorded` column**: deliberately deferred but planned — will be
  implemented when auditing past beliefs becomes a real requirement; see §10 for
  the full decision record, migration plan, and the standing
  all-writes-through-functions constraint that keeps that migration additive.

**Never:**
- A second, parallel hyperedge table structure — hub nodes cover it with one
  uniform traversal surface, which matters when agents are the consumers.
- Graph algorithms (PageRank, community detection) in Postgres — export to a graph
  engine for those workloads if they arrive.

**Known limits (accepted trade-offs):**
- Very deep / unbounded traversals over huge fan-out: recursive CTEs materialize
  breadth-first and can't prune like native graph engines. Mitigations: depth
  bounds, closure tables, or Apache AGE (openCypher over Postgres storage) as a
  migration path — this schema is nearly isomorphic to AGE's internal layout.
- JSONB planner statistics are weaker than column statistics; if a props attribute
  becomes a dominant filter, promote it to a real column.
- EAV-adjacent risk ("nobody knows the schema anymore"): the catalogs *are* the
  schema — treat catalog changes with migration-level discipline (review, version
  control, descriptions kept current).

---

## 13. Search subsystem — full-content, chunked, multilingual retrieval

The base schema's `node.tsv` and `node.embedding` cover **name + summary** only —
enough to find nodes *about* a topic, not sections that merely *mention* a term
("torque to 45 Nm" buried in a procedure body). This section extends search to full
document content, at chunk granularity, in multiple languages (English/French
initially), while keeping the graph tables lean and every seam intact.

### 13.1 Design rules

1. **Content lives in sidecar tables, not on `node`.** The node table is the hot
   traversal surface; megabytes of prose do not belong in its heap. Sections (and
   other prose-bearing nodes) get a `node_content` row; components, hub nodes, etc.
   do not.
2. **Chunks are retrieval apparatus, not knowledge.** Chunks are never graph nodes —
   nobody asserts SUPERSEDES against "characters 2,000–3,500 of §3.2." Vector hits
   on chunks are lifted to their owning section node before anything else happens.
3. **Language is a per-section attribute**, detected deterministically at ingestion
   (lingua / CLD / fastText in the Stage-1 parser — not the LLM), stored with
   confidence. Low-confidence detections flow into the same review queue as
   uncertain edges. `'simple'` (tokenize-only, no stemming) is the honest fallback
   for unknown language: it under-recalls but never wrong-stems.
4. **Summaries are canonical-English regardless of content language** (the extraction
   LLM translates as a side effect of summarizing). `node.tsv` therefore stays a
   monolingual layer with no dispatch logic; original-language fidelity lives in
   `node_content`. Store the original-language summary in `node.props` if the UI
   needs it. Relationship *mentions* stay verbatim in the source language —
   provenance must be faithful.
5. **Pick a multilingual embedding model on day one.** Cross-lingual embeddings are
   what let an English query find French sections when lexical search cannot cross
   the boundary — and re-embedding a document set later is the one genuinely expensive
   reversal in this design. On GCP, use a Vertex multilingual embedding model
   (verify current model names/dimensions against Vertex docs before committing).
   If the document set turns out monolingual, the multilingual model cost you nothing.

### 13.2 Schema — content, chunks, language configs

```sql
CREATE EXTENSION IF NOT EXISTS unaccent;   -- accent-insensitive matching (réglage/reglage)

-- Accent-folding text search configurations (supported on Cloud SQL & AlloyDB)
CREATE TEXT SEARCH CONFIGURATION english_ua (COPY = english);
ALTER TEXT SEARCH CONFIGURATION english_ua
    ALTER MAPPING FOR hword, hword_part, word WITH unaccent, english_stem;

CREATE TEXT SEARCH CONFIGURATION french_ua (COPY = french);
ALTER TEXT SEARCH CONFIGURATION french_ua
    ALTER MAPPING FOR hword, hword_part, word WITH unaccent, french_stem;

-- Full body content, sidecar to node. tsv is language-dispatched per row.
CREATE TABLE node_content (
    node_id         bigint PRIMARY KEY REFERENCES node(id),
    content         text NOT NULL,
    content_hash    text NOT NULL,          -- re-chunk/re-embed trigger
    lang            text NOT NULL DEFAULT 'simple',   -- 'en' | 'fr' | 'simple'
    lang_confidence real,
    tsv             tsvector GENERATED ALWAYS AS (
        to_tsvector(
            CASE lang WHEN 'fr' THEN 'french_ua'::regconfig
                      WHEN 'en' THEN 'english_ua'::regconfig
                      ELSE 'simple'::regconfig END,
            content)
    ) STORED
);
CREATE INDEX node_content_tsv_idx ON node_content USING gin (tsv);
-- tsvectors from different configs coexist in one column and one GIN index;
-- correctness comes from matching the QUERY's parsing to the row's (§13.4).
-- Note: a tsvector maxes out ~1MB — one more reason content is indexed at
-- SECTION granularity, never whole-document.

-- Sub-section chunks: the fine-grained vector retrieval unit.
CREATE TABLE chunk (
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    node_id   bigint NOT NULL REFERENCES node(id),  -- owning section
    ordinal   int    NOT NULL,
    span      int4range NOT NULL,  -- offsets within node_content.content
    lang      text   NOT NULL DEFAULT 'simple',
    embedding vector(1536),     -- dimension must match your chosen multilingual model
    UNIQUE (node_id, ordinal)
);
-- NO content column: chunk text is node_content.content sliced by span, on
-- demand, via the internal content accessor (§19.2). Storing it again would
-- duplicate every document; the span is the chunk. Chunk text is needed only
-- server-side (embedding regeneration, snippet rendering) — same database,
-- one substring.
CREATE INDEX chunk_embedding_idx ON chunk USING hnsw (embedding vector_cosine_ops);
-- On AlloyDB: swap for USING scann (embedding cosine) WITH (num_leaves = ...)

-- Trigram over names: autocomplete + fuzzy identifiers, language-immune
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX node_name_trgm_idx ON node USING gin (name gin_trgm_ops);
```

Chunking policy: ~500–1,000 tokens, 10–15% overlap, **never across section
boundaries** (the section is already a semantically coherent hard boundary — half
of what fancy chunking strategies try to recover). Re-chunk and re-embed when
`content_hash` changes.

### 13.3 The four retrieval legs

Each leg answers a different question; RRF fuses them without needing comparable
scores. All legs land on **section/node IDs** before fusion.

| Leg | Index | Answers | Language handling |
|-----|-------|---------|-------------------|
| 1. Trigram | `node_name_trgm_idx` | fuzzy/partial names, identifiers, autocomplete (`xj42`, `hydralic`) | immune (character n-grams) |
| 2. Weighted lexical | `node.tsv` (weight A) + `node_content.tsv` (weight C) | keyword relevance; heading/summary hits outrank body mentions | per-row config + dual-parsed query |
| 3. Section embeddings | `node.embedding` | sections *about* X (topical; embeds summary + relationship context) | cross-lingual model |
| 4. Chunk embeddings | `chunk.embedding` | sections *containing* X (deep content) — lifted to owning node | cross-lingual model |

Leg 3's embedded text should be composed: `name + kind + summary + one hop of
relationship context` ("XJ-42 — component — hydraulic pump seal — supersedes XJ-40,
contained in Assembly B"). Including neighborhood structure in the embedding is the
cheapest GraphRAG trick available: adjacent *structure* becomes findable by vector
search, not just adjacent wording. Flag `stale_embedding` by trigger when a node's
local neighborhood changes; regenerate in a background job (on AlloyDB, optionally
in-SQL via `google_ml.embedding()`).

### 13.4 The lexical seam — `kb_lexical_candidates`

Wrapping the lexical leg in its own function is the seam that absorbs every future
upgrade (BM25-style reranking, more languages) without touching callers or fusion.

```sql
CREATE FUNCTION kb_lexical_candidates(q text, k int DEFAULT 20)
RETURNS TABLE (node_id bigint, rank real) AS $$
    WITH tsq AS (
        -- Query language is unknown (and users cross languages): parse under BOTH
        -- configs and OR the tsqueries. Degrades gracefully at 2 languages;
        -- at 4+ switch to detecting the query language instead.
        SELECT websearch_to_tsquery('english_ua', q)
            || websearch_to_tsquery('french_ua',  q) AS query
    )
    SELECT n.id,
           ts_rank_cd(                                   -- cover density: rewards
               setweight(n.tsv, 'A') ||                  -- proximity of query terms
               setweight(coalesce(c.tsv, ''::tsvector), 'C'),
               tsq.query
           ) AS rank
    FROM tsq, node n
    LEFT JOIN node_content c ON c.node_id = n.id
    WHERE n.tsv @@ tsq.query
       OR c.tsv @@ tsq.query
    ORDER BY rank DESC
    LIMIT k;
$$ LANGUAGE sql STABLE;
```

Weighting rationale: a hit in a section heading or summary (weight A) should
dominate a passing body mention (weight C) — the poor man's field-boosted BM25,
using tsvector's built-in A/B/C/D weights.

### 13.5 Chunk leg with lift, and the extended fusion

```sql
CREATE FUNCTION kb_chunk_candidates(query_embedding vector(1536), k int DEFAULT 20)
RETURNS TABLE (node_id bigint, distance real) AS $$
    SELECT c.node_id, min(c.embedding <=> query_embedding) AS distance
    FROM chunk c
    WHERE c.embedding IS NOT NULL
    GROUP BY c.node_id                 -- LIFT: best chunk represents its section
    ORDER BY distance
    LIMIT k;
$$ LANGUAGE sql STABLE;
```

`kb_search` (§8.4) extends from two fused rank lists to four — trigram, lexical,
section-vector, chunk-vector — with the same reciprocal-rank-fusion pattern
(`sum(1.0 / (60 + rnk))` per node across lists). Everything downstream is
unchanged: fused entry nodes feed the same temporally-filtered graph expansion,
provenance joins, and citation output. Chunks exist only for the milliseconds
between ANN lookup and lift.

### 13.6 How search composes with the browse-first UX

Search is just *another way to acquire a focus node*: whether the user clicked a
node in the graph or typed a query, the result is a focus section + its rendered
neighborhood, and traversal proceeds identically. Two UX rules worth enforcing:

- **Full-text hits land on section nodes** — exactly the right graph entry point
  (user lands on §3.2, sees its supersession edges, and reaches the document
  through `section.document_id`).
- **Superseded results explain themselves rather than hide.** If a hit's node is
  superseded as of the browsing date, show it with the supersession edge rendered
  ("§3.2 — superseded by Bulletin D2 since 2024-01-01"). Search results that
  announce their own staleness are the temporal-fidelity feature made visible;
  the edges already carry everything needed to render it.

### 13.7 Managed-GCP notes (Cloud SQL → AlloyDB)

- Everything in this section — `unaccent`, `pg_trgm`, custom text-search configs,
  pgvector HNSW — is supported on both Cloud SQL and AlloyDB.
- True in-database BM25 (ParadeDB `pg_search`, VectorChord-bm25) is **not** on
  either service's extension allowlist. The pragmatic rank-quality upgrade path on
  managed GCP is a **rerank step behind the seam**: retrieve a generous candidate
  set (ts_rank_cd + trigram), rerank top ~50 with a cross-encoder — on AlloyDB
  optionally in-SQL via `google_ml_integration` semantic ranking. Reranking often
  beats BM25 on quality anyway.
- On AlloyDB migration: swap chunk/node HNSW indexes for ScaNN
  (`num_leaves ≈ √row_count`); queries are unchanged (same pgvector `<=>`
  operator). Keep embedding model + dimensions consistent across the migration so
  vectors survive without regeneration.

### 13.8 Language verification (the "assume multilingual, verify later" plan)

After the first ingestion batch:

```sql
SELECT lang, count(*), avg(lang_confidence)
FROM node_content GROUP BY lang ORDER BY count(*) DESC;
```

Low-confidence rows are already in the review queue. If the document set proves
all-English: update `lang` on the affected rows (the generated tsv recomputes
itself), drop nothing, and the multilingual embedding model remains the right
choice. If more languages appear: add a text-search config per language (one
`CREATE TEXT SEARCH CONFIGURATION` + one CASE branch), and revisit the dual-parse
query strategy in favor of query-language detection.

---

### 13.9 Image assets — the fifth retrieval leg

Documents contain images (figures, diagrams, photos, tables rendered as images).
Images are **assets, not graph nodes** — the same rule as chunks: retrieval
apparatus that lifts to the owning section, never a target of relationships.
Bytes live in object storage; Postgres carries a sidecar row per image:

```sql
CREATE TABLE asset (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    node_id      bigint NOT NULL REFERENCES node(id),   -- owning section
    ordinal      int,
    kind         text NOT NULL,        -- 'figure' | 'table_image' | 'photo' | 'scan'
    uri          text NOT NULL,        -- GCS location of the extracted image
    content_hash text NOT NULL,        -- incremental re-processing trigger
    anchor       int4range,            -- position within section content
    caption      text,                 -- from the document, if present
    description  text,                 -- VLM-generated, canonical English
    ocr_text     text,                 -- text found inside the image
    embedding    vector(1408),         -- optional: multimodal embedding (defer)
    tsv          tsvector GENERATED ALWAYS AS (to_tsvector('english',
                   coalesce(caption,'') || ' ' || coalesce(description,'') || ' ' ||
                   coalesce(ocr_text,''))) STORED
);
CREATE INDEX asset_tsv_idx  ON asset USING gin (tsv);
CREATE INDEX asset_node_idx ON asset (node_id);
```

Pipeline: Stage 1 parsing extracts images, positions, and captions (Docling /
Unstructured handle this). A **VLM pass** — the one new step — produces two text
derivatives per image: a canonical-English `description` and `ocr_text` for words
inside the image. The image thereby becomes *searchable text*, and the entire
existing search stack applies unmodified:

- The asset `tsv` (and optionally a multimodal embedding) is a **fifth candidate
  leg** in `kb_search` fusion — matched at asset granularity, lifted to the
  owning section with the identical `GROUP BY node_id` move chunks use. The
  granularity ladder is now asset/chunk → section → document; every rung climbs
  with the same aggregate-and-keep-best pattern.
- `anchor` lets results render as "section §3.2, matching figure highlighted."
- **Defer the multimodal-vector leg** until image-similarity search ("find
  figures that look like this") is actually requested: for technical documents,
  the VLM description indexed as text captures most retrieval value, because
  people search in words. The `embedding` column is a cheap later addition
  (Vertex multimodal embedding models; note the different dimension).
- Descriptions and OCR used only for search need **no curation** — same rule as
  summaries: retrieval apparatus, not asserted fact.
- Incremental ingestion extends for free: unchanged `asset.content_hash` in a
  changed section skips the (expensive) VLM call; §15 reconciliation treats
  assets as owned derivatives of their section, like chunks.

Scope note: this covers images *in* documents. Fully scanned documents (every
page an image) are the same machinery at higher intensity — Stage 1 becomes
OCR-first parsing. Photos/video as *primary sources* (e.g., inspection photos)
are a new source class requiring a deliberate decision under the guarded-
expansion test, not a default extension.

---

## 14. Extraction pipeline — LLM-probabilistic, human-curated, section-grained

Extraction produces a **three-layer graph**, each layer built by a different
mechanism at a different confidence level. The design rule underneath:
**determinism for structure, probability for semantics** — and every probabilistic
claim gets pinned to a deterministic anchor (a section, a span, a verbatim
sentence) a human can verify in seconds.

| Layer | Content | Mechanism | Trust |
|-------|---------|-----------|-------|
| 1. Section tree | document + section nodes, hierarchy in columns (§3) | deterministic parser | direct to real tables, no review |
| 2. Domain entities | `entity` nodes + DEFINES/MENTIONS links | LLM extraction + identity resolution | staged; provisional entities always reviewed |
| 3. Relations | SUPERSEDES, SUPPLEMENTS | LLM extraction + reference resolution | staged; gated promotion |

### 14.1 Layer 1 — the section tree (deterministic, LLM-free)

Sections are nodes, **including nested sections**. The hierarchy is
intra-document, tree-shaped and parser-produced, so it lives in **columns on the
section node** (§3), not in edges — decision 0002:

```
node(kind='section', heading='Maintenance',      document_id=SB-102, parent_section_id=NULL,   ordinal=3, path={3},     path_label='3')
node(kind='section', heading='Seal Replacement', document_id=SB-102, parent_section_id=§3,     ordinal=2, path={3,2},   path_label='3.2')
node(kind='section', heading='Torque Specs',     document_id=SB-102, parent_section_id=§3.2,   ordinal=1, path={3,2,1}, path_label='3.2.1')
```

Rendering a document's whole tree is therefore one indexed range scan
(`WHERE document_id = :d ORDER BY path`) with no recursion: `path` already
encodes the hierarchy, and storing it as `int[]` sorts correctly where a text
path would put "10" before "2". A section subtree is a recursive CTE over
`parent_section_id`, to which §12's closure-table optimisation still applies.

- Parsing is a parser's job — never the LLM's. Stage-1 is a **format router**
  with **Docling as the primary parser** (it covers PDF, DOCX, PPTX, XLSX,
  HTML, and images natively — one dependency tree, one normalization codepath,
  one version pinned against the hashes); markitdown is the secondary lane for
  the long tail it handles more gracefully (CSV, plain HTML, odd exports).
  Lane assignment is empirical, not doctrinal: benchmark both on ~20
  representative documents per source during the spike (diff the section
  trees) before collapsing or splitting lanes — the router abstraction absorbs
  either outcome. All lanes emit one contract — normalized **markdown as the
  canonical content form** + section tree + spans + assets (headings → paths
  deterministically; heading-free documents get synthetic sections by
  paragraph clusters, flagged low-confidence).

  **Fallback: the `StructureRecoverer` port** — for documents where
  deterministic layout parsing fails (degeneracy checks: structure collapse —
  1 section over long content; flatness vs length; junk headings; and the
  distinct no-text case — chars/page below floor, which routes to OCR first,
  since no structure-recoverer helps a document with no text).

  **Image documents (scans, image-only PDFs) — the composed flow:** classic
  Docling **with OCR enabled** (Tesseract/EasyOCR/RapidOCR) runs first — OCR
  is non-generative recognition (misreads characters, cannot invent
  paragraphs), so it stays in the quasi-deterministic lane with the
  degeneracy checks as the net. Only on structure/text-quality failure does
  the recoverer fire. Born-structured sources (Confluence XHTML, Coda
  markdown) never touch this machinery — their structure is declared, not
  inferred; embedded images inside their pages are §13.9 assets, a separate
  path. Expectation-setting: scan-heavy sources run systematically
  lower-confidence through the whole pipeline (worse spans, more review load)
  — record scan-heaviness per source; the curation-capacity estimate (§19.7)
  is sensitive to it.

  **Recoverer implementations (three, behind one port):**
  1. **Google Document AI Layout Parser — MVP primary.** Inside the GCP trust
     boundary (C-8), OCR built in (collapses the no-text branch),
     non-generative (vision-model layout classification, closer to
     deterministic than an LLM-composed tree), per-page pricing with zero
     infrastructure — a config, not a deployment.
  2. **granite-docling (258M VLM, local).** Plugs into Docling as its VLM
     pipeline; end-to-end page image → DocTags structure. Strengths: complex/
     unusual layouts, equations, code; **no network after the one-time weight
     download** (~0.5GB bf16) — C-8 satisfied with no external call at all.
     Trust category differs: a *generative* model composing structure (can
     drop content or emit plausible-but-absent structure the degeneracy
     metrics won't catch) — which the port's contract already handles
     (low-confidence → review). **Deployment economics — the model is tiny;
     serving compute is the cost:** ~0.35s/page on GPU with vLLM vs. tens of
     seconds/page on CPU. For the async, low-volume fallback lane: CPU
     inference in-process is free-and-fine first; Cloud Run GPU (L4)
     scale-to-zero if volume grows; a standing GPU node is ruled out by
     NFR-9.1 (idle infrastructure for an occasional lane). At MVP volumes,
     Document AI's per-page pricing beats any GPU standing cost — revisit
     with measured fallback volume from `ingestion_run.stats`.
  3. **Roll-your-own Gemini** structured-output tree proposer via the
     provider seam — the documented DIY option.

  Self-hosted open-source PageIndex remains reference material for
  tree-merging technique; the PageIndex *cloud* SDK is excluded by C-8
  (content leaves the boundary). Spike bake-off: run Document AI and
  granite-docling (CPU — costs nothing to evaluate) side-by-side over the
  documents the degeneracy checks flag; curator judgment on recovered trees +
  cost + latency decide the primary. All fallback output is flagged
  low-confidence structure → **review, not the trusted path**; router verdict
  + parser version are recorded in `ingestion_run.stats` (mass-churn
  investigations need parser provenance), and the Governance Console offers a
  per-document "re-parse with fallback" override — each override is a labeled
  example for tuning the degeneracy thresholds (per-source profiles; start
  conservative, widen from logged metric distributions).

  Deterministic output writes **directly to the real tables**, bypassing
  staging: don't burn review capacity on "does §3.2 exist."
- Each section node gets: `heading`, `document_id`, `parent_section_id`,
  `ordinal`, `path`/`path_label` (all real columns, §3), a `node_content` row
  (body text, language detection, content hash), chunks + embeddings (§13), and
  provenance spans against the source file.
- The one LLM touch in this layer: the canonical-English `summary` per section —
  cheap, and what makes section nodes legible in the graph UI.
- **Placement decision:** because the tree is columns, it is outside the graph
  entirely and no `graph_id` question arises for it. Every `edge` row is
  therefore an asserted fact that a human approved or a threshold auto-approved,
  which keeps §6's covering indexes free of one structural row per section and
  stops `kb_search`'s untyped expansion surfacing sibling sections as findings.
  Intra-document cross-references ("see §4.1") are `MENTIONS` edges, and remain
  good `graph_id` candidates if they are ever scoped.

Catalog additions:

```sql
-- document/section/entity are seeded in §2 as the v1 vocabulary (decision 0002).

-- reconcile the two document concepts: provenance file vs. knowledge node
ALTER TABLE document ADD COLUMN node_id bigint REFERENCES node(id);
```

### 14.2 Assert edges at the level the source states them; roll up by view

If a bulletin says "supersedes SB-102 §3.2", the edge targets the **section** node —
a document-level edge would be a false claim (the rest of SB-102 stays valid). If
it says "supersedes SB-102 in its entirety", the edge targets the document node.
Never store both; the document-level display is **derived**:

```sql
CREATE VIEW doc_level_relations AS
SELECT
    src_doc.id   AS src_document,
    dst_doc.id   AS dst_document,
    e.rel_type,
    e.validity,
    e.id         AS underlying_edge,
    dst_sec.id   AS via_section,
    dst_sec.path_label AS section_path,
    (dst_sec.id IS NULL) AS whole_document
FROM edge e
JOIN node src_any ON src_any.id = e.src
JOIN node src_doc ON src_doc.id = coalesce(src_any.document_id, e.src)
                 AND src_doc.kind = 'document'
JOIN node dst_any ON dst_any.id = e.dst
LEFT JOIN node dst_sec ON dst_sec.id = e.dst AND dst_sec.kind = 'section'
JOIN node dst_doc ON dst_doc.id = coalesce(dst_any.document_id, e.dst)
                 AND dst_doc.kind = 'document'
WHERE e.rel_type IN ('SUPERSEDES', 'SUPPLEMENTS');
```

The document browser renders "D2 supersedes D1 *(partially: §3.2)*" from this view;
`whole_document` distinguishes full from partial; `via_section` is the drill-down.
Derived-not-stored means it can never contradict section-level truth. With
structure in columns (decision 0002) the two self-joins through `CONTAINS` that
this view previously needed collapse into `coalesce(node.document_id, e.src)`.
Note:
"document *effectively* superseded" (all sections individually superseded) is a
judgment, not an aggregation — surface "5 of 7 sections superseded" and let humans
or agents conclude. The no-overlap exclusion constraint now operates **per
section**, which is exactly right: §3.2 and §4.1 can be superseded independently.

### 14.3 Layer 2 — domain entities (the refinement layer)

The things sections are *about* become nodes — **one** node per real-world entity,
regardless of how many documents mention it:

```
(§3.2 of SB-102) -[:DEFINES]->  (XJ-42)
(§1.4 of SB-107) -[:MENTIONS]-> (XJ-42)
(bulletin D2)    -[:SUPERSEDES {validity}]-> (§3.2 of SB-102)
```

This is what makes agent queries like "everything about XJ-42, current as of
today" possible: all sections across all documents that refer to the entity, each
section's supersession status, and the entity's own supersession chain — in one
traversal.

`DEFINES` and `MENTIONS` are seeded in §2 with the rest of the v1 vocabulary
(decision 0002). Two notes on what changed and why:

- **`DESCRIBES` was dropped in favour of `DEFINES`.** Where something is defined
  is a knowledge claim with an authoritative answer that supersession can
  change; what merely discusses something is a retrieval signal, and hybrid
  search (§13) already serves that. Accepted cost: FR-8.9 wants "substantially
  about" ranked above "mentioned in passing", so that ranking input is deferred.
- **`REPLACES` was dropped**, folded into same-category `SUPERSEDES`. The
  extractor emits mentions rather than node ids (§14.4), so it cannot know
  whether a target resolves to a document or an entity; asking it to choose
  between two relationship names would be asking it to predict its own
  resolution outcome. One `SUPERSEDES` lets resolution and the trigger decide.

```sql
-- entity lifecycle for auto-created entities
ALTER TABLE node ADD COLUMN status text NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','provisional','retired'));
```

DEFINES vs MENTIONS matters for retrieval ranking — weight them differently when
expanding from entity nodes.

### 14.4 The per-section extraction contract

Layers 2 and 3 come from **one LLM call per section** with structured output. The
prompt is assembled from the catalogs — extractor vocabulary and database cannot
drift:

```
SYSTEM: Extract entities and relationships from a document section.

Entity kinds (generated from:
  SELECT name, description FROM node_kind WHERE NOT is_relation):
  - component: ...
  - assembly: ...

Relationship types (generated from:
  SELECT name, description, src_kinds, dst_kinds FROM rel_type):
  - SUPERSEDES: src supersedes dst during validity. Direction: superseder -> superseded.
  - SUPPLEMENTS / DEFINES / MENTIONS: ...

Rules:
- NEVER invent identifiers. Report references EXACTLY as written (verbatim required).
- Dates: report as written PLUS ISO-8601 if unambiguous; else leave iso fields null.
- Source text may be English or French; verbatim stays in the source language.
- If unsure whether a relationship is stated, include it with confidence < 0.5
  rather than omitting.

USER: Document: SB-107 | Section 2.1 "Component Updates" | <section text>
```

Output schema (maps 1:1 onto `staging_node` / `staging_edge`):

```json
{
  "entities": [
    {"ref": "e1", "kind": "component", "name": "XJ-42",
     "aliases": ["seal XJ-42", "the XJ-42 seal"],
     "verbatim": "seal assembly XJ-42", "span": [214, 236], "confidence": 0.95}
  ],
  "relations": [
    {"rel_type": "SUPERSEDES",
     "src": {"kind": "document_ref", "doc_ref": "SB-107", "section_ref": null},
     "dst": {"kind": "document_ref", "doc_ref": "SB-102", "section_ref": "3.2"},
     "verbatim": "supersedes SB-102 §3.2 effective 1 March 2024",
     "effective_from": "2024-03-01", "effective_to": null,
     "span": [88, 142], "confidence": 0.9},
    {"rel_type": "SUPERSEDES",
     "src": {"kind": "entity_ref", "ref": "e1"},
     "dst": {"kind": "entity_ref", "name": "XJ-40"},
     "verbatim": "XJ-42 replaces the discontinued XJ-40",
     "effective_from": "2024-03-01", "effective_to": null,
     "span": [240, 277], "confidence": 0.85}
  ]
}
```

**The vocabulary is closed at extraction time — the LLM never invents type
names.** `HAS_SUPPLIER` vs `SUPPLIED_BY` vs `supplier_of` was decided once, by a
human, when the hub kind entered the catalog; extraction can only select from
the menu. Open-vocabulary extraction is how the same relation ends up as three
unqueryable variants across documents — the catalog is the anti-chaos mechanism,
and its price is that **new relation shapes are a governance act, not an
extraction outcome** (one INSERT into the catalogs + hub_role; no migration).

When the source states a relationship the vocabulary does not cover, the
contract is: never improvise a type. Extraction emits it into the designated
overflow — `rel_type = 'UNCLASSIFIED'` in staging, verbatim and span attached —
together with a **structured type suggestion**: the agent's proposed name and
inferred shape, which is input to the human's decision, never schema by default.
(Naming note: `UNDEFINED` was considered; `UNCLASSIFIED` was kept because it
describes the *instance's* pending state — the row awaits classification, and a
healthy share of these rows end dismissed rather than defined.)

```sql
ALTER TABLE staging_edge ADD COLUMN suggested_type text;   -- agent's proposed name
ALTER TABLE staging_edge ADD COLUMN suggestion jsonb;      -- structured proposal:
-- {"name": "REQUIRES", "direction": "requirer->required",
--  "src_kind_guess": "entity", "dst_kind_guess": "entity",
--  "is_temporal_guess": false, "rationale": "text states a dependency, which no
--   v1 type covers"}
-- populated ONLY when rel_type = 'UNCLASSIFIED'
ALTER TABLE staging_edge ADD COLUMN resolved_to text REFERENCES rel_type(name);
```

The curation queue exposes a "recurring unclassified patterns" view, clustering
rows by `suggested_type` (trigram-grouped, so REQUIRES/REQUIRED_BY cluster
together) with occurrence counts, document spread, and sample verbatims. The
curator's four outcomes — MECE by design (catalog changes? × was the agent's
framing right?):

| # | Outcome | Catalog change | Meaning |
|---|---------|----------------|---------|
| 1 | **Accept suggestion** | INSERT (agent-shaped) | proposal is right; suggestion JSON pre-fills the new `rel_type`; curator polishes the two-question description |
| 2 | **Map to new type** | INSERT (human-shaped) | pattern is real, agent's framing is wrong; curator authors the type from scratch, informed by the cluster |
| 3 | **Map to existing type** | none | paraphrase of existing vocabulary — also a signal the type's description has a gap the extractor fell through; fix the description |
| 4 | **Dismiss** | none | not worth a type; recorded with reason (extractor regression data) |

Outcomes 1–2 converge mechanically: catalog INSERT (under NFR-7.4
vocabulary-change review), affected sections **re-extract** under the updated
catalog (the original rows were extracted without the type in the prompt, so
re-extraction catches missed instances and applies the ratified semantics
uniformly), instances arrive through the normal gate as typed rows, and the
UNCLASSIFIED rows close with `resolved_to` — preserving the audit trail of how
the vocabulary grew. **Vocabulary metrics fall out of the outcome mix**: the
1:2 ratio measures the extractor's ontological instincts; volume of 3 measures
description quality.

Vocabulary provenance: the catalog records how each type came to exist —

```sql
ALTER TABLE rel_type ADD COLUMN origin text NOT NULL DEFAULT 'human-authored'
    CHECK (origin IN ('seed','human-authored','agent-suggested','human-remapped'));
ALTER TABLE rel_type ADD COLUMN motivated_by jsonb;  -- pointer to the UNCLASSIFIED
                                                     -- cluster that motivated it
```

so the vocabulary is self-documenting with the same evidence discipline as the
facts it governs.

This turns vocabulary evolution into an evidence-driven loop: the document set itself
surfaces which relation types are worth adding, without silent loss and without
vocabulary pollution. UNCLASSIFIED rows are never promoted.

Three load-bearing contract details:

1. **verbatim + span are mandatory** — they make review a five-second check
   (exact sentence highlighted in context) and populate `assertion.span`.
2. **References are never node IDs** — the LLM emits *mentions*
   (`doc_ref`/`section_ref`/entity names); a deterministic resolver maps them to
   nodes. Letting the model guess IDs is how hallucinated edges are born.
3. **Open-ended temporality flows through**: stated start + no stated end →
   `daterange(start, NULL)` at promotion. The extraction contract and the
   temporal model (§10) agree by construction.

### 14.5 Resolution — references and identities

Two resolver problems, same tiered pattern:

**Reference resolution** (which node is "SB-102 §3.2"?): exact document-identifier
match → trigram fuzzy → embedding similarity; within the target document, resolve
sections by `path` first, heading trigram second. **Unresolved mentions are
first-class**, not failures — "supersedes a document we haven't ingested yet" is
real information. Park with `status='unresolved'`; re-run resolution when new
documents arrive (§15.5 makes this automatic). Ingestion order therefore does not
matter.

**Identity resolution** (is "XJ-42" here the same node as "seal XJ-42" from last
month?): exact normalized-identifier match → alias match (accumulate `aliases`
into `node.props.aliases`, trigram-indexed) → embedding similarity above a high
threshold → **else create a new node with `status='provisional'`**. Provisional
entities always surface in curation ("new entity — confirm or merge"): duplicate
prevention is cheap now, expensive later.

Each tier records *how* it resolved and with what confidence into
`staging_edge.props`. **Resolution confidence multiplies with extraction
confidence** to gate promotion: a 0.95 extraction on a tier-1 exact match can
auto-approve; a 0.85 extraction resolved by embedding similarity should not.

**Hard floor — lifecycle-mutating relationships never auto-approve on fuzzy
resolutions.** Rel_types that mutate an entity's lifecycle (SUPERSEDES,
anything that closes validity or fires temporal integrity
constraints) MUST be human-reviewed whenever either endpoint resolved at
tier 3 (embedding) or tier 4 (LLM-ranked), regardless of score. Rationale:
the risk is asymmetric — a wrong MENTIONS link adds retrieval noise and
self-heals; a wrong SUPERSEDES **falsifies history for the wrong entity**,
closing its validity and changing every future as-of answer about it. Fuzzy
identity × destructive semantics = human, always. This is a floor beneath
the tunable (rel_type × tier) threshold table, not a tunable itself.

Supporting mechanics for vague references ("supersedes the legacy high-temp
seal" where the graph knows "XJ-40 Seal Assembly"):

- **Enriched mentions**: the extraction contract requires the LLM to emit the
  most specific referent the section supports — name form, kind guess, and
  qualifiers gathered via intra-section coreference — so the resolver
  receives a description, not just a phrase.
- **Tier 4 (LLM ranker, escalation only)**: when tiers 1–3 leave multiple
  plausible candidates, an LLM ranks a *shortlist* (mention + qualifiers +
  top-k candidate dossier summaries — never documents, never the document set) and
  proposes a match with rationale; output is always a low-confidence
  resolution routed to review, never auto-promoted.
- **Alias accretion on approval**: when a curator confirms a vague phrasing
  resolves to a node, the phrasing joins that node's aliases — the next
  occurrence resolves at tier 2. Every adjudication makes the next one
  cheaper (the §17.4 fail-improve loop operating on identity).
- **Unresolved-pending is the honest terminal state** when even the curator
  cannot disambiguate: the relationship asserts against no one (better no
  edge than a falsifying one) and surfaces through gap-aware answers (§17.3)
  as an unratified claim against an ambiguous target.

```sql
ALTER TABLE staging_edge ADD COLUMN src_ref jsonb;   -- the mention, verbatim
ALTER TABLE staging_edge ADD COLUMN dst_ref jsonb;
ALTER TABLE staging_edge ADD COLUMN resolution jsonb; -- {tier, method, score} per endpoint

-- merge function: merging without one corrupts provenance
CREATE FUNCTION kb_merge_nodes(survivor bigint, duplicate bigint) RETURNS void AS $$
BEGIN
    UPDATE edge SET src = survivor WHERE src = duplicate;
    UPDATE edge SET dst = survivor WHERE dst = duplicate;
    UPDATE edge SET graph_id = survivor WHERE graph_id = duplicate;
    UPDATE assertion SET node_id = survivor WHERE node_id = duplicate;
    UPDATE node SET props = jsonb_set(props, '{aliases}',
        coalesce(props->'aliases','[]'::jsonb) ||
        coalesce((SELECT props->'aliases' FROM node WHERE id = duplicate),'[]'::jsonb) ||
        to_jsonb((SELECT ARRAY[name] FROM node WHERE id = duplicate)))
      WHERE id = survivor;
    UPDATE node SET status = 'retired',
                    props = jsonb_set(props, '{merged_into}', to_jsonb(survivor))
      WHERE id = duplicate;
END $$ LANGUAGE plpgsql;
```

(Retire, don't delete — history and inbound references must survive. Depending on
edge-level unique constraints, repointing may collide with existing edges; handle
per rel_type inside the function as those constraints accumulate.)

### 14.6 Curation routing — effort where probability lives

| What | Review policy |
|------|---------------|
| Layer 1 structure | none |
| DEFINES/MENTIONS links | spot-check; auto-approve above threshold (low-harm, retrieval-redundant) |
| New provisional entities | always review (dedup) |
| Layer 3 relations, esp. temporal/SUPERSEDES | start at 100% review; loosen per (rel_type × resolution-tier) as the queue's own accept/reject history proves precision — subject to the §14.5 hard floor: lifecycle-mutating rel_types on tier-3/4 resolutions never auto-approve |

Review UI needs exactly three things per item: the verbatim sentence highlighted
in section context, the proposed graph delta rendered visually, and
approve / reject / edit-target. **Keep rejections with reasons** — they are the
prompt-regression test set and future fine-tuning data. The review queue is a
labeled dataset being built as a side effect of operation.

---

### 14.7 Knowledge-bearing images

Some images *contain the payload* — a superseded-part-number table rendered as a
PNG, a labeled diagram encoding effective dates. If that content only becomes
OCR text it is findable but not *known*: no SUPERSEDES edge, no temporal
integrity, no as-of correctness — the exact failure Portolan exists to prevent,
hiding inside an image.

Extension to the extraction pass:

- Sections whose assets look knowledge-bearing (tables, labeled diagrams — the
  asset `kind` from §13.9 is the first filter) include the VLM output
  (description + OCR) in the per-section extraction call (§14.4).
- Claims extracted from an image flow through the **same** staging → mention
  resolution → curation gate as text-extracted claims. No separate pipeline.
- Provenance points at the asset:

```sql
ALTER TABLE assertion ADD COLUMN asset_id bigint REFERENCES asset(id);
-- sibling to section_node_id; an assertion may cite a section span OR an asset
```

- **Curation bar: lower auto-approve thresholds / review more** for
  image-sourced claims. OCR of part numbers is precisely where transcription
  errors produce plausible-looking wrong edges, and a wrong SUPERSEDES is the
  costliest mistake in the system. Treat image-extraction as its own
  (rel_type × source-modality) slice in the threshold model of §14.6, earning
  relaxation from its own measured precision.
- Review UI: the reviewer sees the image itself alongside the proposed graph
  delta — the visual is the verbatim here.

---

## 15. Incremental ingestion — change detection & the watcher

The section is the **unit of change**; extraction, embeddings, and review are
already keyed to it, so incremental ingestion reduces to diff-and-reconcile with a
hash hierarchy on top. Goal: re-ingesting an unchanged document set costs ~zero; a
one-section edit re-processes one section.

### 15.1 Change detection — hash hierarchy, cheapest check first

```sql
ALTER TABLE document ADD COLUMN structure_hash text;  -- hash of ordered (path, heading, section_hash) list
ALTER TABLE document ADD COLUMN last_checked timestamptz;
```

Decision cascade:

1. File `content_hash` unchanged → touch `last_checked`, **done** (the 99% path —
   must be free).
2. Changed → re-run Stage 1 parsing (deterministic, cheap, no LLM) → compare
   per-section `node_content.content_hash`.
3. Only changed sections proceed to re-extraction.

**Hash normalized text** (whitespace/encoding-folded): a re-exported PDF with
identical content must be a no-op even when bytes differ, or every regeneration
triggers a full rebuild and the scheme is pointless.

### 15.2 Section reconciliation — tiered matching, ordered by strength

**Ownership ruling: Portolan reconciles, not Textus.** The diff's inputs are
Portolan's stored tree; its outputs are trusted-graph mutations (section
`path`/`parent_section_id` updated, sections retired, assertions staled, edges preserved across moves) —
mutation authority belongs to the custodian, atomically with the state it
mutates, in one implementation serving every submitter. A client *declaring*
these mutations would let an extractor bug silently reattach edges to wrong
text — the exact misattribution the tiers exist to prevent. Portolan also holds
everything the diff needs: the old tree in its tables, the new side as
metadata + content-addressed references (§18.2) — no client-side matching mode
exists. Mechanically: reconciliation runs at structure-submission time, and
**the submission response is the reconciliation report** ("changed: [3.2, 5];
added: [6]; moved: [4.x→5.x, edges preserved]") — precisely the input the
extractor needs to scope extraction. Textus's only diff is the document-level
normalized-hash short-circuit. The matcher itself is a **pure, side-effect-free
module inside core** — tree-diff logic exhaustively unit-testable against
synthetic renumbering nightmares without a database in the loop; applying its
output is the transactional part.

New parse tree vs. stored tree; misordered tiers cause either mass churn or silent
misattribution:

| Tier | Match on | Meaning | Action |
|------|----------|---------|--------|
| 1 | content hash (any path) | moved/renumbered section | update the section row's `path`/`ordinal`/`parent_section_id`; **keep node, edges, assertions, embeddings, chunks; zero re-extraction** |
| 2 | path + fuzzy heading, content changed | edited in place | the re-extract set (§15.3) |
| 3 | heading/content similarity at different path | moved AND edited | match if similarity high; else remove+add and queue "possible section move" for review |
| — | unmatched old | removed | mark node `retired` (never delete); queue |
| — | unmatched new | added | full Layer 1–3 treatment |

Tier 1 is what survives the classic nightmare — a revision inserts a new §2 and
everything below renumbers from §3.x to §4.x; hash matching sails through what
path matching would misread as "everything deleted, everything new."

### 15.3 Scoped invalidation — exactly what a changed section dirties

Prerequisite — provenance must record *which section* supported each fact:

```sql
ALTER TABLE assertion ADD COLUMN section_node_id bigint REFERENCES node(id);
CREATE INDEX assertion_section_idx ON assertion (section_node_id)
    WHERE section_node_id IS NOT NULL;
```

For a Tier-2 changed section:

- **Content derivatives**: re-summarize, re-embed, re-chunk (the `content_hash`
  trigger from §13 already covers chunks).
- **Edges extracted FROM the section**
  (`SELECT edge_id FROM assertion WHERE section_node_id = :changed`): re-extract,
  then diff proposed vs existing —
  - identical claim re-found → refresh the assertion, no review;
  - claim no longer present → **do not auto-delete**: mark the assertion `stale`,
    queue "source no longer supports this edge." A temporal SUPERSEDES fact may
    remain *true* after the sentence stating it is edited away — auto-deleting
    confirmed knowledge because a revision tightened its wording is the
    incremental-pipeline footgun;
  - new claims → normal staging/gate flow.
- **Edges pointing AT the section** (D2 SUPERSEDES §3.2): **not invalidated** —
  D2's claim is asserted from D2's text. But content change at a supersession
  target is high-signal: emit a low-priority review item ("§3.2 revised; 2 inbound
  SUPERSEDES edges reference it").
- **Entity links** (DEFINES/MENTIONS): recomputed with re-extraction; entity
  *nodes* untouched (XJ-42 doesn't vanish because one section stopped mentioning
  it — it loses one link).

```sql
ALTER TABLE assertion
    ADD CONSTRAINT assertion_status_chk CHECK
    (status IN ('pending','approved','rejected','stale'));  -- replaces the §4 check
```

### 15.4 The watcher — ledger, triggering, idempotency

```sql
CREATE TABLE ingestion_run (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    document_id bigint NOT NULL REFERENCES document(id),
    started_at  timestamptz NOT NULL DEFAULT now(),
    finished_at timestamptz,
    outcome     text,     -- unchanged | updated | new | failed
    stats       jsonb     -- {sections_matched: 41, moved: 3, changed: 2, added: 1,
                          --  removed: 0, edges_invalidated: 5}
);
CREATE INDEX ingestion_run_doc_idx ON ingestion_run (document_id, started_at DESC);
```

Design points (the loop itself is trivial — a small Python service):

- **Triggering**: GCS bucket notifications → Pub/Sub for cloud-storage sources
  (event-driven, near-zero idle); a polling loop ordered by `last_checked`
  staleness otherwise. Both feed the same per-document pipeline.
- **Phases with different transactionality**: hash check → parse + reconcile in
  **one transaction** (the section tree must never be observably half-updated) →
  per-section re-extraction as **independent retryable jobs** (LLM calls fail; a
  failed section must not roll back forty reconciled ones).
- **Idempotency is free from the hash design**: re-running an already-processed
  document exits at the `content_hash` match.
- **Concurrency**: `pg_advisory_xact_lock(document_id)` so two watcher instances
  cannot race on one file.
- **`stats` is the sanity alarm**: `removed: 38, added: 41` on a minor revision
  means Tier-1 matching broke (parser or normalization change) — alert *before*
  promotion, not after.

### 15.5 Re-resolution for free

When a new document lands, the same watcher pass re-runs reference resolution over
parked `unresolved` mentions (§14.5) — "new nodes exist" is just another change
event. This is the incremental-evolution loop closed: ingestion order never
matters.

### 15.6 Version semantics — policy, not heuristic (decide per source)

A changed file means one of two different things, and a hash cannot tell them
apart:

- **In-place revision** (same document, corrected): the reconciliation flow above.
  Right for genuinely mutable sources — wikis, living specs.
- **New revision that supersedes** (SB-102 Rev C): a **new document node** +
  auto-proposed whole-document `SUPERSEDES` edge to its predecessor (into the
  review queue), predecessor's sections intact and temporally closed. Right for
  supersession-heavy, revision-lettered domains.

Cleanest rule: key documents by *logical identity + revision* where the domain
provides one (bulletin number + rev letter → new revision = new document). Make
the regime a **per-source configuration** on the watcher, not an inference. The
temporal model (§10) handles both.

### 15.7 Source connectors — template, and SharePoint in detail

**The connector template (one interface, N implementations).** All source
access lives in the watcher; parsers receive bytes on disk and never hold
credentials — secrets exist in exactly one component, and a parser bug's blast
radius never includes source-system access. Every connector implements the
same contract:

1. **Identity**: a service principal (machine credential) with a
   least-privilege, per-source grant — never a borrowed human account.
2. **Fetch-and-mirror**: on detected change, fetch the artifact once, write it
   to the **artifact store** content-addressed (key = content hash) — GCS in
   the cloud profile, a local object directory (`~/.portolan/objects/<hash>`)
   in the personal profile (§20) — and hand the local file to Stage 1. Source systems are touched for detection and
   fetching only — never re-processing (§15.1).
3. **Delta cursor**: use the source's native change-detection affordance where
   one exists; persist its cursor/token in the per-source configuration
   (§15.6's config record grows: connector type, credentials *reference* —
   Secret Manager path, never the secret itself per NFR-6.4 — delta cursor,
   version-semantics policy, sensitivity tier, ACL label default).
4. **Cheap pre-checks before download**: source metadata (modified timestamps,
   etags) gates fetching; the normalized content hash remains the sole
   authority on whether anything actually changed — source systems touch
   metadata liberally, and metadata-only churn must stay free.

**Confluence Cloud, concretely (REST v2 + CQL polling):**

1. *Identity*: pragmatic standard — a **dedicated service account**
   (`svc-portolan@...`) with an **API token**, Basic auth (`email:token`);
   least privilege via space permissions granting that account read on only
   the watched spaces. Ceremonious alternative: an OAuth 2.0 (3LO) app with
   read-content scopes and rotating tokens. Token in Secret Manager,
   referenced from source config. (Confluence *Data Center* is a different
   connector — different APIs and auth; confirm the flavor per source, same
   caveat as on-prem SharePoint.)
2. *Enumeration & fetch*: REST v2 — list spaces, list pages per space
   (`/wiki/api/v2/spaces/{id}/pages`), fetch each page's **storage-format
   body** (XHTML). No binary parsing: the body flows into the router's HTML
   lane, and Confluence's heading elements yield the section tree nearly for
   free. Attachments (embedded PDFs etc.) come via the attachments endpoint
   and take the binary lanes. Mirror fetched XHTML to GCS content-addressed
   as usual.
3. *Delta cursor*: no delta token — the idiom is **CQL polling**:
   `type=page AND space in (...) AND lastmodified >= "<cursor>"`, cursor =
   last successful poll time, persisted in source config. (Webhooks are
   Forge/Connect-app territory; scheduled CQL fits the watcher MVP.)
4. *Pre-checks & versioning bonus*: `version.number` + `lastModified` gate
   fetching. Pages carry **explicit version numbers** — record in
   `document.props` for provenance; Confluence is the canonical "genuinely
   mutable source" for §15.6's in-place-revision regime, and the version
   number is a cleaner change signal than timestamps alone.
5. *Mapping decision (per-source config)*: what is a "document"? Default:
   **page = document** (matches how people cite Confluence; the page's own
   headings become sections). Alternative for tightly-coupled manuals:
   page-tree = document with pages as sections. Decide per space.

**Coda, concretely (bearer token + export API):**

1. *Identity*: API **bearer tokens** are user-bound — use a **service user**
   owning the token, invited to only the watched docs/folders. Simplest auth
   of the three; Secret Manager as always. Rate limits are the tightest of
   the three sources — pre-checks matter more here.
2. *Enumeration & fetch*: `/docs` lists documents; `/docs/{id}/pages` lists
   pages; the standout: a first-class **export endpoint** returning a page as
   **markdown** (async job: request export, poll, download) — the canonical
   form handed to you directly, shrinking the router's job to normalization.
   Still hash *your* normalization of it, not their raw export — Coda's
   exporter is a version-pinning dependency exactly like markitdown (§15.1
   rule).
3. *Delta cursor*: no delta API — poll `/docs` gated on `updatedAt`, cursor =
   last poll time in source config.
4. *Pre-checks*: `updatedAt` per doc gates page-level enumeration; normalized
   hash remains the authority. Note: Coda tables export as markdown tables —
   if a table is *knowledge-bearing* (e.g., a supersession registry maintained
   as a table), that is the §14.7 pattern in text form; extraction handles it.

**Local directory, concretely (the personal-profile connector):**

1. *Identity*: none — the filesystem is the trust boundary.
2. *Enumeration & hints*: walk configured paths; mtime + size are the cheap
   change hints (they lie under copies/touches — the normalized hash remains
   the authority, same as everywhere).
3. *Change detection*: **filesystem events** (watchdog/inotify) — push-based,
   better than any cloud API's polling; scheduled walks as the catch-up net.
4. *Fetch-and-mirror*: copy into the local artifact store — still required:
   users edit files in place, and without the mirror the as-ingested bytes
   vanish while assertion spans still point at them.

**Google Drive (Docs, Sheets, Slides), concretely:**

1. *Identity & authentication*: a GCP **service account** with the Drive API
   enabled, scope `drive.readonly`. Since Portolan runs on GCP, prefer
   **workload identity** (no key to manage) over a downloaded key; if a key is
   unavoidable, it lives in Secret Manager like every other credential.
   Least-privilege grant: **share the watched folders / Shared Drives with
   the service account's email** — the Drive-native equivalent of
   `Sites.Selected`; avoid domain-wide delegation (user impersonation) unless
   genuinely required — folder sharing is the auditable, minimal grant.
2. *The "no original bytes" ruling*: Google-native files are cloud objects,
   not files — there is nothing to fetch except an **export**. The mirror
   therefore stores *the export actually processed*, content-addressed as
   usual, with provenance recording
   `{export_mime, drive_file_id, drive_version, export_time}` in
   `document.props` — an honest acknowledgment that the mirror is a rendering
   of a cloud object at a version. Drive `version`/revision numbers slot into
   §15.6 exactly like Confluence's: canonical in-place-mutable sources.
3. *Per-format lanes*:
   - **Docs** — the gift lane: Drive export supports **`text/markdown`
     natively**, headings intact for the tree walk. Like Coda, an
     upstream-controlled converter: hash *your* normalization, record the
     export format in provenance. (Docs API structured JSON is the deferred
     deeper integration if character-precise native spans are ever needed.)
   - **Slides** — export PPTX/PDF into the Docling lane, or Slides API JSON.
     Mapping: **one slide = one section** (path = slide number, heading =
     slide title), body + speaker notes as content. Slides are image-heavy by
     nature — the §13.9 asset pipeline (extract, VLM-describe, index) does
     most of the real work; expect this source to exercise asset machinery
     hardest.
   - **Sheets — the deliberate misfit, opt-in per folder, never default-on.**
     Mapping: one sheet-tab = one section, content as a markdown table →
     the §14.7 knowledge-bearing-table pattern (a supersession registry
     maintained as a Sheet is the best reason to ingest Sheets at all).
     Guardrails: a per-source **row/size cap** — small tables ingest whole;
     large data sheets ingest headers + sample, marked content-truncated, or
     skip the tab (10k-row transaction dumps are neither sectionable nor
     extractable, and blow the tsvector ceiling). Most spreadsheets are
     operational data, not knowledge; indiscriminate ingestion is landfill.
4. *Change detection — best in the fleet*: the Drive **Changes API** is a
   true delta feed — fetch `startPageToken` once, each `changes.list` poll
   returns exactly what changed plus a new token; one mechanism across all
   three formats. Pre-download gates: `modifiedTime` + `version`. Drive push
   notifications (webhook channels) are the later event-driven upgrade;
   delta polling is the MVP, same posture as Graph.
5. *Exclusion*: Docs **comments and suggestion threads do not ingest** —
   conversational, unratified content (C-7 territory); exports conveniently
   return only the accepted document state, which is exactly the
   ratified-surface behavior wanted. Decision-laden comment threads are the
   §16 Compositor's hunting ground, not the extractor's.

**Comparative summary:** SharePoint = binary files + best change API (delta
tokens); Confluence = structured XHTML + versioned pages + CQL polling; Coda =
native markdown export + crudest change detection + tightest rate limits;
Google Drive = no original bytes (mirror-the-export) + Changes-token delta
feed (best of all) + three formats of wildly different knowledge density
(Docs rich, Slides image-heavy, Sheets opt-in);
local directory = fs events + no auth + in-place
edits (mirror mandatory). All
five fit the template unbent — service principal, fetch-and-mirror, cursor in
source config, hash as authority — which is the template doing its job: five
connectors, one contract, and the parser never knows which world a document
came from.

**SharePoint Online, concretely (Microsoft Graph, app-only):**

1. *App registration* (Entra ID): register an application; request the Graph
   **application permission `Sites.Selected`** (least privilege — access only
   to explicitly granted site collections, not `Sites.Read.All`). Tenant
   admin consents, then grants the app read on each watched site. Credential:
   **certificate** over client secret (corporate-standard posture); cert
   stored in GCP Secret Manager, referenced from source config.
2. *Token acquisition* (in the watcher): MSAL
   `ConfidentialClientApplication` with the certificate, client-credentials
   flow, scope `https://graph.microsoft.com/.default`. No user involved — the
   watcher is a service principal with its own explicit grant (§18 machine
   identity), not a borrowed human.
3. *Enumeration & fetch*: Graph drive endpoints —
   `/sites/{site-id}/drive/root/children` to enumerate,
   `/sites/{site-id}/drive/items/{item-id}/content` to download; bytes go to
   the GCS mirror, then to the parser.
4. *Change detection*: the **delta query** —
   `/sites/{site-id}/drive/root/delta` returns only items changed since the
   last delta token; persist the token per source and each poll costs one
   call. `lastModifiedDateTime`/`eTag` provide the pre-download gate.
   Graph change-notification webhooks are the later push upgrade; scheduled
   delta polling is the MVP.
5. *Boundary check*: this is SharePoint **Online**. On-prem SharePoint
   (2016/2019) has no Graph — older REST APIs, NTLM/ADFS — a different and
   worse connector; confirm which flavor each source is before committing.

---

## 16. Conversational-source formalization — decisions from meetings, email, Slack

Conversational sources (email threads, Slack conversations, meeting recordings)
are NOT ingested directly. Knowledge that lives in conversations enters Portolan
through a **formalize → ratify → ingest** loop: an LLM agent detects that
something formalization-worthy occurred, drafts a formal document, a human with
authority ratifies it, and the ratified document enters the normal ingestion
pipeline (§14/§15) as an authoritative source. The epistemic firewall is
preserved: everything in the graph remains ratified truth; transcripts and
threads never become graph facts.

**Division of labor:** the LLM drafts (the labor); the human ratifies (the
authority). Requiring humans to originate the documents recreates the historical
knowledge-base failure mode — capture starves on the cost of writing. Reviewing a
draft costs minutes; the loop survives.

### 16.1 Signal detection — the trigger

The agent's first job is not drafting but classification: did this conversation
produce a **decision**, a **commitment**, or a **material fact change**? Most
conversations produce none, and drafting for them recreates the searchable
landfill.

- Cheap classification runs continuously; drafting runs rarely.
- Detection thresholds are tunable from ratification outcomes — the same
  evidence-driven gate pattern as extraction confidence (§14.6).
- "Conversation done-ness" is per-source: a meeting ends; a thread is done
  heuristically (resolution language, inactivity timeout, explicit markers).
  A reopened thread that reverses itself simply produces a superseding memo —
  the temporal model (§10, §15.6) absorbs it.

**Capture layer (meetings).** Transcription is a solved, adoptable commodity —
**Meetily** (MIT, local Whisper/Parakeet + diarization, OpenAI-compatible
endpoint so it can point at Vertex per C-8) or Whisper directly. It produces
transcript + generic summary: Compositor's *input*, never its output. No
open-source project fills Compositor's actual role (domain-specific decision
detection, Portolan-shaped memo contract, dedup against the graph, ratification
authority) — the ecosystem scan confirmed that middle is empty because every
part of it is deployment-specific.

**Event-log workspaces (Buzz and kin).** Block's Buzz *replaces* Slack/GitHub
rather than connecting to them — no inbound bridges exist, and a bridge would
contradict its model (mirrored messages carry the bridge's signature, not the
speaker's, hollowing out the identity guarantees that make it interesting).
Therefore: if such a workspace is adopted for its own reasons, it becomes
Compositor's ideal *host* — one signed event shape instead of three
heterogeneous connectors, workflow triggers built in, and **approval as a
signed event** (cryptographic ratification evidence, §16.4). Never adopt it
*for* Portolan; conversations that live in Slack/email today are served by
direct connectors under the §15.7 template.

### 16.2 The unit of document — decision-shaped, not transcript-shaped

The durable artifact is one memo per decision/commitment cluster: context, the
decision, effective date, owners — NOT "minutes of the March 4 sync, all topics."
One meeting may yield three memos; ten meetings may yield none.

**The memo template is a first-class, per-source configurable artifact** — not
a hardcoded prompt. Decision memos for engineering syncs and for contract
negotiations want different shapes, and every tool in this space that survives
contact with real users exposes the template. Same treatment as `rel_type`
descriptions: configuration data, versioned, reviewed.

Why decision-shaped:
- Supersession is clean: memo B reversing memo A is a whole-document SUPERSEDES —
  §15.6 machinery verbatim.
- Review is fast: a human ratifies a one-decision memo in minutes.
- Extraction is near-deterministic: the drafting agent writes *to be extractable*,
  sharing the catalog vocabulary (§14.4) with the extractor.
- Cross-channel deduplication happens at drafting time: the same decision
  discussed in Slack and ratified in a meeting is ONE memo citing both sources.
  The drafting agent checks Portolan for existing open decisions on the same
  subject via `kb_search` — Portolan is a consumer of itself here.

### 16.3 Provenance — the chain behind the memo

- Source artifacts (recording, transcript, thread export) live in object storage,
  registered like any source file; the memo's provenance records a DERIVED_FROM
  link to them. Transcripts are never searchable and never in the graph — they
  are the audit anchor, the same role the original PDF plays for extracted facts.
- Draft sections carry spans into the source transcript, so ratification review
  is "claim highlighted against the moment it was said" — fast verification, not
  a memory test.
- The ratifier's identity is recorded in provenance; it is part of what makes the
  document authoritative.

### 16.4 Ratification — authority, not just review

Ratifying a decision memo differs from §14 curation: the curator verifies
*extraction fidelity*; the ratifier asserts *the decision was actually made*.
That assertion is only meaningful from someone with standing — the decision owner
or a participant, never a knowledge-base admin. Drafts route to participants for
approval. Ratification is a distinct role in the access model.

**Ratification must be evidence, not bookkeeping.** Record the ratifier's
verified identity **and the hash of the artifact approved** — sign the thing
you approved. "Approved: true" from a workflow webhook is a claim that a click
happened; identity-plus-artifact-hash is a claim that *this person* approved
*this exact memo*. Cheap to build in from the start, impossible to retrofit
onto historical approvals. (Event-log workspaces give this natively via signed
events, §16.1 — but the property is worth having regardless of host.)

**Prototyping scaffold: n8n.** For phase 2, n8n (self-hosted; note: fair-code
Sustainable Use License, not OSI — check procurement before it is
load-bearing) supplies the grunt work: Slack/Gmail/Drive/Teams connectors
already credentialed and rate-limit-handled, schedule/webhook triggers, retry
branches, and a **human-approval node** that is exactly the ratification-
routing plumbing. The seam that keeps it useful rather than fought:
**orchestration on the canvas, intelligence in a versioned service.** The
workflow is four nodes — `[trigger] → [HTTP: POST conversation to Compositor
service] → [IF: signal detected] → [approval node] → [HTTP: POST ratified memo
to Portolan]` — while the classifier, memo contract, dedup call, and structured
output schema live in a repo where they can be diffed, reviewed, and
regression-tested (NFR-7.5). Prompts in workflow config are prompts that
cannot be tested. Graduation is then a swap, not a rewrite: replace four nodes
with a service loop when volume warrants.

### 16.5 The fused gate — one human touch, not two

A ratified memo would ordinarily re-enter §14 extraction and hit curation again —
double review of the same content. Instead, the drafting agent emits the memo
AND its structured facts together (the SUPERSEDES edge, the commitment hub node,
validity dates), and the ratifier's single approval covers both: the review UI
(§14.6) already renders document + graph delta side by side. Extraction over
agent-authored memos is near-deterministic (author and extractor share the
catalog), so the probabilistic gate adds nothing.

- Formalized memos: **fused gate** (one ratification covers document + facts).
- Documents from outside the formalization loop: unchanged two-step
  (ingest → extract → curate).

### 16.6 Deliberate non-goals

- **Raw conversational ingestion**: transcripts, threads, and recordings never
  become graph facts or searchable knowledge-base content; only ratified memos do.
- **Expertise location** ("who has discussed X across the year"): the
  formalization loop captures *what was decided*, not *who knows what* — an
  aggregate over conversations that no single memo records. If required, it is a
  separate, lighter mechanism (person↔topic links at a lower epistemic tier) and
  a separate decision; do not let it creep into this loop.
- **Full epistemic tiering** (ratified/decided/claimed): human ratification is
  the ratifying act, so this design collapses the tiers back to a uniform
  ratified graph — buying uniform trust semantics at the price of a cheap human
  gate. Tiering becomes relevant only if unratified sources are ever admitted.

### 16.7 New moving parts (summary)

| Part | Nature |
|------|--------|
| Source watchers (email, Slack, recordings) | connectors feeding the signal classifier |
| Signal classifier | cheap continuous LLM classification; threshold tuned from outcomes |
| Drafting agent | composes decision-shaped memo + structured facts + source spans; dedup via kb_search |
| Ratification workflow | routes to participants; records ratifier; single approval promotes document + facts |
| Person entities | ratifiers, owners, and commitment participants become first-class `person` nodes (identity-resolved across email addresses/handles; role changes handled temporally, never by deletion) |
| Consent & visibility controls | per-source consent for monitoring; conversational sources are surveillance-adjacent — designed in, not bolted on |

---

## 17. Trust & maintenance surface — dossiers, explainability, gaps, consolidation

Patterns adopted (with adaptation) from agent-memory systems in the wild —
notably GBrain's compiled-truth/timeline pages, explainable retrieval, and
self-maintenance cycles — filtered through Portolan's governance model. The
recurring adaptation: what those systems do by convention, Portolan does as
derived views over ratified, gated data.

### 17.1 Entity dossiers — compiled truth + evidence timeline (derived view)

Every entity (and document) gets a rendered dossier page, generated from the
graph, never stored as truth:

- **Above the fold — compiled truth:** the entity's current state as-of today:
  name, kind, summary, active relationships (temporal-filtered), rendered as
  prose + structured facts. Regenerated when the underlying facts change
  (the stale-summary trigger already exists for embeddings; dossiers ride it).
- **Below — the evidence timeline:** append-only history derived from
  assertions and temporal edges: when each fact was asserted, from which
  document (citation), when superseded and by what, merges, retirements.

The dossier is the human-facing payoff of provenance + temporality: the
synthesis is **pre-computed** rather than re-derived per query, and it can
never contradict the graph because it is a projection of it. Serve via
`kb_dossier(node_id, as_of)` — same seam rules as everything else.

**Vault export — the personal profile's viewer, nearly free.** Because
dossiers already exist and markdown is already the canonical content form,
exporting a tenant's knowledge as an **Obsidian-compatible vault** is a small
command (`portolan export --vault ./kb`): one `.md` per entity/document
(dossier body), edges as `[[wikilinks]]`, properties as YAML frontmatter,
supersession status as a property. Three payoffs: (1) an **interim viewer** —
Obsidian's graph view + backlinks is the most battle-tested human interface
for browsing a knowledge graph, at zero UI cost, especially for the personal
profile (§20); (2) an **escape hatch** — "your knowledge can always leave as
plain files" is the strongest available answer to the graveyard fear and a
trust anchor for local users; (3) it breaks nothing — the export is a
read-only derived projection of exactly what §19.2 already permits to leave
(metadata, summaries, compiled dossiers), while truth stays in Postgres.

### 17.2 Retrieval explainability — kb_search explain mode

`kb_search(..., explain => true)` returns per-result attribution:

- which legs surfaced it (trigram / lexical / section-vector / chunk-vector /
  asset), each leg's rank, and the fused RRF contribution;
- which boosts applied (heading-weight, DEFINES-vs-MENTIONS weighting);
- an **evidence tag** per result: why it matched (matched name, matched body
  span, semantic-only) — agents use this to calibrate confidence, humans use
  it to debug "why didn't §3.2 surface."

The legs already compute independent ranks; explain mode is exposure, not new
computation. A `kb_search_diagnose(query, target_node)` variant traces which
leg surfaces or misses a specific expected result — the RRF-tuning tool.

Resolution output gets the same treatment: identity/reference resolution
returns its tier (exact / alias / embedding / unresolved) as an
exists-probable-unknown signal, so consumers (the drafting agent, curation UI)
know whether they are citing a known entity or gambling.

### 17.3 Gap-aware answers

Portolan's pipeline leftovers are first-class data — unresolved mentions
(§14.5) and UNCLASSIFIED relations (§14.4) — so answers can state what the
knowledge base does *not* know, honestly and specifically:

> "No ratified fact covers X. Two unresolved references from SB-201 §4
> mention it (ingested 2026-06-02, pending target)."

`kb_search` (and the MCP answer surface) SHOULD include a gaps section
assembled from: unresolved mentions matching the query terms, UNCLASSIFIED
staging rows matching them, and query-relevant sections whose supersession
target is pending review. Calibrated ignorance is a trust feature: an agent
that can say "not known yet, here's the trail" is safer than one that
synthesizes around the hole.

### 17.4 Resolver fail-improve loop

Every tier-3 (embedding-similarity) resolution and every unresolved mention is
a logged signal that tiers 1–2 (exact, alias/trigram) missed. Periodically mine
these for new normalization rules and aliases; promote the patterns into the
deterministic tiers. Same philosophy as the review queue: operational exhaust
becomes training data for the cheap path. Track the tier-mix over time — a
healthy system shifts resolution volume *down* the tiers.

### 17.5 Attention-tiered enrichment

Spend derivation budget by evidence of attention, not uniformly:

| Signal | Investment |
|--------|------------|
| Provisional entity, 1 mention | stub: name + kind only |
| Recurring mentions / growing edge degree | summary + embedding |
| High degree / frequent retrieval hits | neighborhood-aware embedding refresh, dossier pre-rendering, closure-table candidacy |

Mention counts and edge degree are already queryable; retrieval-hit counts come
from search logging. This serves NFR-9.1 directly: cost follows change *and
attention*, not document-set size.

### 17.6 The consolidation cycle

One named, scheduled maintenance pass (nightly/weekly), with a ledger row per
run (mirror of `ingestion_run`):

1. re-run resolution over parked unresolved mentions (§15.5);
2. duplicate-candidate sweep (name/alias/embedding proximity among active
   entities) → merge proposals into the curation queue;
3. summary/dossier/embedding refresh for entities whose neighborhood changed
   (stale flags) — prioritized by attention tier;
4. integrity audit (graph-global invariants from §7's "accepted looseness"
   bucket: containment acyclicity, hub-shape conformance drift);
5. UNCLASSIFIED pattern report → candidate vocabulary additions (§14.4);
6. resolver tier-mix and extraction-precision metrics snapshot (§17.4, §14.6).

Nothing in the cycle writes to the trusted graph directly — it proposes into
the same staging/curation gates as everything else. Self-maintenance under
governance, not self-modification.

### 17.7 Self-improvement — three tiers, one gate

Review outcomes are a labeled dataset produced as a side effect of operation
(§14.6). What that dataset is allowed to change automatically is tiered:

**Tier 1 — data-level learning: automate freely.** Alias accretion (§14.5),
per-(rel_type × tier) threshold tuning from accept/reject rates, degeneracy
thresholds from parser overrides (§14.1), resolver normalization rules mined
from tier-3 fallbacks (§17.4). Narrow, attributable, reversible, and already
passing through existing gates.

**Tier 2 — the governed prompt channel.** Extraction prompts are *generated
from the catalog* (§14.4), so prompts improve by refining the two-question
`rel_type`/`node_kind` descriptions — which is exactly what "map to existing
type" outcomes report (the extractor fell through a gap in a description).
**Catalog governance is therefore prompt governance**: amendments flow through
NFR-7.4's vocabulary-change discipline, no new gate needed. A curated
**example store** (approved extractions as few-shot exemplars per rel_type,
rejections as negative examples) belongs in this tier too — data, not
instructions; attributable, per-type, revertible.

**Tier 3 — auto-rewriting prompt instructions: propose, never promote.** Three
reasons: *blast radius* (an alias affects one entity; a prompt edit affects
everything the extractor does, unattributably); *reproducibility* (NFR-4.2
wants to know which extractor produced an assertion — a self-mutating prompt
breaks that); and the deep one, *reflexivity* — the loop's signal is "a
curator approved this," not "this is true," and a system optimizing on
approval converges toward *approvable* output. If the thing that changes the
prompt is also the thing that measures it, nothing anchors the drift; human
promotion keeps the definition an independent standard rather than a moving
average of recent behavior.

**Mechanics for any amendment (Tier 2 or 3):**

1. **Evidence with the diff** — the reviewer sees the motivating cluster
   (e.g. 40 rows curators remapped), the before/after text, and **measured
   precision deltas** against the golden set and retained review set. Not
   "here's better wording" but an evidence-backed decision.
2. **Versioned descriptions, stamped on extractions** — `rel_type` carries a
   version counter (alongside `origin`/`motivated_by`, §14.4); each assertion
   records the catalog version its extraction ran against. This makes
   "precision improved after the amendment" measurable rather than a vibe.
3. **Re-extraction is part of promotion** — a materially changed description
   means affected sections were extracted under different instructions;
   promoted change → re-extract → normal gate. Otherwise the document set becomes a
   stratigraphy of extraction eras that nothing reconciles.
4. **Precondition: the evaluation harness** (§19.7). Without ground truth
   outside the loop, "self-improving" and "self-drifting" are
   indistinguishable.

Compositor's ratification decisions feed *drafting* prompts exactly as
curation feeds *extraction* prompts — two loops, one pattern.

---

## 18. Component architecture — custody boundaries & the ingestion contract

Four deployable components plus the watcher, arranged so that exactly one thing
writes to the database and everything else is a replaceable client.

```
                          ┌─────────────────────────────┐
  GCS / Pub/Sub triggers → │  Extraction (+ Watcher)     │  Python-adjacent OK
                          │  fetch → parse → extract     │
                          └────────────┬────────────────┘
                                       │ Ingestion API (REST, batch)
                                       ▼
                    ┌───────────────────────────────────────┐
                    │  Portolan Service (Python)               │
                    │  hosts Portolan Core library             │
                    │  REST + MCP · sole DB custody          │
                    └───────┬───────────────────┬───────────┘
                            │                   │
                     PostgreSQL           REST / MCP consumers:
                 (Cloud SQL→AlloyDB)   ┌────────────────────────────┐
                                       │ Governance Console (UI)     │ curation +
                                       │ Viewer (thin graph UI)      │ ratification +
                                       │ AI agents (via MCP)         │ vocabulary
                                       └────────────────────────────┘
```

### 18.1 The components

The system is **Python end to end** (§19.5) — one ecosystem, one toolchain,
and a personal profile that installs with `pip`. Seven components:

**1. Portolan Core (library).** Owns the schema, migrations (Alembic or plain
SQL — no other component ever runs DDL), the `kb_*` operation surface,
staging, promotion, resolution, reconciliation, and query. PostgreSQL is its
only infrastructure dependency. Being the sole SQL speaker is what makes the
§10 standing constraint (all writes through the operation surface) structural
rather than aspirational.

**2. Portolan Service (deployment).** The one process hosting Core; exposes
REST (humans, extraction, UIs) and MCP (agents) — same service, two protocol
faces, kept distinct. In the enterprise profile it holds the only connection
string.

**3. Portolan CLI — one CLI, two backends.** Default backend: thin HTTP client
of the Service (enterprise; custody preserved, tenant from credential).
`--local` backend: binds Core directly in-process (personal profile; no HTTP,
no service to run). **The enterprise profile disables the local backend by
configuration** — a CLI linking Core in that profile would put database
credentials on laptops and take tenant from a flag rather than a credential
(§19.3). Same command surface, same output contracts, both backends.

**4. Textus — Extraction + Watcher (one deployable).**
Pipeline: **watch → fetch & mirror → parse → extract → submit.** Source
monitoring (§15.4 triggering, per-source version-semantics config), Stage-1
parsing, VLM asset pass, per-section LLM extraction, client-side validation
against cached catalogs. Talks to the Portolan Service over the ingestion API —
never to the database. Explicitly the watcher's charter too, so orchestration
never leaks into Core. Internal stages are **modules, not services**; the only
internal boundary that must stay clean is deterministic stages vs. the LLM
module (isolates flakiness/cost/retries; enables standalone regression runs
against the golden set per NFR-7.5; the one plausible future scaling split).
MVP runs sequentially in-process — no internal queue; retryability comes from
the hash cascade + idempotent batch upserts (crash, restart, re-run: completed
work is skipped).

*Note: mention resolution is NOT in this pipeline — it lives in Portolan
ingestion (see below). Textus is graph-ignorant: it emits mentions, never node
IDs. This keeps any extractor valid under the FR-10.7 contract and puts
resolution where the indexes live.*

Technology: **plain Python (as is the whole system, §19.5), no orchestration framework.** Stage-1 format
router per §14.1: **Docling primary** (PDF + Office + HTML natively),
markitdown secondary for the long tail, `StructureRecoverer` port (Document AI
primary) as the review-routed fallback; all lanes emit the same contract
(normalized **markdown as the canonical content form** + section tree + spans
+ assets). Markdown-canonical
makes §15.1's normalized hashing concrete: headings → paths deterministically;
heading-free documents fall back to synthetic sections by paragraph clusters,
flagged low-confidence structure (one rule in the segmenter, not per-incident).
**PageIndex** (open-source) sits on the bench as the fallback structure parser
for degenerate PDFs (Docling yields one giant section) — caveat: its
LLM-assisted tree building makes structure probabilistic, so fallback outputs
route to review rather than the trusted path. LLM calls: Pydantic structured
outputs against the catalog-generated prompt + a provider seam (direct Vertex
SDK or LiteLLM per NFR-8.4); Instructor optional for validation-retry.
LangChain/LangGraph deliberately not used — the pipeline is linear and the
prompt/schema are versioned contract artifacts that must stay visible (NFR-7.5),
not templated through framework indirection. (Autonomous agent platforms — e.g.
Shakudo Kaji — are a category mismatch for this pipeline; they are candidates
for the §16 Compositor, which IS mission-shaped work.)

**Portolan-side ingestion flow (receiving a batch).** receive → catalog-validate
→ **resolve mentions (tiers 1–4, tenant-scoped, §14.5)** → gate (confidence ×
tier thresholds + the lifecycle hard floor) → curate → promote. Resolution
lives in core because: (1) *data gravity* — it is index lookups (identifiers,
aliases, trigram, entity embeddings) over tables in Postgres, milliseconds
locally vs. chatty API calls or stale exported caches; (2) *one resolver, many
callers* — §15.5 re-resolution, §17.6 consolidation sweeps, and §14.5 alias
accretion all share it; (3) *extractor replaceability* — every submitter gets
identical resolution; (4) *custody* — resolution feeds gate decisions, which
are promotion-adjacent. Core's LLM surface is exactly **two injected ports**
(§19.5): `EmbeddingProvider` (embed a string — used by search and tier 3) and
`ResolutionRanker` (tier 4: mention + top-k candidate summaries → ranked
verdict, always review-routed). A third LLM-shaped need in core is a
design-review moment, not a port addition — the smallness of this surface is
an invariant: everything probabilistic core consumes arrives through a gate or
through a review-routed port.

**5. Compositor (deployment; phase 2).** Conversational-source formalization
(§16): watches email, Slack, Teams and meeting capture, detects decision
signals, drafts decision-shaped memos, routes for ratification, submits
ratified documents through the same ingestion API as Textus. A sibling of
Textus, never a peer of Core.

**6. Governance Console + Web UI.** Curation queue, provisional-entity
confirmation, merges, UNCLASSIFIED pattern view with the four outcomes
(§14.4), ratification workflow (§16.4), vocabulary administration; plus the
browse/search/dossier Viewer (§18.6, and the vault export of §17.1 as the
zero-cost interim). Speaks REST/MCP only. Two permission domains from day
one: curate vs ratify (NFR-6.2). May host Core in-process (trusted, same
custody boundary) or speak REST.

**Viewer (UI, thin) — decided: embedded graph canvas, custom shell.** Browse/
search/traverse over `kb_search` / `kb_dossier` / `doc_level_relations`. The
market scan is conclusive: existing graph UIs (KG-UI, LinkedDataHub, Open
Semantic Explorer, Gephi) target Neo4j/RDF/SPARQL backends, none speak
Postgres, and **none render supersession badges, as-of pinning, rollup
drill-down, dossiers, or gap panels — which are Portolan's entire visible
value.** Adopting one means writing an adapter that fakes their backend; more
work than the shell it saves. Decision:

- **Library: Cytoscape.js** (mature, MIT, selector-based styling that maps
  directly onto rendering rules — `node[status="retired"]`,
  `edge[rel_type="SUPERSEDES"][valid=false]`). G6 if richer built-in
  interactions are wanted; react-force-graph for a same-day throwaway; Sigma
  only if thousands of nodes must render at once (they should not — see next
  point).
- **Bounded neighborhood, never the whole graph.** The Viewer's unit is a
  *focus node + 1–2 hops*, node-capped, filtered by rel_type and `as_of` —
  exactly what `kb_search`/`kb_dossier` already return. A global graph view is
  a trap: unreadable past a few hundred nodes and it tempts unbounded queries.
  Overview belongs to dossier lists and document indexes.
- **Graph-fragment DTO.** The service returns `nodes[] / edges[]` with
  *display attributes already computed* (status, validity, confidence,
  citation refs) — presentation state is never derived client-side from raw
  facts. Keeps as-of logic server-side where it is tested, and keeps the MCP
  surface and Viewer consistent.
- **Styling owns the semantics.** Retired nodes dimmed, provisional entities
  dashed, superseded edges shown-but-struck (never hidden — FR-2.6 made
  visual), validity ranges on edge labels, a date pinner that re-queries on
  change, dossier side panel with citations. The library is ~10% of the work;
  this styling and the panels are the rest.
- **Expansion re-focuses**, it does not accrete: clicking a node calls the
  same bounded-neighborhood endpoint with a new focus, rather than merging
  fetches into one canvas — otherwise state management and layout stability
  become the project.
- **Build order: dossier panel first, canvas second.** Readers spend most of
  their time on one entity's compiled state; the panel answers questions, the
  graph answers "what's nearby."

**Naming warning:** do not call it "Atlas" — Apache Atlas is an unrelated
Hadoop-era metadata catalog and the collision pollutes every future search.
**Interim, zero-cost viewers** (phase-1, per §19.7 discipline): an MCP client
(Claude/Cursor) pointed at the Portolan Service for conversational, cited,
as-of-correct querying; the Obsidian vault export (§17.1) for visual browsing;
and — for the Governance Console's CRUD-shaped screens specifically — a
Python-native Streamlit console in the same repo is a legitimate first cut.
Whatever the tool, **the operation surface stays the only integration point**:
low-code builders (Appsmith/Budibase/ToolJet/Retool) may call REST endpoints,
never Postgres directly — a connection string in a low-code tool is a second
writer and voids custody (§18.4).

### 18.2 The ingestion contract (transport-independent batch document)

The unit of ingestion is a **self-contained batch document**, identical
regardless of transport:

- header: `batch_id`, extractor id+version, source document ref, catalog
  version used;
- structure submission (Stage 1, deterministic → trusted path): sections with
  paths/headings/ordinals, normalized content + hashes, assets, spans;
- candidate submission (probabilistic → staging): entities and relations as
  mentions — verbatim, span, confidence, suggested_type/suggestion where
  UNCLASSIFIED.

**The content-addressed section-object protocol (how content travels):**

1. Textus parses + normalizes, computes per-section content hashes, and writes
   each section's normalized markdown to the **artifact store as a
   content-addressed object** (key = section hash). Unchanged sections hash to
   keys that already exist — a HEAD check skips the upload, so **bandwidth
   scales with change automatically**; dedup falls out of the addressing
   scheme, no delta logic anywhere.
2. The **structure batch is metadata only**: document identity, the full tree
   (paths, headings, ordinals), per-section `(content_hash, artifact_uri)`,
   plus summaries/language/assets. Kilobytes regardless of document size.
3. Portolan **diffs hash lists against its stored hash lists** (tier 1 resolved
   without touching content), pulls content from the artifact store **only for
   new hashes** — which it needs anyway (that content enters `node_content`;
   §19.2 unchanged — this is transport, never storage), and which tiers 2–3
   need for fuzzy move-and-edit matching.
4. On pull, Portolan **re-hashes fetched bytes and requires equality with the
   claimed hash** — exact, free integrity verification; Portolan never trusts,
   it checks. Content-addressed objects are immutable, so no staleness or
   concurrency machinery exists to need.
5. Portolan reconciles atomically; the **submission response is the
   reconciliation report** (§15.2), scoping candidate extraction to changed
   sections. New documents are the degenerate case: report = "all new," and
   structure + candidates may be combined in one round trip.

**Section-object retention is configurable (per source / per profile):**
`retain` — keep objects after successful ingestion as the normalized-form
provenance mirror (cheap for text; pairs with the as-fetched original mirror,
§15.7); or `clean` — delete after Portolan confirms ingestion (hash verified,
`node_content` written, `ingestion_run` recorded), to save storage cost. Under
`clean`, `node_content` is the sole normalized copy and re-derivation
(re-chunk/re-embed/re-extract) is unaffected (§19.2); only the audit
convenience of an immutable normalized snapshot is forgone — the as-fetched
original in the mirror remains the provenance anchor either way. Never delete
before the ingestion outcome is durably recorded; cleanup is
confirmation-driven, not time-driven.

**Transport independence:** this batch contract is the *module API* of
ingestion. HTTP is one binding (cloud profile); **in-process invocation of the
same operations is another** (personal profile, §20) — idempotency, protocol
steps, and validation semantics identical in both; locally the artifact store
degenerates to a shared directory.

Semantics: idempotent by `(batch_id, ext_id)` upsert — retry until 2xx (or the
in-process equivalent: retry until accepted) is the entire client error model. Inline catalog validation on submit returns
violations in the response; client-side pre-validation against cached catalogs
(ETag/versioned catalog endpoint — bump on vocabulary change, clients
revalidate) catches most malformation at the producer. UNCLASSIFIED is the
*valid encoding* of "no type fits," so unknown-type flow and validation never
conflict.

### 18.3 Transport evolution

1. **Now (MVP and steady state): REST batch API.** Load is change-driven
   (NFR-9.1) — even busy days are trivial REST traffic; synchronous validation
   feedback and natural backpressure (429/503 + Retry-After) are worth more
   than async decoupling. Pub/Sub is used only for *triggering* (GCS
   notifications → watcher), never data transport.
2. **When bulk hurts (initial backfill; re-extraction campaigns after prompt or
   model upgrades pass NFR-7.5 regression): claim-check.** The same batch
   document is written to GCS; a small Pub/Sub message carries the object
   pointer; the service consumes pointers and streams payloads (COPY-friendly).
   GCS gives unbounded payload + replay-from-bucket recovery; Pub/Sub gives
   delivery/retry; the 10MB message ceiling never applies. Because the batch
   document is transport-independent, this upgrade adds an ingestion path and
   changes nothing in extraction output, staging, or promotion.

### 18.4 Custody rules (the invariants this architecture exists to protect)

1. One writer: only the Portolan Service (hosting Core) touches PostgreSQL.
2. All writes through the `kb_*` function surface (§10 bitemporal migration
   insurance).
3. Extraction is a client, never a peer: catalog reads and batch submissions
   over the API, including deterministic Stage-1 structure (via a trusted
   `submit_structure` endpoint) — no carve-out direct writes.
4. UIs and agents consume the same REST/MCP surface (FR-10.3); the viewer gets
   no privileged access the MCP client lacks.
5. The **artifact store is the one sanctioned shared surface** between Textus
   and Portolan: Textus writes (originals + content-addressed section objects),
   Portolan reads — trustless by construction, since content addressing +
   re-hash-on-pull make every read exactly verifiable (§18.2). It stores
   artifacts, never truth: the graph and `node_content` remain Portolan-only.

### 18.5 CLI adapters — the third face of the operation surface

Two thin CLIs make the system drivable from agentic tools (Claude Code, Codex)
and human shells; both are **clients, never peers**:

- **`portolan` CLI — default (HTTP) backend.** Wraps the Portolan Service's REST
  API — the same endpoints Textus and UIs use. In the enterprise profile this
  is the *only* permitted backend: a library-linking CLI there would be a
  second writer with connection strings on arbitrary laptops (§18.4 rule 1
  dies), would need a `--tenant` flag (the passed-identity flaw, §19.3), and
  would bypass the egress policy. The `--local` backend (binding Core
  in-process) exists for the personal profile, where the OS user is the trust
  boundary and there is no service to talk to; it is disabled by configuration
  in the enterprise profile.
- **Peripherals (component 7).** Agent skills, slash-commands, scripts, and
  scheduled jobs consume Portolan *through the CLI* rather than through bespoke
  integrations — the CLI's `--json` contract is their API. This keeps the
  peripheral tier free of credentials, free of core coupling, and identical
  across profiles.
- **`textus` CLI (same codebase as Textus).** Runs the pipeline as a one-shot:
  `textus ingest ./doc.pdf` = parse → extract → submit without the watcher
  loop. Designed-in decomposition: **the watcher is "the thing that calls the
  same entrypoint the CLI exposes"** — one codepath, two triggers, provably
  identical. Free consequences: manual ingestion (the primary MVP interface —
  skip the watcher entirely at first), the re-extraction campaign tool, and
  `textus extract-only --dry-run` printing batch JSON without submitting —
  which is the NFR-7.5 regression rig materializing.

Agent-ergonomics conventions (agents are a first-class audience): `--json`
output on every command; exit codes mapped to API error classes (0 =
accepted; distinct codes for validation-failed / parked-unresolved / auth);
stdout for data, stderr for progress; idempotency inherited from the batch
contract so retries are always safe (§18.2). Auth: API key in env
(`PORTOLAN_API_KEY`) — which is also where tenant binding attaches (§19.3: the
key IS the tenant); the CLI never grows a `--tenant` flag.

### 18.6 Reasoning-based traversal (PageIndex-style) as an agent pattern

Portolan already stores the data structure PageIndex builds for long documents:
the section tree (`parent_section_id` + paths + headings) with per-section canonical
summaries **is a hierarchical ToC index**. Reasoning-based retrieval —
navigate the tree, read summaries, drill into the relevant section — is
therefore an **agent behavior over the existing MCP surface, zero new
infrastructure**, complementing the four-leg fusion for long, structured,
professional documents (the class where tree navigation empirically beats
similarity search). It composes cleanly with the egress policy: tree +
summaries are exactly what §19.2 allows out; the agent follows `document.uri`
for full text. Future convenience operation: `kb_toc(document_id)` returning
the tree + summaries in one call.

### 18.7 Repository & build structure — one monorepo, enforced boundaries

**Decision: a single repository with independently buildable packages** (uv
workspaces; Poetry/PDM equivalents work identically). One lockfile, one
`uv sync`, per-package builds and tests.

```
portolan/
├── pyproject.toml              # workspace root: members, shared dev deps, tool config
├── uv.lock                     # ONE lockfile — a principal reason for the monorepo
├── packages/
│   ├── portolan-contract/      # Pydantic models: batches, mentions, search DTOs
│   ├── portolan-core/          # library: schema, migrations, operations, ports
│   ├── portolan-router/        # REST routers + MCP tool registry; depends on core
│   ├── portolan-extractor/     # parse, normalize, extract claims; emits mentions
│   ├── portolan-watch/         # change detection; calls the extractor in-process
│   ├── portolan-cli/           # NOT BUILT — dual backend (§18.5): core (local) | httpx (remote)
│   └── compositor/             # NOT BUILT — phase 2
├── ui/                         # NOT BUILT — separate toolchain (node/vite), own CI job
└── docs/                       # REQUIREMENTS.md, ARCHITECTURE.md, decisions/
```

Entries marked NOT BUILT are specified here but absent from the repository;
per GUIDELINES §5.3 they are proposals until they exist in code.

`portolan-extractor` and `portolan-watch` are two packages but remain **one
deployable** (§18.1): the watcher imports the extractor and drives it
in-process, with no queue between them. They are split so the extractor stays
installable without connector dependencies, which the one-shot CLI path and
re-extraction campaigns both need.

**Import namespace.** Distributions are named `portolan-<name>`; the import
path is `portolan.<name>` (`portolan.core`, `portolan.contract`). `portolan/`
is a PEP 420 implicit namespace package, so **no package may contain
`src/portolan/__init__.py`** — adding one shadows the namespace and breaks
every sibling import.

`uv build --package portolan-core` publishes the library alone;
`uv run --package portolan-extractor pytest` tests extraction alone; CI runs
per-package jobs, path-filtered so touching the extractor does not re-run
core's testcontainers suite.

**The dependency rule that matters more than the layout: the submitting
packages — `portolan-extractor`, `portolan-watch`, and Compositor — MUST NOT
depend on `portolan-core`.** They are API clients speaking the batch
contract (§18.2). Importing core would silently make them second writers with
database access, voiding custody (§18.4) the first time someone calls a core
function "just for a lookup." What they share instead is
**`portolan-contract`** — the Pydantic models for batches, mentions, and search
DTOs — giving type-checked contract compliance across the boundary without
leaking implementation.

Allowed dependency graph: `router → core, contract`; `cli → core (local
backend only), contract, httpx`; `extractor → contract`; `watch → extractor`;
`compositor → contract`; `core → contract`.

**Enforcement is tooling, not discipline.** Python will not stop a stray
import, so the graph above is declared in **import-linter** (or an equivalent
CI check) and violations fail the build — ten lines of config protecting the
architecture's most important rule, and precisely the kind of compensation the
Python decision (§19.5) committed to.

**Why monorepo here:** contract changes land atomically with their consumers
in one PR (versus version-bump choreography across five repos); one lockfile
prevents core/router/CLI dependency drift; the personal profile's
single-install story builds trivially from the workspace; and at solo/small-
team scale with contracts still moving, separate repos buy coordination cost
without benefit. Splitting extraction out later — if it needs a different team
and cadence — stays easy *because* the dependency rule kept it a pure API
client.

---

## 19. Multi-tenancy & content custody

### 19.1 Multi-tenancy (core requirement)

Core hosts disjoint sets of documents, vocabulary, and knowledge for multiple
tenants. MVP isolation model: shared tables, mandatory tenant scoping.

```sql
CREATE TABLE tenant (
    id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    slug  text UNIQUE NOT NULL,
    props jsonb NOT NULL DEFAULT '{}'
);
-- tenant_id (NOT NULL, FK) lands on every root and, denormalized, on every
-- hot table: document, node, edge, staging_*, node_content, chunk, asset,
-- assertion, ingestion_run — and on the catalogs (see below).
```

Rules that make it real:

1. **Every `kb_*` function takes `tenant_id` as its first parameter**, applied
   as a mandatory predicate — the `as_of` pattern again: callers cannot forget
   it because no function signature permits a cross-tenant query.
2. **Indexes are tenant-prefixed**: `edge (tenant_id, src, rel_type) INCLUDE
   (dst, validity)` and siblings — every scan is confined to one tenant's index
   slice, so tenants do not degrade each other.
3. **Edges cannot cross tenants**: the §7 validation trigger checks that src,
   dst, and graph_id all belong to `NEW.tenant_id` — cross-tenant links are
   structurally impossible, not merely unqueried.
4. **Catalogs are per-tenant with a seeded kernel.** Different tenants have
   different domains and vocabularies; a shared catalog would make one
   tenant's UNCLASSIFIED ratification change every tenant's extraction
   prompt. `tenant_id` on `node_kind` / `rel_type` / `hub_role`; uniqueness
   becomes `(tenant_id, name)`; tenant creation seeds the kernel vocabulary
   (document, section, entity, SUPERSEDES, SUPPLEMENTS, DEFINES, MENTIONS); each
   tenant's vocabulary evolves independently through its own four-outcome
   loop (§14.4).

Escalation path (pre-decided): shared-tables-with-tenant_id is the MVP; if a
tenant demands hard isolation (compliance, noisy-neighbor, residency),
schema-per-tenant or database-per-tenant is a data migration behind the same
function surface — `tenant_id` keeps its meaning and changes only what it
routes to. Postgres RLS keyed on tenant_id is available later as
belt-and-suspenders; redundant while the service is the sole writer.

### 19.2 Content custody & the egress policy

**Raw content stays; it just never leaves.** Stored text is required for
capabilities that vectors and hashes cannot provide (embeddings are one-way —
they find text, they cannot produce it): snippets/highlighting, curation's
verbatim-in-context, re-extraction under an evolved vocabulary (§14.4),
re-chunking and re-embedding on model migration, and §15 similarity
reconciliation. Hash-only storage would freeze the system into a one-shot
index. (Scale check: text ≈ 15GB vs vectors ≈ 60GB at target scale — text is
not the storage problem.)

**The internal accessor.** All content reads inside core go through one seam:

```sql
-- INTERNAL: not exposed on any public surface
CREATE FUNCTION content_of(t bigint, node bigint, slice int4range DEFAULT NULL)
RETURNS text ...;   -- whole section content, or a span slice (chunk text)
```

This is also the externalization escape hatch: if content ever moves to
content-addressed GCS objects (key = content_hash, spans remain valid against
their hash), only this function changes.

**Egress policy — what the public surface returns:**

| Leaves core's public surface | Never leaves it |
|---|---|
| entity/document/section metadata: names, kinds, paths, headings | `node_content.content` (full section text) |
| canonical summaries (derived, bounded — a policy owner should consciously accept that summaries paraphrase content) | chunk text |
| facts/edges with temporal validity | full `asset.ocr_text` |
| citations as **pointers**: document ref + section path + span offsets + source URI — not quoted text | staging verbatims / `assertion` verbatim text |
| bounded match **snippets** ("highlights"): ts_headline output capped (~200 chars per match, few per result) | |
| gap information; dossier compiled truth (built only from the above) | |

Consequences, accepted deliberately:

- **Portolan is a knowledge server, not a document server.** Agents synthesize
  from facts, summaries, and snippets; for full text they follow
  `document.uri` to the source system, whose own permissions apply. Portolan
  never becomes an oracle that bypasses source-system ACLs.
- **Curation is the explicit exception**: reviewing a claim requires the
  verbatim highlighted in section context (FR-5.4), so content-bearing
  responses exist only on the governance (curation/ratification) surface,
  role-gated at the service layer. The endpoint split exists from day one
  even while full authorization is deferred.
- No public `kb_*` function includes a content column in its return type —
  the policy is enforced by function signatures, not by response filtering.

### 19.3 Authorization deferral (MVP posture) — with one non-deferrable rule

Core is authorization-ignorant beyond tenancy: the service layer (or IAP
alone, in MVP) gates who reaches which tenant and which endpoint class
(public vs governance). Fine-grained document-level ACLs, verified-principal
agent identity, and label-scoped curation are service-layer concerns,
deferred — but the seams they will need already exist: the operation context
(§19.4), the governance/public endpoint split, and single-writer custody.

**Non-deferrable: tenant binding.** `tenant_id` is never caller-supplied
data. The service layer derives it from the presented credential (this API
key / token IS tenant N) during operation-context construction, before any
core call. A tenant parameter accepted from request payloads would be the
"passed identity" flaw in another costume; the binding rule must exist before
the second tenant does.

### 19.4 The operation context (signature design)

Core operations distinguish two kinds of input:

- **Context** — who/for-whom/as-of-when: tenant, as_of, with reserved slots
  for belief_at (bitemporal, §10) and principal/groups (future authz).
  Ambient, security-relevant, identical across a request.
- **Options** — per-call knobs: hops, k, explain, snippet caps.

Rules:

1. One immutable `OperationContext` is the first argument to every core
   operation; per-function options objects carry the knobs. Nothing per-call
   ever enters context.
2. Context is constructed **exactly once, at the service boundary**, from the
   verified credential — this is where tenant binding (§19.3) happens. It is
   never rebuilt or mutated mid-request.
3. New cross-cutting concerns (belief_at, principal) are added as optional
   context fields with safe defaults — field additions, not signature changes
   across library/REST/MCP.
4. Known hazard: context objects convert accidental *omission* (impossible)
   into accidental *reuse* (possible — caching across requests or tenants in
   a worker pool). Immutability + per-request construction from the credential
   is the discipline.

The few SQL-resident functions (§19.5) keep explicit scalar parameters — a
context type in SQL is ceremony at that scale.

### 19.5 Where the function surface lives — and what it is built from

**The core library is the function surface.** "The `kb_*` functions"
throughout this document name *operations of the core library*, not database
objects. Logic embedded in the database is minimized: SQL/plpgsql functions
exist only where residing in the database is intrinsic to correctness — the
close-and-assert atomicity of supersession, the promotion transaction, the
internal `content_of` accessor. Everything else — search fusion, traversal
orchestration, dossier assembly, resolution, validation against catalogs —
lives in library code, where it is testable, observable, and evolvable
without migrations.

**Language decision: Python (revised).** Rationale — one ecosystem across the
whole system beats a type-system advantage in one component:

- **Textus must be Python** (Docling, markitdown, VLM tooling have no JVM
  equivalents), Compositor wants Python, the CLI wants Python for agent
  ergonomics. A JVM core would force the *personal profile* (§20) to require a
  JVM **and** a Python runtime **and** Postgres on one machine, with a JSON
  boundary between components that §20 says should collapse to in-process
  calls. Python-everywhere makes the local profile a `pip install` and makes
  "Textus calls core in-process" literally true.
- The workload is IO-bound; raw performance is irrelevant.
- Costs are real but bounded — see the mitigation stack below.

**What is given up, and how it is bought back.** The invariants split in two:

*DB-enforced, therefore unaffected by the language choice:* temporal
non-overlap (exclusion constraints), tenant isolation (FK + trigger), catalog
conformance, hub-shape validation at promotion, referential integrity. These
were always in the DDL — which is exactly why this decision is affordable, and
it argues for pushing *more* invariants down into constraints/triggers rather
than trusting application code.

*Application-enforced, needing compensation:*

| Invariant | Kotlin mechanism | Python replacement |
|---|---|---|
| Egress by signature (§19.2) | return types without content fields | **Pydantic return models with no content field** — content cannot leak from a model that cannot represent it (structurally equivalent) |
| OperationContext threading (§19.4) | compiler | frozen Pydantic model as first parameter + **CI lint/AST check** that no core operation lacks it |
| MECE exhaustiveness | sealed classes | `enum.Enum` + `typing.assert_never` in match statements, checked by mypy |
| Typed SQL | jOOQ generated from schema | **the real loss** — no equivalent exists. Hand-written SQL (which this schema wants anyway: window functions, ranges, `&&`, RRF) with **integration tests against real Postgres (testcontainers)** as the verification layer instead of the compiler |

Non-negotiable companions: `mypy --strict` in CI, Pydantic at every boundary,
and the testcontainers suite as a *load-bearing* requirement rather than a
nicety — it is now the thing standing where the compiler stood.

**Framework independence (dependency contract).** The core is a domain
library over Postgres; it depends only on what helps it be that:

- *Allowed:* Python stdlib; **psycopg** (+ SQLAlchemy Core *only* if used as a
  query builder, never the ORM) as the sole database layer, with explicit
  transaction blocks (transaction shape is load-bearing, §10); **Pydantic**
  for domain and boundary models; the `logging` stdlib module (never a
  configured handler — shells configure logging); migration files owned by
  core (Alembic or plain SQL — decide once, open checkbox; nothing else runs
  DDL).
- *Forbidden:* FastAPI/Flask/Django or any web framework, MCP libraries, DI
  containers, Celery/task frameworks, config frameworks, metrics
  implementations, **any ORM**. These answer service-shell questions the core
  must not know exist. No framework type ever crosses the core API.
- *Ports:* everything worldly is an injected protocol (`typing.Protocol`) —
  connection/pool, `EmbeddingProvider`, `ResolutionRanker`, `Clock`. Shells
  wire implementations (Vertex client now; in-database `google_ml`
  post-AlloyDB; frozen clock in tests).

**Shells may be frameworks.** The Portolan Service hosting core may be FastAPI
(the natural Python choice) or anything else — HTTP, metrics, config, and the
auth edge where OperationContext is constructed all live there. The boundary
rule, not the framework, is the invariant: no framework decorators on core
operations, no ambient request state below the shell, context constructed in
the shell and passed in. Consequences worth the discipline: the core API
becomes the literal FR-10.3 operation surface (REST, MCP, and CLI adapters are
provably equivalent thin mappers over one interface), and the entire
integration test rig is core + testcontainers-Postgres — every invariant
exercisable as a plain function call against a real database.

### 19.6 jsonb graduation rule (schema hygiene)

`props` is for *incidental* attributes. A jsonb field **graduates to a real
column** when any of: (a) it is read in a query path, (b) it is required for
correctness, (c) it is referenced by more than one component. Already overdue
by this test: `document_id` stamped on section nodes and `path`/`ordinal` — both
**resolved by decision 0002**, which promoted them to real columns on `node`
alongside `parent_section_id` and removed the CONTAINS edge entirely.
Borderline: `aliases`
(text[] column or alias table). Legitimately jsonb: `suggestion`,
`motivated_by`, `merged_into`, incidental domain attributes. The judgment has
an owner: every low-level design review re-runs this test over current jsonb
usage — load-bearing fields hiding in jsonb is the EAV failure mode
re-entering through our own back door.

### 19.7 Parked for later phases (recorded so they are decisions, not gaps)

- **Curation-capacity sizing (pre-build gate):** before building most of the
  pipeline, prototype extraction on a real document sample, multiply candidate
  volume to full-collection scale, and check it against actual reviewer hours. If it
  does not fit, the fix is launch scope (fewer rel_types; SUPERSEDES-first),
  decided before the vocabulary sprawls. Per-tenant vocabularies multiply
  this linearly — each tenant is its own curation economy.
- **Postgres as the personal profile's honest cost:** the schema leans on
  Postgres-specific machinery (exclusion constraints, ranges, tsvector,
  pgvector) that SQLite cannot replicate — the local app bundles/manages a
  Postgres (embedded distribution or container). This is the price of
  invariants-in-the-database; owned, not hidden.
- **Evaluation harness:** a golden question set (including temporal traps)
  must exist before `kb_search` tuning begins; parked at high-level design,
  revisit at low-level design. NFR-7.5 regression testing assumes it.
- **MVP slicing:** this document describes the complete system; the build
  proves a thin vertical spine (ingest → extract → curate → query-with-as-of)
  before going horizontal. Slice to be named at low-level design. An
  all-Python **throwaway spike** of the spine is a sanctioned measurement
  instrument (extraction quality, curation volume, search feel) — provided it
  is named disposable in writing; prototypes ossify, and the system build
  follows §19.5's language decision.

---

## 20. Deployment profiles & the end-to-end ingestion flow

### 20.1 Two profiles, one core

The invariants (cost-follows-change; every claim anchored; stratified trust;
one custodian; as-of correctness) hold in any deployment. Everything else is
*binding*. Two named profiles bind the ports differently:

| Concern | **Enterprise / cloud** | **Personal / local** |
|---|---|---|
| Topology | Portolan Service (REST+MCP) + remote Textus | one process: core library embedded, pipeline in-process, CLI/MCP shell |
| Ingestion transport | HTTP binding of the batch contract | in-process binding, same contract (§18.2) |
| Artifact store | GCS, content-addressed | local object dir (`~/.portolan/objects/<hash>`) |
| Triggers | connector polling / Pub/Sub | filesystem events (§15.7 local connector) |
| StructureRecoverer | Document AI primary | granite-docling on CPU, or park degenerate docs |
| Embedding provider | Vertex (C-8) | user-consented provider choice via the same seam |
| Auth / tenant binding | credential → OperationContext at service edge | single implicit tenant; OS user is the trust boundary |
| Curation thresholds | conservative; start 100% review | permissive defaults; review only lifecycle edges + provisional entities |
| Hard floor (§14.5) | applies | **applies** — falsified history is bad at n=1 too |
| Ratification (§16) | role with standing | collapses — the user has inherent authority |
| Section-object retention (§18.2) | `retain` default | `clean` default (cost-sensitive) |
| C-8 | binding constraint | scoped out; consent replaces boundary |
| Postgres | Cloud SQL → AlloyDB | bundled/embedded Postgres or container (§19.7 — owned cost) |

Profiles are configuration over the same core, contract, and tests — no
forked logic. A third profile (e.g., on-prem enterprise) is a new column, not
a new design.

### 20.2 The end-to-end flow — who sends what

| # | Step | Actor | Payload / action | Response / outcome |
|---|------|-------|------------------|--------------------|
| 1 | Detect | Textus connector | enumerate; change hints (eTag/version/updatedAt/mtime); delta cursor persisted | candidate documents |
| 2 | Fetch + mirror | Textus | download original once → artifact store, content-addressed (as-fetched bytes) | local file for parsing |
| 3 | Normalize + hash | Textus | canonical markdown; **normalized doc hash** | unchanged? → touch `last_checked`, stop (the free path) |
| 4 | Parse | Textus | router (Docling primary) + degeneracy checks (+ recoverer) | section tree + spans + assets (+ low-confidence flag) |
| 5 | Stage section objects | Textus | per-section normalized markdown → artifact store keyed by section hash; HEAD-skip existing | content staged, change-scoped |
| 6 | Structure submission | Textus → Portolan | **metadata batch**: doc identity, tree (paths/headings/ordinals), per-section (hash, uri), summaries, language, assets | **reconciliation report**: changed / added / moved(edges preserved) / removed(retired, queued) |
| 7 | Reconcile + ingest content | Portolan | diff hash lists (pure matcher module); pull **new-hash content only**; re-hash-verify; write `node_content`; re-derive (tsv, chunks, embeddings); apply tree mutations atomically | trusted structure updated; §15.3 scoped invalidation (stale assertions queued) |
| 8 | Extract | Textus (LLM) | per **report-listed** section: one structured-output call, catalog-generated prompt | entities + relations as mentions (verbatim, span, confidence, UNCLASSIFIED+suggestion) |
| 9 | Candidate submission | Textus → Portolan | staged batch of mentions; idempotent by (batch_id, ext_id) | accepted into staging; validation violations returned |
| 10 | Resolve | Portolan | tiers 1–4 over its indexes (tenant-scoped); tier-4 via ResolutionRanker port, review-routed | resolutions with tier+score; unresolved parked |
| 11 | Gate | Portolan | confidence × tier thresholds (profile defaults) + lifecycle hard floor | auto-approve / review queue |
| 12 | Curate | human (Console/CLI) | approve / reject+reason / edit-target; alias accretion; four-outcome vocabulary loop | promotion inputs; labeled data |
| 13 | Promote | Portolan | atomic promotion transaction (kb_supersede for temporal; hub-shape validation) | facts in trusted graph, cited, as-of-correct |
| 14 | Cleanup (optional) | Textus/store policy | per retention config: `clean` deletes section objects **after** step 7 confirmed durable | storage cost bounded |

New documents: report = "all new"; steps 6+9 may combine into one round trip.
Re-extraction campaigns: steps 8–13 only, reading content from `node_content`.
Local profile: identical steps, in-process calls, shared-directory store.

---

## Appendix — complete DDL execution order

0. Tenancy: `tenant` table; `tenant_id` on all tables & catalogs; tenant-
   prefixed indexes (§19.1)
1. Extensions (§1, §13.2: vector, btree_gist, unaccent, pg_trgm)
2. Text search configurations: `english_ua`, `french_ua` (§13.2)
3. Catalogs: `node_kind`, `rel_type` (incl. `origin`/`motivated_by` §14.4),
   `hub_role`, `rel_disjoint` (§2); v1 vocabulary = document/section/entity
   kinds + SUPERSEDES/SUPPLEMENTS/DEFINES/MENTIONS (decision 0002)
4. Core: `node` (incl. `status`, §14.3), `edge` (§3)
5. Provenance: `document` (incl. `node_id` §14.1, `structure_hash`/`last_checked`
   §15.1), `assertion` (incl. `section_node_id` §15.3, `asset_id` §14.7,
   `stale` status §15.3) (§4)
6. Staging: `staging_node`, `staging_edge` (incl. `src_ref`/`dst_ref`/`resolution`
   §14.5; `suggested_type`/`suggestion`/`resolved_to` §14.4) (§5)
7. Search sidecars: `node_content`, `chunk` (§13.2), `asset` (§13.9)
8. Ingestion ledger: `ingestion_run` (§15.4)
9. Indexes (§6, §13.2, §15.3)
10. Constraints + `edge_validate` trigger (§7)
11. Views: `doc_level_relations` (§14.2)
12. Functions: `kb_supersede`, `kb_descendants`, `kb_graph`,
    `kb_lexical_candidates`, `kb_chunk_candidates`, `kb_search` (incl. explain
    mode §17.2), `kb_merge_nodes`, `kb_dossier`, internal `content_of`
    (§8, §13.4–13.5, §14.5, §17, §19.2) — all tenant-scoped (§19.1), none
    returning content on the public surface (§19.2)
13. Seed catalogs with your vocabulary (§2, §14.1, §14.3)
