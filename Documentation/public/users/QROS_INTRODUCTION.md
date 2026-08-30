# An Introduction to QROS

QROS is a knowledge foundation: where an organization's knowledge is stored,
checked, and served to the applications and AI that use it — independent of any
application, user interface, or backend a product happens to put in front of it.
Under the hood that is a contextual-statement data model, a native closed-world
schema, an embedded store, and a FIND query language, plus an entailment
layer (transitivity, type propagation, property characteristics) and optional
user-defined rules. This document introduces QROS on its own terms. It assumes no
tool and shows no screens.

> **QROS and TReS.** TReS is a cloud-based user platform with data import and curation tools,
> access control, visualization, query, quality testing and an LLM grounding pipeline. QROS is the
> foundation in which TReS stores knowledge. A TReS deployment runs on QROS. This document is
> about QROS itself, not about any surface over it.

## The idea: constraint unlocks meaning

Two graph families dominate today, and each was built for a different job. The
open-web graph standards were designed for open-world publishing and integration on
the Web; their schema-optional, open-world philosophy is maximally flexible, and that
flexibility has a cost: fragmented ontologies, data-quality problems found late,
and a query implementation even experts find demanding. Property graphs (LPG) were
designed for fast traversal — relationship queries, pathfinding, graph
algorithms — where schema, provenance, and governance were never the point.

Both approaches position a graph database near the end of the data delivery pipeline, closest to end user applications while never quite being end user applicatons themselves. Neither was built to *govern data you own and operate* so it is trustworthy enough to ground an AI: closed-world, provenance-bearing, and checkable the moment it is written. That is the job QROS is designed for. It matches LPG on traversal through its vector database, and adds the schema, provenance, and closed-world guarantees the others were never meant to carry. The usual response to these gaps is more machinery.

QROS takes the opposite path: *thoughtful constraint*. Like the structure of a
poem, constraining how data may be represented unlocks meaning rather than limiting
it. Every choice below — fixed base classes, closed-world evaluation, no blank
nodes, content-addressed identity — trades some openness for a property that
matters when data is something you *own and operate* rather than publish to an open
web: predictability. A query returns the same answer twice. A schema can be checked
against data the moment it is written. A language model can be told what "good" looks
like, because "good" is decidable.

## The contextual statement

The unit of knowledge in QROS is a statement: a subject, a predicate, an object —
and, attached directly to it, structured metadata as inline tags.

```
:Paris :population "2165000"^^xsd:integer
    @confidence=0.98
    @validFrom=2024-01-01
    @layer=instance .
```

In older graph standards, attaching that confidence to the fact requires *reification* — a new
resource standing in for the statement, plus several more triples to hang the
metadata on. QROS makes the metadata first-class on the statement itself. A fact
carries its own provenance, certainty, validity, and context, as part of the fact.

Tags fall into a reserved, centralized vocabulary — among them:

- **Layer** — `@layer` ∈ {ontology, instance, vocabulary, inferred, alignment}: the
  statement's role.
- **Provenance** — `@created`, `@updated`: who/what/when/why a fact entered or changed.
- **Temporal validity** — `@validFrom`, `@validUntil` (see *Time*, below).
- **Certainty and authority** — `@confidence` ∈ [0,1], `@authority` ∈ {primary,
  secondary, tertiary}.
- **Inference lineage** — `@derivedFrom`, `@rule`: where a derived fact came from.
- **Contextual scope** — `@context`, `@within` (see *Context*, below).

## Identity: a fact is the same fact everywhere

Every *statement* gets a deterministic identifier computed from its content — a hash
over the normalized subject, predicate, and object. The same fact gets the same id
in every store, which gives automatic deduplication, stable cross-references, and
distribution-friendly identity.

Tags are deliberately *excluded* from that identifier. The same fact may carry
different confidence from different sources, but it remains the same fact. This is a
core QROS principle: **identity is decided by the content of the assertion, not by
the metadata around it.** This is a substantial departure from open-world ontology reasoning.

## Context: collapsing n-ary relationships

The most common reason real ontologies reach for reification is the *n-ary*
relationship — a fact qualified by several dimensions. A price *in a market, in a
currency*. A role *in a story*. Older graph standards model this with an intermediate node and a
cluster of triples. QROS expresses it as one statement plus **scope**, carried in
tags.

```
:LukeSkywalker :isPlayedBy :MarkHamill @context=:StarWarsEpisode4 .

:ProductX :price "19.99" @within({ market: :US, currency: :USD }) .
```

The scope *specifier* takes one of three forms — a ladder from concrete to general:

1. **An entity** (or an explicit set of entities) — an enumerated scope.
2. **A class** — every instance of that class.
3. **A rule** — the set computed by a named inference rule.

The author writes scope in whichever form is natural; its identity is decided by the
specifier and canonicalized per store, so the same scoped fact does not fragment
into many near-duplicates. And because scope is just structured tag data, the query
language reads it directly — there is no separate "is the metadata queryable?"
problem.

## A fixed top to the world: the eleven base classes

Other graph standards let everything be an instance of a universal "Thing". QROS has no universal superclass. Instead,
every domain class must inherit from one of **eleven concrete base classes**:

`:Person` · `:Species` · `:Organization` · `:Event` · `:Place` ·
`:TangibleObject` · `:CreativeWork` · `:Concept` · `:Action` · `:DataSet` ·
`:Vocabulary`

They are **mutually disjoint** by default — an entity cannot be both a Person and an
Organization. Fixing the top of the type lattice this way is what makes reasoning
deterministic, write-time validation reliable, and the world legible to a language
model: there is a known, closed set of upper categories to reason from, not an
open-ended guess.

## Closed world: a fact not stated is false

This is the assumption that most distinguishes QROS. Under the **open-world**
assumption of the open-web standards, a fact you have not stated is merely *unknown* — the system
cannot conclude it is false, only that it is not (yet) asserted. That is correct for
the open Web and limiting for data you own.

QROS is **closed-world**: within the world the data describes, a fact is true if it
is asserted or derivable by an explicit rule, and **false otherwise**. The practical
consequences:

- **Reliable negation.** "Is there a model that does X?" can be answered *no*, and
  the *no* is trustworthy — not an "unknown" in disguise.
- **A computed gap, not a phantom.** Ignorance becomes an *absent statement* you can
  detect and report, not an anonymous unfalsifiable triple sitting in the graph.
- **Decidable quality.** Because "what should be here" can be a schema rule and "what
  is here" is closed, a record is either complete against the rule or flagged — a
  result you can act on.

Closed-world evaluation is what lets QROS treat *honest "I don't have that"* as a
first-class, correct answer rather than a failure.

## Time

Temporal validity is carried by `@validFrom` and `@validUntil`, and a bound may be a
**date** *or* an **entity reference**. The entity form matters for real catalogs: a
reprint or re-release should resolve to the *origin* of the thing, not to the date
of the copy in hand. A declared temporal-anchor property tells the engine which
relationship to walk, and a late-binding resolver maps the entity to its earliest
reachable instant — so reprints resolve to the original date instead of drifting,
while bare-date bounds stay exactly as written. Point-in-time queries (`AS OF`)
then filter to the statements valid at a chosen instant.

## Querying: QQL

QQL is QROS's query language, designed so that the things that are hard to express
elsewhere are first-class: you can query the tags and the scope directly, ask
point-in-time `AS OF` questions, and read an honest *support-set* denominator (the
facts an answer actually rests on). It is also designed to be **reliably generated
by a language model** — a deliberate goal, because an AI that grounds itself in a
knowledge graph has to be able to ask it questions without ceremony.

## What QROS leaves out, on purpose

Predictability comes as much from what is excluded as what is included:

- **No "same-as" identity conflation.** Identity assertions propagate
  unpredictably and conflate context-specific equivalences. QROS records a
  correspondence (`:externalId`) without asserting interchangeability, and resolves
  real-world matches through an explicit, reviewable workflow with confidence and
  provenance.
- **No open-world assumption** — covered above.
- **No blank nodes.** Anonymous existential nodes are useful in open-world reasoning
  and become liabilities under a closed-world, write-validated model (they declare
  no layer, are not locally identity-decidable, and cannot be falsified). QROS gives
  every entity an explicit, stable identifier; what other standards express with blank nodes
  becomes either a content-addressed reified statement or a write-time constraint.
  **Alphabet Soup** QROS collapses the many standards-based namespaces of the older graph stacks into a single default namespace and simplifies the available predicates.

None of these is a rejection of the older standards' design for their purpose — the open Web. They
are the consequences of a different purpose: data you own, operate, and answer
questions about with confidence.

## Why it matters

Two payoffs follow directly from the model, and neither depends on any interface:

1. **More context, captured at the point of authoring.** Because a fact carries its
   scope, provenance, certainty, and validity inline — instead of forcing the author
   to build a reification scaffold — richer context is cheap to record. More of what
   makes a fact trustworthy survives into the store.
2. **A foundation an AI can trust.** Fixed upper classes give a model known
   categories to reason from; closed-world evaluation lets it decline honestly
   instead of hallucinating; first-class context travels *with* each fact into the
   model. A well-built QROS schema is at once a validation spec, a glossary for
   people, and a grounding document for a language model.

## Where to go next

- **`TReS_QROS_Technical_Paper.md`** — the full, rigorous treatment: schema system
  and serialization, the entailment and rules layer, the storage and intelligence layer, the
  measured construction result, and the AI-integration argument in depth.
- **`QQL_GUIDE.md`** — the query language in practice.
- **`ONTOLOGY_GUIDE.md`** / **`VOCABULARY_GUIDE.md`** — building schema and
  controlled vocabularies on the model introduced here.
