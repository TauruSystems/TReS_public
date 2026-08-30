# Ontology Guide

How to author and edit a QROS ontology in TReS — the model your data is shaped by,
and the contract that makes facts trustworthy and queryable. Companions: the
[Vocabulary Guide](VOCABULARY_GUIDE.md) for the taxonomist Vocabulary-tab workflow,
the [QQL Guide](QQL_GUIDE.md) for querying, and the [Import Guide](IMPORT_GUIDE.md)
for bringing existing data in.  This guide uses examples from the public dataset and ontology in the Preview instance of TReS at tres.taurusystems.com. You can see the ontology there and interact with the data through displays, queries, and natural-language questions (the **LLM** item in the main menu).

## What a QROS ontology is

A QROS ontology is an agreement about what your data *means*: the kinds of things
(classes), how they relate (properties), the controlled values they draw on
(vocabulary), and — uniquely to QROS — the **context every fact carries** (tags),
the **scope** a fact holds in, the **rules** that compute new facts, and the
**contract** each predicate's statements must satisfy. It is not a separate modeling
project that lives apart from the data: facts and the ontology share one store, and a
well-formed ontology is what lets you — and your AI — trust and query the data. 

IMPORTANT: A "fact" in this context is a statement (a "triple" in RDF terms) plus
its tags: subject + predicate + object + tags = fact. The same subject, predicate,
and object held in different **contexts** — different scope tags such as `@context`
or `@within` — are different facts. Purely descriptive tags (`@confidence`,
`@source`) qualify a fact rather than create a new one: sources accumulate on the
same fact, and confidence keeps the maximum.

## The model: base classes and anchoring

Every class in a QROS ontology descends from one of **eleven universal base classes**. In other words, your classes must be subClasses of one of the following. You may create instances of the base classes directly to avoid unnecessary subClasses.

`:Person` · `:Species` · `:Organization` · `:Event` · `:Place` · `:TangibleObject` ·
`:CreativeWork` · `:Concept` · `:Action` · `:DataSet` · `:Vocabulary`

A domain class **anchors to exactly one** base via `:subClassOf`:

```
:Model :type :Class ; :subClassOf :CreativeWork ; :label "Model" .
:DiffusionModel :type :Class ; :subClassOf :Model ; :label "Diffusion Model" .
```

NOTE: Unlike in other graph stores, every entity must be an instance of a class:
authoring `:xyz123 :somePredicate :someObject` without also declaring
`:xyz123 :type :SomeClass` breaks the model's core contract. Today the store will
accept such a statement — but an untyped entity is invisible to constraint
validation and classification, so nothing will ever check or correct its facts.
Always type your entities.

**The 3-level guidance.** Keep the class chain at most three levels deep *including
the base* (`:CreativeWork → :Model → :DiffusionModel`). A hierarchy deeper than that
is a taxonomy, and a taxonomy belongs in **vocabulary**, not the class tree. This is
guidance, not a rule the system enforces: the store accepts deeper chains and queries
operate over any depth without complaint. When you find yourself adding a fourth
level, that's the signal to re-model — usually by moving the depth into vocabulary —
not a condition the system balks at.

**Class vs. vocabulary.** Use a class when instances have their own properties and
identity (a `:Model`, a `:Task`). Use a vocabulary term for a controlled value selected
from a list (a function, a media type, a license, a job title). Crucially, a vocabulary
term is still a first-class entity, so it can **carry relationships** — the function
`:VideoGeneration` can declare `:consumes :Text ; :produces :Video`, and a model that
`:hasFunction :VideoGeneration` inherits that I/O contract. Collapsing a deep taxonomy
into vocabulary loses nothing.

**The deeper test: function vs. metadata.** The surest way to tell a class from a
vocabulary is to ask how the data behaves. **Functional (transactional)** entities are
minted through normal use — the system creates new *cases*, *tasks*, or *products* as
people work. **Vocabulary** is the controlled reference set those records draw on —
*license types*, *functions*, *classification codes*, *regions* — which changes slowly
and only by deliberate curation, never as a byproduct of daily operation. Vocabulary is
metadata *on* your functional data. That is why it lives in its own `@layer vocabulary`
partition (carrying its own small set of classes), identified by that layer rather than
by any single named graph, and why the layer doubles as the controlled-value glossary an
AI assistant grounds against. See the [Vocabulary Guide](VOCABULARY_GUIDE.md) for the
full model.

## Properties

Properties relate things. Declare each with a `:domain` (the subject's class) and a
`:range` (the object's class or datatype):

```
:hasFunction :type :ObjectProperty ; :domain :Model ; :range :Function .
:benchmark   :type :DatatypeProperty ; :domain :Model ; :range xsd:decimal .
```

- **Object properties** link to another entity (`:range` is a class).
- **Datatype properties** carry a literal value (`:range` is an `xsd:` type).

## Tags: context on a fact

A fact is a subject-predicate-object statement that can carry **tags** — structured
metadata attached directly to the statement, with no extra nodes. Where you author a
statement (entity authoring, the ontology editor), tags are editable fields and show on
the entity as chips.

Every built-in tag is listed below, grouped by what it does. The "Written by" column
is the important one: some tags are yours to author, and some are stamped by the
engine or by a workflow — you will see those on facts, but you don't add them by hand.

**General fact metadata** — the everyday tags you author on a fact:

| Tag | Meaning | Written by |
|---|---|---|
| `@confidence` | How certain the fact is, 0.0–1.0 | You |
| `@validFrom` / `@validUntil` | When the fact is/was true (a date, or an entity it's anchored to) | You |
| `@context` | The single context the fact holds in (see Scope) | You |
| `@within` | Multi-dimensional context — named dimensions, each with a value (see Scope) | You |
| `@source` | Where the fact originally came from | You; imports stamp it too |
| `@authority` | Source precedence (primary / secondary / tertiary) | You |
| `@language` | The language of a text value, as a language code (`en`, `ja`) | You |
| `@list` | Marks the fact as a member of the list formed by its subject and predicate; an optional number gives its position | You |

**Schema constraints** — authored on a property declaration, they state what the
data should look like:

| Tag | Meaning | Written by |
|---|---|---|
| `@required` | A value for this property is required | You |
| `@minCardinality` | Minimum number of values | You |
| `@maxCardinality` | Maximum number of values | You |

**Citation locators** — pinpoint where in a source document a fact is supported:

| Tag | Meaning | Written by |
|---|---|---|
| `@page` | Page in the source document | You |
| `@section` | Section in the source document | You |

**Engine-written (not hand-authored)** — these appear on facts automatically; they
carry provenance and workflow state, and you read them rather than write them:

| Tag | Meaning | Written by |
|---|---|---|
| `@layer` | The partition the fact lives in (ontology / instance / vocabulary / inferred / alignment) | The engine |
| `@created` / `@updated` | Creation and update provenance: who/what/when, with reason | The engine |
| `@rule` | The rule that computed an inferred fact | The rules engine |
| `@derivedFrom` | The statements an inferred fact was derived from | The rules engine |
| `@importedAt` | When the fact was imported | The import pipeline |
| `@convertedFrom` | The original form the fact was converted from | The import pipeline |
| `@extractedBy` / `@extractedFrom` | The extractor that produced the fact, and the document it came from | A document-extraction pipeline (not yet delivered; updated 2026-08-28) |
| `@matchStatus` / `@matchEvidence` / `@matchScore` | Outcome, evidence, and score of an instance-resolution match | The resolution workflow |
| `@rejectionReason` / `@reasoning` | Why a match was rejected; the reasoning behind a decision | The resolution workflow |
| `@supersededBy` | The newer statement that replaces this one | The resolution workflow |
| `@proposedBy` / `@confirmedBy` / `@confirmedAt` | Who proposed an alignment, who confirmed it, and when | The alignment workflow |
| `@measuredAt` / `@method` / `@benchmark` / `@environment` | When, how, against what, and where a quality measurement was taken | The measurement workflow |

Tags ride with the fact — you don't manage them as separate records — and QQL queries
them directly.

**`@source` records data lineage, not the upload event.** It names where a fact
originally came from, and it survives format conversion and re-upload: convert a
source file to native QROS and load the converted file, and the facts still carry
the original file as their `@source` — that is the point. Who uploaded which file,
when, lives in the audit log. If `@source` were rewritten on every load, the real
lineage would be lost each time data changed containers.

## Tag contracts: a predicate's tag-shape

The ontology can declare which tags a predicate's statements should carry, so the right
context is captured consistently rather than left to memory. Tags are declared as `:Tag`
entities, each with the kind of value it holds (`:literal`, `:date`, or `:entityRef` —
the last being a link to another entity):

```
:metric  :type :Tag ; :label "metric"   ; :tagValue :literal .
:runDate :type :Tag ; :label "run date" ; :tagValue :date .
:source  :type :Tag ; :label "source"   ; :tagValue :entityRef .
```

A predicate then names the tags its statements **require** and the ones they **may** carry:

```
:benchmark :requiresTag :metric ; :requiresTag :runDate ;
           :allowsTag :testSet ; :allowsTag :source ; :allowsTag :confidence .
:approved  :requiresTag :by ; :requiresTag :on ;
           :allowsTag :validFrom ; :allowsTag :validUntil .
```

This makes the tag-shape part of the ontology: an authoring surface can prompt for the
required tags, and the AI reads the contract so it captures — and later queries — the
right tags (knowing an `:approved` fact carries `@validUntil`, and so reaching for a
point-in-time `AS OF` query). It is also what makes rich context *easy* to author: one
tagged statement replaces a multi-node modeling pattern.

## Constraints

Beyond the tag-contract, an ontology can declare integrity constraints — required
properties, cardinality (min/max), and disjointness between classes. Validation against
these reports violations as findings (like the data-quality check), never silently
dropped and never blocking a load (updated 2026-08-28).

## Scope: facts that hold in a context

Many facts are only true *within* something — a price within a market, a role within a
story, an approval within a use. Rather than build intermediate nodes (n-ary
reification), QROS keeps one statement and qualifies it with **scope**:

```
:CaptainVael :portrayedBy :NoraKell @context = :StarfarerOrigins
:ProductX :price "19.99" @within({ market: :US, currency: :USD })
```

A scope specifier takes one of three forms — a ladder from specific to general:

1. **An entity** (or an explicit set) — "in these things."
2. **A class** — "in any instance of this class."
3. **A rule** — "in whatever this named rule selects."

You author scope in whichever form is natural; QROS gives a scoped set a stable identity
so the same scope written different ways doesn't fragment. Query it with
`@context INCLUDES …`, which understands all three forms (see the QQL Guide).

## Validity that resolves late

`@validFrom` / `@validUntil` accept a date **or** an entity. The entity form matters when
validity is anchored to another thing — a reprint should resolve to the original release,
not the date on the copy in hand. An administrator declares the anchoring relationship
once (a temporal-anchor property); QROS then resolves entity-valued bounds to their origin
automatically, and `AS OF` queries consume it.

**Validity vs. recency.** A `@validUntil` says when a fact *stops being true* (an approval
lapses). A measurement date like `@runDate`, with no `@validUntil`, does **not** expire —
several dated readings are all true at once, and "current" means the most recent. Model
the difference deliberately: approvals are validity-bounded; benchmark runs are recency-
stamped.

## Rules: computing scope and inferences

A rule is a `WHEN` / `THEN` pattern, defined in the `@rules` block (and editable on the
Rules surface, with a live Test preview):

```
WHEN  ?x :locatedIn ?y . ?y :locatedIn ?z
THEN  ?x :locatedIn ?z
```

Inferred facts are kept separate (tagged `@layer=inferred`) and carry their lineage — the
rule and the source facts — so any inferred result is explainable.

Rule matching also honors the vocabulary ladder: classification **propagates up
`:broader` automatically** (no concept scheme involved — `:broader` links terms
directly). A fact whose value is a narrower term derives the same fact against
every broader ancestor before rules evaluate, so a rule conditioned on the
broader term matches narrower-classified data. Propagated facts carry
`@rule=BroaderPropagation`, their source lineage, and a confidence no higher
than the weakest `:broader` link in the chain. A rule with a `WHEN`
but no `THEN` is a **selector**: it defines a set without asserting anything, usable as a
query scope (`@context INCLUDES rule:<name>`). Reference a saved rule anywhere a scope is
accepted as `rule:<name>`.

## The native format

QROS stores an ontology and its data as human-readable native text — diffable, and
the canonical form. You rarely hand-write it (the editor and import produce it), but
authoring or reviewing it is straightforward.

### Two file types — the functional split

| File | Holds (`@layer`) | It is… |
|---|---|---|
| **`.tqs`** (schema) | `ontology`, `vocabulary`, and the `@rules` block | the curated, slowly-changing substrate: your class/property model and controlled vocabulary |
| **`.tqd`** (data) | `instance`, `inferred`, `alignment` | transactional and derived facts: records minted through use, rule-derived facts, and external-URI (`sameAs`) links |

The split mirrors the **functional-vs-metadata** distinction (see *Class vs.
vocabulary*, above): `.tqs` is what a curator defines, `.tqd` is what the system
accumulates. A controlled vocabulary — its classes *and* its terms — is curated
reference data, so a self-contained vocabulary file is a **`.tqs`**.

### Anatomy of a file

```
@format tqs 1.1
@prefix : <urn:tres:> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

!# Authoring notes go in a block comment like this — stripped on load,
   never stored. Use one block for a big note, not stacked # lines. #!

@layer ontology {
    :Region :type :Class ;
        :subClassOf :Place ;
        :label "Region" ;
        :definition "A geographic area used to group locations." .
}

@layer vocabulary {
    :NorthAmerica :type :Region ;
        :label "North America" ;
        :scopeNote "Loaded from the standard region list; folded to :Place." .
}
```

- **`@format <tqs|tqd> 1.1`** — the first line. (1.0 files still load.)
- **`@prefix`** — namespace prefixes. The **blank prefix `:` is QROS's own
  namespace**, so `:Region` is a QROS term; use other prefixes for external URIs.
- **`@layer <name> { … }`** — groups statements by layer; everything inside carries
  that layer.
- **Statements** read subject–predicate–object. A `;` continues the same subject; a
  `.` ends it. Typed literals use `"value"^^xsd:type`.
- **Tags** attach context inline — `@created`, `@source`, `@context` (scope),
  `@code`, `@citation`, … — the machinery covered under *Tags*, above.
- **Comments**: `#` to end of line; `!# … #!` for a block (both stripped on load).

### Give the ontology itself a definition

The ontology file's header entity — the subject typed `:Ontology` — should
carry a **`:definition` stating what the model is for**, alongside its
`:versionInfo` and version-history `:comment`s:

```
<http://example.org/fleet> :type :Ontology ;
    :versionInfo "v. 2" ;
    :definition "Connects vehicles, their maintenance tasks, and the
        certified providers allowed to perform them, so schedulers can
        find an approved provider for any task without knowing the
        certification codes." .
```

Write it as a purpose statement, not a content inventory: *who* uses the
model, *what question* it answers for them, and *which link* makes that
possible. It earns its keep twice. For **teams**, it is the one-paragraph
orientation a new curator or reviewer reads before touching anything — as
soon as a model has more than one author, undocumented intent starts
drifting. For the **AI assistant**, it is intent: retrieval grounds the
model in what the classes *are*, and the definition tells it what the
model is *for*, which shapes how an ambiguous question should be read.

### A file never names its graph

The named graph a file loads into is **not** written in the file — it is chosen at
**import** (the folder maps to a graph, or you pick one in the import graph
selector). A file declares only its `@layer`; the layer rides along with the content
into whatever graph you import it to. This keeps two management axes independent:
**layers** are a fixed, functional classification (the five above); **named graphs**
are yours to spend on organization, promotion, or lenses. There are no custom
layers; there can be as many custom named graphs as you like.

(Older 1.0 files may carry an `@graph <…>` header. Such a file still loads, but the
graph designation is ignored — the import assigns the graph.)

### Where construction notes go

When you record *why* a term was modeled a certain way — a base-folding decision, a
converter behavior, a source-ontology quirk — put it in **`:scopeNote`**. It shows in
TReS detail panes for curators but is kept out of public documentation exports.
Reserve **`:definition`** and **`:comment`** for the public-facing meaning of the
term; those *do* appear in published docs.

## Formal schema definition

Everything above explains *why* the model is shaped the way it is. This section is
the reference: the complete set of things a schema file may declare, and what each
one means. Nothing outside this list is interpreted — an unrecognized predicate is
stored as an ordinary fact, not as a schema instruction.

### Layers

Every statement belongs to exactly one layer, set by the enclosing `@layer` block.

| Layer | Lives in | Holds |
|---|---|---|
| `ontology` | `.tqs` | classes, properties, constraints, tag contracts |
| `vocabulary` | `.tqs` | controlled-vocabulary terms |
| `instance` | `.tqd` | records and their facts |
| `inferred` | `.tqd` | facts a rule derived (recomputed, never hand-edited) |
| `alignment` | `.tqd` | links to external URIs |

### Declarable types

A subject becomes part of the schema by being typed as one of these.

| `:type` object | Declares |
|---|---|
| `:Class` | a class |
| `:Property` | a property, kind unspecified |
| `:ObjectProperty` | a property whose values are entities |
| `:DatatypeProperty` | a property whose values are literals |
| `:Tag` | a tag a property may require or allow |
| `:Ontology` | the file's header entity — carries the model's own definition and version |
| `qros:TemporalAnchor` | marks a property as the store's date-resolving anchor for entity-valued validity bounds |

### The eleven base classes

Fixed, reserved, and always present. Every domain class anchors to exactly one of
them, directly or through its ancestors.

`:Person` · `:Species` · `:Organization` · `:Event` · `:Place` ·
`:TangibleObject` · `:CreativeWork` · `:Concept` · `:Action` · `:DataSet` ·
`:Vocabulary`

### Structural predicates

| Predicate | Subject | Object | Meaning |
|---|---|---|---|
| `:type` | any | class or meta-type | category membership |
| `:subClassOf` | class | class | subsumption; the path to a base class |
| `:domain` | property | class | the class whose members carry this property |
| `:range` | property | class or datatype | what its values are |
| `:broader` / `:narrower` | vocabulary term | vocabulary term | term hierarchy, term to term |
| `:baseLanguage` | ontology | language code | the language `:label` is written in |

### Annotation predicates

Human-readable description. Exempt from value-conflict detection — several are
normal.

`:label` (one per subject, in the base language) · `:altLabel` (synonyms and
translations; carries `@language`) · `:definition` · `:comment` · `:scopeNote`
(how a term is meant to be used, including construction notes) · `:note` ·
`:example`

### Tag contracts

| Predicate | Subject | Object | Meaning |
|---|---|---|---|
| `:requiresTag` | property | `:Tag` | every statement using this property must carry the tag |
| `:allowsTag` | property | `:Tag` | statements may carry it |
| `:tagValue` | `:Tag` | value kind | what the tag's value is |

### Reserved tag names

Tags qualify a fact. These names are interpreted by the engine; any other tag name
is stored and queryable but carries no built-in behavior.

| Group | Tags |
|---|---|
| **Scope — participates in a fact's identity** | `@context` (single reference), `@within` (multi-dimensional map), `@validFrom` |
| **Validity** | `@validUntil` (paired with `@validFrom`; the two together bound a fact in time) |
| **Provenance** | `@source`, `@authority`, `@created`, `@updated`, `@confidence` |
| **Derivation** | `@derivedFrom`, `@rule` |
| **Structure** | `@layer`, `@language`, `@list` (ordered or unordered membership) |
| **Constraints** | `@required`, `@minCardinality`, `@maxCardinality` |
| **Resolution** | `@matchStatus`, `@matchEvidence`, `@matchScore`, `@rejectionReason`, `@supersededBy`, `@reasoning` |

The scope tags are the ones that change what a fact *is*: two statements with the
same subject, predicate and value but different scope are two different facts, both
kept. Every other tag qualifies a fact without splitting it.

### Literal datatypes

`xsd:string` · `xsd:integer` · `xsd:decimal` · `xsd:boolean` · `xsd:date` ·
`xsd:dateTime`

Written `"value"^^xsd:type`. An untyped literal is a string.

### Well-formedness

Two different kinds of rule, and it matters which is which.

**Refused at load — the file does not go in, and the error says why:**

1. **`@format <tqs|tqd> 1.1` is the first non-empty line.** A missing header, a
   malformed one, or a `.tqs` presented as `.tqd` is a hard error. (1.0 files
   still load.)
2. **Rule names carry no spaces and may not collide** — including differing only
   by case. A file with an illegal name is refused whole, so the rules in the
   store always match a file you can read.
3. **A replace-mode load that contains only rules** is refused rather than
   clearing the target graph and putting nothing back.

**Required by the model, not currently refused by the loader.** These are the
rules a well-formed ontology follows; today the engine accepts violations and
they surface as data-quality findings rather than load failures. Treat them as
binding on your authoring, not as something the file format will catch:

4. **Every resource is an instance of some class.** A subject with facts but no
   `:type` is a bare fact — nothing in the model can say what it is, and it is
   also skipped by domain/range checking, so it is quieter than it should be.
5. **One `:label` per subject**, in the base language; alternatives and
   translations are `:altLabel`. Conversion from RDF collapses duplicates; a
   hand-authored second `:label` is stored.
6. **A class anchors to a base class.** An unanchored class imported from
   RDF is placed under `:Concept`, and a high count of those defaults is a signal
   the source model is under-defined — worth reviewing rather than accepting.

**Structural, so it cannot be violated:** a file never names its graph. There is
no syntax for it; where statements land is decided at load time.

## Authoring surfaces

| To… | Use… |
|---|---|
| Add/edit an entity and its tagged statements | Entity authoring on the entity page |
| Shape classes, properties, vocabulary, rules | The ontology editor |
| Manage controlled vocabularies / taxonomies (no code) | The Vocabulary tab — see [Vocabulary Guide](VOCABULARY_GUIDE.md) |
| Bring existing graph/spreadsheet data in | Import — see [Import Guide](IMPORT_GUIDE.md) |
| Ask questions of the result | QQL — see [QQL Guide](QQL_GUIDE.md) |

## Why this matters

Classes and properties say what your data *is*; tags, scope, rules, and contracts say
what is **true, who said it, when, and in what context** — and let an AI ground on facts
that carry their own confidence, provenance, and validity. That is the difference between
a store of assertions and queryable knowledge you can defend.

The design rules in this guide are not style preferences. Each one buys a piece of a
single property: **closed-world reasoning over falsifiable data.** A conventional
open-world store can only ever say "not found" — absence means nothing, so every answer
is hedged. A QROS store treats its contents as the whole truth for the question asked,
which lets it say three much harder things: *yes*, with the exact support; *no*, this is
contradicted, with the reason; and *there is no such record* — stated as an answer, not
a shrug. The rules are what make that possible. Typing every entity makes its facts
checkable against its class's constraints — an untyped entity can never be proven wrong,
which also means it can never be proven right. Declaring domains and ranges makes a
mis-modeled statement falsifiable instead of silently ignorable. Scoping facts and
bounding their validity turns "true when, true where?" from a caveat into a computable
question.

For an AI assistant, this is the difference between retrieval and grounding. Over an
open-world store, the assistant retrieves what matches and pads the gaps with
plausibility. Over a closed-world, falsifiable store, its claims are adjudicated by the
engine — a proposed fact comes back supported, contradicted, or not derivable — so it
answers "no" with a reason instead of guessing from an empty result, reports the set it
ranged over instead of a bare figure, and declines to invent what the store cannot
witness. Hard answers, defensibly wrong or defensibly right — never merely plausible.
