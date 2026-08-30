# The Northwind QQL Corpus

The standard test corpus for QQL. Northwind is a small, public, relational
schema — orders, order details, products, categories, suppliers, employees,
shippers — which makes it a good base for a corpus that ships: it carries no
customer data and no dataset specifics of ours.

| File | What it is |
|---|---|
| `northwind-ontology.tqs` | The schema — classes, properties, declared domains and ranges |
| `northwind-rules.tqs` | The rule canon (below) |
| `northwind-data.tqd` | The instance data — 77 products, 830 orders, 2,155 order-detail lines |
| `northwind-context.tqd` | The contextual layer — price history bounded in time, and disagreeing stock readings told apart only by their tags |

## Building a store

```bash
qros load -d <store-dir> northwind-ontology.tqs
qros load -d <store-dir> northwind-rules.tqs
qros load -d <store-dir> northwind-data.tqd
qros load -d <store-dir> northwind-context.tqd
```

Load order is not significant for the facts; the rules attach to the rule
store and apply to whatever is present when the closure is computed.

## The rule canon

Three rules, chosen so the closure can **detect an ordering bug**. A set of
independent rules cannot: each of these reads what the one before it derived,
so a closure computed in the wrong order is a different closure, not merely a
slower one.

| Rule | What it exercises | Derives |
|---|---|---|
| `LineRevenue` | A computed measure — `unitPrice × quantity × (1 − discount)`, from base facts only | 2,155 |
| `StrandedLine` | Positive chaining — reads `:lineRevenue`, which no base fact carries | 228 |
| `CleanCatalogEntry` | Stratified negation — `FILTER NOT EXISTS` over a *derived* predicate | 69 |

Materialization reports **two strata**, not three: positive chaining converges
by fixpoint *within* a stratum, so `LineRevenue` and `StrandedLine` share one.
Only the negation forces a boundary.

`CleanCatalogEntry` is the load-bearing one. Negation over a derived predicate
is sound only if that predicate is fully materialized first, so it tests the
stratification claim rather than assuming it. Evaluate it before
`StrandedLine` finishes and eight discontinued products are wrongly declared
clean — a wrong answer that looks entirely well-formed and counts the same.

Revenue lives in a rule rather than in each query because it is business
semantics: the decision that revenue is *net of discount* is a modelling
choice, not a query detail.

### Why the negation routes through `:discontinued`

A vacuous negation proves nothing, and the obvious targets are vacuous here:
every product appears on an order detail, and every order carries lines, so
"never sold" and "order with no lines" both derive zero. `:discontinued`
splits 8 / 69, which gives the negation something real to exclude.

## The contextual layer

The base Northwind export is flat: every fact is unqualified, and its only tags
are import provenance. That makes tag *syntax* run and demonstrates nothing.
`northwind-context.tqd` adds context that CHANGES ANSWERS.

**A price history on `:listPrice`**, bounded with `@validFrom` / `@validUntil`,
so `AS OF` discriminates:

```
FIND ?p WHERE { :product-38 :listPrice ?p } AS OF "1996-06-01"   -> 210.00
FIND ?p WHERE { :product-38 :listPrice ?p } AS OF "1997-06-01"   -> 263.50
```

**Disagreeing stock readings on `:stockReading`** — the same subject and
predicate, different values, separable only by their tags:

```
FIND ?v WHERE { :product-38 :stockReading ?v . @source="warehouse-audit" }  -> 17
FIND ?v WHERE { :product-38 :stockReading ?v . @source="supplier-feed" }    -> 22
```

Both statements are true within their own context. This is Law 1 — a
statement's identity includes its context — so they coexist as distinct facts
rather than being resolved into one on ingest.

Both use predicates of their own, so the base counts and the rule yields above
are untouched. The history is invented for the corpus; Northwind has no real
one, and the file says so.

## Determinism

The closure must be the same closure every time — closed-world answers are
only worth something if they are predictable. Three assertions run as part of
the evaluation preflight, before any model spend:

- repeated materialization yields an identical closure;
- the chain derives exactly what each stratum should, with the discontinued
  products **not** declared clean;
- rule insertion order does not change the closure.

The fingerprint covers the **rule-derived slice** specifically, not the whole
store: materialization clears and rebuilds exactly that slice, so folding base
facts in would mask a drift in the derived half.

## Coverage intent

The corpus is being grown so that the dataset, ontology, rules and query bank
together use **every keyword in the QQL grammar at least once**, with enough
combinations to catch bugs that change results rather than merely fail to
parse. Coverage is measured against `QQL_SPEC.md` itself by
`tests/qql_spec_coverage.py`, not against a hand-kept list, so the corpus
cannot silently drift from the language.
