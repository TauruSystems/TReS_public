# Querying with QQL

QQL is the query language for TReS — your knowledge, queried.

QQL (the QROS Query Language) reads like a question with conditions. Its
distinguishing feature: the metadata on a fact — how confident it is, when it was
true, what context it holds in — is part of the query, not an afterthought. And a
query that depends on scope or time tells you **what it computed the answer over**.

This guide is task-oriented. For the metadata itself (tags, scope, rules), see the
[Ontology Guide](ONTOLOGY_GUIDE.md).

## The shape of a query

```
FIND ?product ?price
WHERE {
    ?product :type :Product
    ?product :price ?price
}
ORDER BY ?price DESC
LIMIT 10
```

- `FIND` lists what you want back (variables start with `?`; use `FIND *` for all
  variables that appear in the query).
- `WHERE { ... }` lists the patterns a result must match.
- `ORDER BY`, `LIMIT`, `OFFSET` shape the output.

A pattern is `subject predicate object`. Core predicates use the `:` prefix
(`:type`, `:label`, `:subClassOf`); your domain predicates use their own names
(`:price`, `:manufacturedBy`).

### What `FIND` projects

`FIND` is the only query keyword. It names what you want back:

- **Variables** — `FIND ?product ?price` returns a row per match.
- **All variables** — `FIND *` returns every variable that appears in the query.
- **Aggregates and expressions** — `FIND ?m COUNT(?p) AS ?count` computes over
  groups; see [Aggregates and grouping](#aggregates-and-grouping).

Add **`DISTINCT`** immediately after `FIND` to drop duplicate rows:

```
FIND DISTINCT ?manufacturer
WHERE { ?product :manufacturedBy ?manufacturer }
```

### Clause order

Clauses are read in a fixed order. Only `FIND` is required; the rest are
optional:

```
[EXPLAIN]
FIND [DISTINCT] ...
WHERE { ... }
AS OF <date | entity | TODAY() | NOW()>
GROUP BY ?vars
HAVING <expr>
ORDER BY <expr> [ASC | DESC]
LIMIT <n>
OFFSET <n>
```

You can annotate a query with `#` or `--` line comments.

## Patterns in `WHERE`

Inside the braces you combine several kinds of pattern.

**Triple patterns** — the core. Any of the three positions may be a variable:

```
?s :type :Product
?s :price ?price
```

Patterns are separated by `.` (or a newline). Turtle-style shorthand also works: a
`;` repeats the subject, and `,` repeats the subject and predicate — so
`?s :type :Product ; :price ?price` is the two triples above, and
`?s :tag :a , :b` is `?s :tag :a . ?s :tag :b`.

**Prefixed names and IRIs** — core vocabulary uses the bare `:` prefix; your own
terms use their prefix or a full IRI in angle brackets.

**Property paths** — reach across relationships without naming the intermediate.
Paths combine with `/` (sequence), `|` (alternative), and the quantifiers `*`,
`+`, `?`:

```
FIND ?ancestor
WHERE { :Terrier :subClassOf+ ?ancestor }      # one or more hops up the hierarchy

FIND ?product
WHERE { ?product :manufacturedBy :Acme }        # reverse direction: bind the object
```

There is no inverse operator. To ask a question in the reverse direction, bind
the object and let the subjects come back, as above; where the reversal falls in
the middle of a path, join on a shared variable:

```
FIND ?sibling
WHERE {
  :p1 :type ?t
  ?sibling :type ?t
}
```

**Paths stop at 10 hops.** A traversal that would need more returns an error,
not a shortened answer — a partial result you could not distinguish from a
complete one is worse than a refusal. Ten is enough for the great majority of
hierarchies; if you hit it, bind an end of the pattern (`:x :subClassOf+ ?y` or
`?x :subClassOf+ :y`) so the walk starts from a known point, or ask for a fixed
number of hops. The limit is fixed and not configurable.

Relationships you traverse in both directions routinely are worth declaring in
the ontology in both directions. Reaching for an operator at query time to walk
a relationship backwards is usually a sign the model has not said something it
should.

Paths apply to the **predicate** position only, and a path may not contain a
variable — write out a two-triple pattern if you need to bind an intermediate node.

**`:type+`** — matches the class or any descendant in the subclass hierarchy.
`?s :type :Product` matches only things typed `:Product` exactly; `?s :type+
:Product` also accepts anything typed to a subclass of it, however deep:

```
FIND ?product
WHERE { ?product :type+ :Product }       # typed :Product or any subclass of it
```

`:type*` is rejected — zero hops of `:type` would match every class to itself,
so the parser asks for `:type+` instead.

**`OPTIONAL`** — keep a row even when the inner pattern does not match; unmatched
variables come back unbound:

```
FIND ?product ?discount
WHERE {
    ?product :type :Product
    OPTIONAL { ?product :discount ?discount }
}
```

**`UNION`** — rows that match either alternative. The left alternative is the
pattern(s) that precede `UNION`; the right alternative is braced. Chain more with
additional `UNION { ... }`:

```
FIND ?item
WHERE {
    ?item :type :Product
    UNION { ?item :type :Service }
}
```

Each alternative is matched on its own, so a pattern shared by both arms must be
repeated inside each arm rather than factored out in front.

**`BIND`** — compute a value into a new variable:

```
FIND ?product ?net
WHERE {
    ?product :price ?price
    ?product :tax ?tax
    BIND (?price + ?tax AS ?net)
}
```

**Subqueries** — nest a `FIND` block in braces to compute an inner result the outer
query joins against.

## Filtering with `FILTER`

`FILTER` keeps only rows for which an expression is true. Expressions support:

- Comparison: `=` `!=` `<` `<=` `>` `>=`
- Boolean: `AND` `OR` `NOT` (also `!`)
- Arithmetic: `+` `-` `*` `/` `%`
- String concatenation: `||`
- Set and range tests: `IN (...)`, `NOT IN (...)`, `BETWEEN x AND y`,
  `IS NULL`, `IS NOT NULL`

```
FIND ?product ?price
WHERE {
    ?product :price ?price
    FILTER (?price >= 10 AND ?price < 100)
}
```

### Functions

These functions are available inside expressions:

| Category | Functions |
|----------|-----------|
| String   | `STR`, `STRLEN`, `SUBSTR`, `UCASE`, `LCASE`, `CONTAINS`, `STRSTARTS`, `STRENDS`, `CONCAT`, `REPLACE` |
| Numeric  | `ABS`, `ROUND`, `CEIL`, `FLOOR` |
| Logic    | `BOUND(?v)`, `COALESCE(...)`, `IF(cond, then, else)` |
| Matching | `REGEX(value, "pattern"[, "flags"])` |
| Temporal | `TODAY()`, `NOW()`, `DAYS_BETWEEN(from, to)`, and `YEAR/MONTH/DAY/HOURS/MINUTES/SECONDS` applied to a datetime |

```
FIND ?product
WHERE {
    ?product :label ?label
    FILTER CONTAINS(LCASE(?label), "wireless")
}
```

- **`REGEX` is a full regular-expression engine.** Anchors, character classes,
  alternation, and quantifiers all evaluate; flags are `i` (case-insensitive), `s`
  (dot matches newline), `m` (multi-line), `x` (ignore whitespace). For a plain
  substring test `CONTAINS`/`STRSTARTS`/`STRENDS` are simpler.
- **A date-typed value is a real datetime.** A value stored as `xsd:date`/
  `xsd:dateTime` binds as a datetime, so `YEAR(?d)` works and `?d < "2026-01-01"`
  compares temporally against the parsed date.
- **`DAYS_BETWEEN(from, to)` counts whole calendar days**, signed: positive when
  `to` is the later date, negative when it is earlier. Both operands are read at
  day granularity, so a clock time on a `dateTime` cannot shift the count, and a
  date held as a plain literal counts the same as a date-typed one. An operand
  that is not a date yields `null`, which fails the comparison rather than the
  query. This is the shape for "older than N days":
  `FILTER (DAYS_BETWEEN(?raised, TODAY()) >= 90)`.
- **`TODAY()` and `NOW()` read the clock when the query runs.** `TODAY()` is
  midnight UTC of the day of execution, `NOW()` the instant of execution. Both
  resolve once per query, so every occurrence in one query agrees — see
  [Asking about today](#asking-about-today).

## Filtering on a fact's tags

Every QROS fact can carry tags — the metadata that makes QROS QROS. You filter on
them inline, and the filter applies to the fact matched by the **immediately
preceding triple pattern** in the same block:

```
FIND ?product ?price
WHERE {
    ?product :type :Product
    ?product :price ?price
    @confidence >= 0.95
}
```

A tag filter is `@name <operator> <value>`. The operators:

| Operator | Meaning |
|----------|---------|
| `=` `==` | equals |
| `!=` `<>` | not equal |
| `<` `<=` `>` `>=` | ordered comparison (numeric or date, see below) |
| `CONTAINS` | tag value contains the given text |
| `STARTS_WITH` / `ENDS_WITH` | text prefix / suffix |
| `EXISTS` | the fact carries this tag (no value) |
| `NOT_EXISTS` | the fact does **not** carry this tag (no value) |
| `INCLUDES` | scope coverage — see [Asking within a context](#asking-within-a-context-scope) |

Values may be a quoted string, a number, `true`/`false`, `null`, an entity
(IRI / prefixed name), or `TODAY()` / `NOW()` for the execution clock. **Dates must
be quoted** (`"2026-07-06"` or a full ISO 8601
datetime) — a bare `2026-07-06` is read as arithmetic. Comparison is
**type-aware**: if both sides look like dates it compares as time, else if both are
numeric it compares as numbers, otherwise as text. So `@confidence > 0.9` is
numeric, `@validFrom >= "1965-01-01"` is temporal, and `@source = "wikipedia"` is
text — automatically.

### Binding a tag into a variable

A tag pattern can also **bind** instead of filter: `@name ?var` puts the matched
fact's tag value into a variable, usable in the projection, `GROUP BY`, and
`ORDER BY` like any other:

```
# Which source asserted each revenue figure?
FIND ?product ?rev ?src WHERE {
    ?product :revenue ?rev
    @source ?src
}

# The provenance census: everything in the store, by source
FIND ?src COUNT(?s) AS ?n WHERE {
    ?s ?p ?o
    @source ?src
} GROUP BY ?src ORDER BY DESC(?n)

# Monthly figures with their period, one row per observation
FIND ?item ?units ?month WHERE {
    ?item :unitsSold ?units
    @context ?month
}
```

The census query is the canonical way to discover what source values exist
before filtering by one — importers stamp `@source` with the source's name as
recorded at load time, which is not always the exact filename you remember.
Facts that do not carry the bound tag are dropped from the result (the binding
requires the tag), so `@source ?src` also acts as an `EXISTS` check.

Several tag patterns can follow one triple pattern — each binds or filters
against the same matched fact, so you can pull a fact's whole provenance in
one row:

```
FIND ?s ?u ?period ?src WHERE {
    ?s :unitsSold ?u
    @context ?period
    @source ?src
}
```

### Auditing extracted and imported facts

Machine-written facts carry their provenance as tags, which makes quality
control a query rather than a hunt. Document extraction stamps every fact it
writes with `@extractedBy` (the model that produced it), `@source` (the
document), `@confidence`, and `@page` — and nothing else writes
`@extractedBy`, so it cleanly separates machine-written facts from
human-authored ones.

```
# Everything extraction ever wrote, with the model that wrote it
FIND ?s ?p ?o ?model WHERE {
    ?s ?p ?o
    @extractedBy ?model
}

# Everything asserted by one document
FIND ?s ?p ?o WHERE {
    ?s ?p ?o
    @source = "quarterly-report.pdf"
}

# The review queue: weakest-attested facts first, with the page to check
FIND ?s ?p ?o ?conf ?page WHERE {
    ?s ?p ?o
    @extractedBy EXISTS
    @confidence ?conf
    @page ?page
} ORDER BY ?conf

# The audit census: how much came from where, written by what
FIND ?src ?model COUNT(?s) AS ?n WHERE {
    ?s ?p ?o
    @source ?src
    @extractedBy ?model
} GROUP BY ?src ?model ORDER BY DESC(?n)
```

Because every machine-written fact is scoped by these tags, remediation is
equally precise: facts that fail review can be retracted by the same tag
scope — everything from one source, written by one model — without touching
anything a person authored.

### The tag catalog

Any `@name` is accepted in a query, so custom tags your team defines are queryable
too. These are the built-in tags:

| Tag | What it records |
|-----|-----------------|
| `@confidence` | how certain the fact is (0–1) |
| `@validFrom`, `@validUntil` | validity window `[from, until)` — drives `AS OF` |
| `@context`, `@within` | the scope a fact holds in — drives `INCLUDES` |
| `@layer` | which layer the fact lives in (see values below) |
| `@source` | where the fact came from |
| `@authority` | the authority that stands behind it |
| `@language` | language of a text value |
| `@rule`, `@derivedFrom` | how an inferred fact was computed |
| `@created`, `@updated` | authoring timestamps |
| `@extractedBy`, `@matchStatus` | import / entity-resolution provenance |
| `@list` | ordered-collection membership |

Only the temporal (`@validFrom`/`@validUntil`) and scope (`@context`/`@within`) tags
have special engine behavior — everything else is a straight value comparison.

`@layer` accepts these values (quote them): **`instance`** (the default when a fact
has no explicit layer), **`ontology`** (the schema), **`vocabulary`**,
**`inferred`**, and **`alignment`**. There is no `schema` value — the schema layer
is `ontology`.

```
FIND ?s ?p ?o
WHERE {
    ?s ?p ?o
    @layer = "inferred"      # only facts the engine computed, not asserted ones
}
```

### Reading a tag into a result

Bind a tag's value to a variable by naming a variable after the tag. This is an
inner join — rows whose fact lacks the tag are dropped:

```
FIND ?product ?price ?confidence
WHERE {
    ?product :price ?price
    @confidence ?confidence
}
ORDER BY ?confidence DESC
```

## Negation and absence

QQL can ask for the **absence** of something — the backbone of data-quality
queries.

**`FILTER NOT EXISTS { ... }`** keeps a row only when the inner pattern finds no
match. This is how you express "has no such triple":

```
FIND ?person
WHERE {
    ?person :type :Person
    FILTER NOT EXISTS { ?person :label ?label }
}
```

`FILTER EXISTS { ... }` is the positive form — keep the row only if a match exists.

**`@tag NOT_EXISTS`** tests the absence of a **tag** on the matched fact (not the
absence of a triple):

```
FIND ?s ?p ?o
WHERE {
    ?s ?p ?o
    @source NOT_EXISTS       # facts with no recorded source
}
```

Within an expression you also have `NOT` / `!`, `NOT IN (...)`, and `IS NOT NULL`.
There is no `MINUS` keyword — the two `NOT EXISTS` forms above are the negation
primitives.

## Asking about a point in time

Use `AS OF` to see the data as it was valid at a moment. Validity is a half-open
interval `[validFrom, validUntil)`; facts with no validity bounds always pass:

```
FIND ?price WHERE { :ProductX :price ?price } AS OF "2024-06-15"
```

`AS OF` also accepts an **entity**, not just a date — useful when a thing's validity
is anchored to another thing (for example, a release rather than a printed date).
Reprints and re-issues resolve to their origin:

```
FIND ?price WHERE { :ProductX :price ?price } AS OF :OriginalRelease
```

You can also filter directly on the validity tags without `AS OF` — handy for
finding facts by their window rather than resolving a snapshot:

```
FIND ?s WHERE {
    ?s :status ?status
    @validFrom <= "2024-06-15"
    @validUntil > "2024-06-15"
}
```

### Be as specific as you can — when you tag, and when you ask

`AS OF` puts the temporal question to **every** fact the query touches. That is the
right thing when you genuinely want a snapshot of the whole answer. But most facts
in a query carry no time dimension at all: what something *is* — its type, its
name, the class it belongs to — is timeless, and stays timeless no matter which
moment you ask about. Usually only one statement in the pattern actually changes
over time.

So aim the temporal test at that one statement. Filter its validity tag directly:

```
FIND ?person WHERE {
    ?person :type :Employee
    ?person :assignedTo ?team
    @validUntil > TODAY()
}
```

The tag filter applies to the statement written immediately above it — here, the
assignment, which is the only fact carrying a window. The type statement is left
alone; it records what the person is, and that has no window to test.

The broader form puts the same question to everything:

```
FIND ?person WHERE {
    ?person :type :Employee
    ?person :assignedTo ?team
} AS OF TODAY()
```

Both are correct, and neither depends on where you write the patterns — a fact with
no window holds at every instant, so the timeless statements pass either way. They
differ in one respect worth knowing: the tag filter requires the window to be
present and open, while `AS OF` also admits a fact that carries no window at all. Where
you can name the fact that carries time, prefer the specific form — it says which
fact the question is about, and the query does less work to answer it.

The same discipline applies when you write the data: put `@validFrom` / `@validUntil`
on the statements that really do change, and leave the rest untagged. A validity
window on a fact that has no time dimension is a claim you will have to keep true
forever.

## Asking about today

`TODAY()` and `NOW()` are the clock. `TODAY()` is midnight UTC of the day the query
runs; `NOW()` is the instant it runs. They are read when the query executes, not
when it is written, so a saved query keeps meaning "today" tomorrow. Both take no
arguments.

They work in the two places time matters, and both read the same execution instant
within one query.

**As the temporal point.** `AS OF TODAY()` asks for the data valid right now — a
fact whose validity window closed yesterday drops out, one with no bounds stays:

```
FIND ?s WHERE { ?s :status ?st } AS OF TODAY()
```

**In a `FILTER`, against a date your data holds as an ordinary fact.** Most work
records time as plain dates on the thing — a start date, a completion date — not as
a validity window. Compare those to `TODAY()` directly:

```
FIND ?t WHERE {
    ?t :completedOn ?d
    FILTER (?d > TODAY())
}
```

That is the "still open" test. The "in progress" shape — started, not yet finished —
is the two dates together:

```
FIND ?job WHERE {
    ?job :startedOn ?start
    ?job :completedOn ?complete
    FILTER (?start <= TODAY() AND ?complete > TODAY())
}
```

The comparison is temporal, not textual: it works whether the date was stored as a
date-typed value or as a plain date literal.

An evergreen set: a rule with no `THEN`
---------------------------------------

A rule that carries a `WHEN` but no `THEN` is a **selector**: it names a set
without asserting anything. Nothing is written, so nothing can go stale — the set
is worked out whenever you refer to it, against the clock at that moment.

```
"In Progress" :
WHEN  ?job :startedOn ?start .
      ?job :completedOn ?complete .
      FILTER (?start <= TODAY() AND ?complete > TODAY())
```

The set is worked out when the rule is referenced — as the scope of a fact
(`@context = rule:In Progress` on the statements it governs, which `INCLUDES`
then reads), or by the derivation runner when another rule names it.

This is the right shape for anything defined by "as of now". A rule that *derives*
facts from `TODAY()` would bake this morning's answer into the store and be wrong
by tomorrow; a selector re-reads the calendar every time it is consulted. To *list*
the members yourself, ask the same question directly as a query — the `FILTER`
form above returns exactly the set the selector names.

## Asking within a context (scope)

Facts can hold within a context — a market, a story, a project. Use `INCLUDES` to ask
whether a fact's scope covers something:

```
FIND ?actor WHERE {
    :LukeSkywalker :isPlayedBy ?actor
    @context INCLUDES :StarWarsEpisode4
}
```

`INCLUDES` understands all three scope forms: a specific entity (or set of entities),
an entire class, or a named rule. So a fact scoped to a class or a rule is correctly
found by a query that names a member of it. `@within` is queried the same way:

```
FIND ?price WHERE {
    :ProductX :price ?price
    @within INCLUDES :US
}
```

Filtering on an individual **dimension** of a multi-dimensional `@within` (e.g. the
market separately from the currency) is not yet a query-layer feature — a scope is
matched as a whole via `INCLUDES`.

## Rules (`@rules`)

A `.tqs` schema can carry named rules in an `@rules { … }` block. Each rule **must**
have a quoted name, and its `WHEN` (and optional `THEN`) bodies use the **same WHERE
grammar as a query** — triple patterns, `.`/`;` separators, `FILTER`, and
`FILTER NOT EXISTS { … }`:

```
@rules {
    "FullyCertifiedOrder" :
        WHEN  ?order :hasSupplier ?lead .
              ?lead :hasCertification :Certified .
              FILTER NOT EXISTS {
                  ?order :hasSubcontractor ?sub .
                  ?sub :hasCertification ?c .
                  FILTER (?c != :Certified)
              }
        THEN  ?order :supplyChainCategory :FullyCertified .
}
```

Two shapes:

- **Selector** (no `THEN`) — a named set the scope grammar references as
  `rule:<name>` (Form 3). Its `WHEN` denotes the set; nothing is derived.
- **Derivation** (`WHEN … THEN …`) — every solution of `WHEN` instantiates the
  `THEN` triples as new facts in the *inferred* layer, tagged with the rule name and
  the facts they came from. `THEN` holds triple templates only (no `FILTER`), and
  every `THEN` variable must be bound by a **positive** `WHEN` pattern — a variable
  used only under `FILTER NOT EXISTS` cannot be derived.

The naming requirement is strict: a rule written without `"Name" :` is rejected at
import with a diagnostic pointing at the missing name.

Derived facts are recomputed as a pure function of the base facts whenever the data
is (re)loaded. Rules are grouped into strata so a `FILTER NOT EXISTS` over a
*derived* predicate only runs once that predicate is fully materialized; a rule set
that recurses through negation is rejected rather than producing an undefined result.

## Aggregates and grouping

`FIND` carries aggregate projections directly. The aggregate functions are `COUNT`
(including `COUNT(*)`), `SUM`, `AVG`, `MIN`, `MAX`, `SAMPLE`, and
`GROUP_CONCAT(?v; SEPARATOR = ", ")`.

`COUNT` and `GROUP_CONCAT` also accept `DISTINCT`: `COUNT(DISTINCT ?v)` counts
distinct values rather than rows, and `GROUP_CONCAT(DISTINCT ?v; SEPARATOR = ", ")`
drops duplicate values (first-seen order) before joining. DISTINCT matters whenever
the `WHERE` pattern matches independent multi-valued properties: a model with two
inputs and two outputs matches four rows, so a plain `GROUP_CONCAT(?input)` lists
each input twice. The row-collapse idiom — one line per entity, each multi-valued
property as a list:

```
FIND ?name
  GROUP_CONCAT(DISTINCT ?input; SEPARATOR = ", ") AS ?inputs
  GROUP_CONCAT(DISTINCT ?output; SEPARATOR = ", ") AS ?outputs
WHERE {
    ?model :type ?class . ?class :subClassOf+ :Model .
    ?model :label ?name .
    ?model :has_input ?i . ?i :label ?input .
    ?model :has_output ?o . ?o :label ?output .
}
GROUP BY ?name
```

```
FIND ?manufacturer COUNT(?product) AS ?count
WHERE {
    ?product :type :Product
    ?product :manufacturedBy ?manufacturer
}
GROUP BY ?manufacturer
HAVING (?count > 5)
ORDER BY ?count DESC
```

`GROUP BY` sets the buckets; `HAVING` filters the buckets after aggregation (as
opposed to `FILTER`, which filters rows before).

## Ordering and paging

`ORDER BY` takes one or more expressions, each optionally wrapped in `ASC(...)` or
`DESC(...)` (ascending is the default). `LIMIT` caps the row count; `OFFSET` skips
rows — together they page results:

```
FIND ?product ?price
WHERE { ?product :price ?price }
ORDER BY DESC(?price)
LIMIT 20
OFFSET 40
```

## Similarity search

When an instance has vector embeddings configured, `SIMILAR TO` filters the `WHERE`
results to those whose fact is semantically close to the query, ranks them by
closeness (closest first), and can bind the score:

```
FIND ?doc ?score
WHERE { ?doc :type :Document }
SIMILAR TO "battery life complaints" LIMIT 10 THRESHOLD 0.7 AS ?score
```

`THRESHOLD` is a closeness **floor** on a 0–1 similarity scale (1 = identical): only
rows at least that similar are kept. `LIMIT` then caps the ranked set. Similarity
search depends on an embedding index being available on the instance; if none is
configured the clause has nothing to match.

## The support set — what the answer was computed over

When a query uses `AS OF` or filters on scope/validity tags, QQL returns a
**support set** alongside the results:

- `asOf` — the resolved instant the answer was computed at.
- `statementCount` — the number of distinct statements that contributed, counted
  **before** any `LIMIT`. This is the denominator behind an aggregate.
- `subjects` — a sample of the contributing subjects (up to 100), with
  `subjectsTruncated` set when there were more.

If a count or an average looks off, the support set shows you the basis — how many
facts, and which subjects — so you (or an LLM grounding on the result) can judge the
answer rather than trust it blindly.

## Data-hygiene queries

Because QQL can ask for absence and read tags directly, it doubles as a
data-quality tool. Some recipes:

**Instances of a class with no label**

```
FIND ?x
WHERE {
    ?x :type :Product
    FILTER NOT EXISTS { ?x :label ?label }
}
```

**Typed things that are missing a required property** — e.g. products with no price

```
FIND ?x
WHERE {
    ?x :type :Product
    FILTER NOT EXISTS { ?x :price ?price }
}
```

**Facts with no recorded source**

```
FIND ?s ?p ?o
WHERE {
    ?s ?p ?o
    @source NOT_EXISTS
}
```

**Low-confidence facts** — surface what to review

```
FIND ?s ?p ?o ?confidence
WHERE {
    ?s ?p ?o
    @confidence ?confidence
    @confidence < 0.5
}
ORDER BY ?confidence
```

**Facts that have already expired** — validity ended before today. A tag
comparison takes a date, or `TODAY()` / `NOW()` to read the clock at execution:

```
FIND ?s
WHERE {
    ?s :status ?status
    @validUntil < TODAY()
}
```

**Facts with no validity window at all** — timeless assertions you may want to
date-stamp

```
FIND ?s ?p ?o
WHERE {
    ?s ?p ?o
    @validFrom NOT_EXISTS
}
```

## Advanced

- **`EXPLAIN`** in front of a query returns the execution plan instead of running it
  — useful for understanding how a slow query is evaluated.
- **`WITH PROVENANCE`** attaches the contributing statements (subject, predicate,
  object, source, confidence) to each result row. It goes at the **end** of the
  query, after every other clause.

## Notes

- The Query page on a QROS instance is **read-only**. You author and edit data
  through the entity, vocabulary, ontology, and rules surfaces — see the
  [Operations Guide](administrators/OPERATIONS_CLOUD.md).
- Coming from another graph query language? QQL keeps `FIND`/`WHERE` familiar but drops reification and
  named-graph gymnastics: a fact's metadata is queried directly through its tags, and
  there is one `:` namespace for core vocabulary. `FILTER NOT EXISTS`, `OPTIONAL`,
  `UNION`, `BIND`, property paths, and the aggregates all carry over; there is no
  `MINUS` and no `VALUES`.
