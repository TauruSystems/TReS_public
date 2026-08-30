# The TReS Grounding-Evaluation Framework
> **[REVIEW 2026-08-28 — relies on the grounding tests.** Figures or conclusions in this document come from grounding evaluation runs that `design/CLAIMS_LEDGER.md` rates PROVISIONAL or DO-NOT-CITE (C1, C4–C6). Verify against the ledger before any external use. Passages are tagged inline.]

*Companion to the Grounding Paper (§6). A reusable way to measure whether ontology-mediated grounding
actually helps a given (dataset, ontology, model) triple — and to size that answer against production
complexity. Written to be applied to a client's OWN data, not just ours. Includes the methodology, the
recommendations (what/how to test), and — as much as the successes — the trials that did NOT work.*

Status 2026-07-02. Harness lives in `evaluation/`. Evidence: three datasets (Northwind, Nova,
Marvel) × a 4-point ontology axis, plus a complexity×familiarity 2×2.

---

## 1. Why a framework (not just a result)

The headline result is a **scaling law**: ontology value and wrong-ontology harm both grow with data
complexity, and both are invisible below a threshold. [REVIEW: grounding tests] That makes *evaluation design itself a hidden
variable* — the same technology looks worthless or decisive depending on the data you test it on. So the
durable deliverable is not a number; it is a **repeatable procedure a team can run on its own ontology and
model to find out where on the curve they actually sit.** This is the "measure it on your data" kit.

## 2. What is built into TReS (the components)

- **The retrieval-first grounding path** (the thing under test): the ontology-infused VDB RESOLVES +
  RETRIEVES (semantic resolver + `list_instances` via native subClassOf-closure, `traverse` over declared
  domain/range, `find_path`, `get_facts` with contextual tags, `verify_fact` closed-world verdict); the LLM
  only SYNTHESIZES. Ontology quality is load-bearing only through this path.
- **The 4-point ontology axis builder.** Deterministic transforms over a schema, instance data held
  byte-identical: **rich** (as-authored) · **sparse-poor** (strip `subClassOf`/`domain`/`range`) ·
  **misleading-poor** (swap `domain`↔`range`, rotate `subClassOf` targets — systematic, not cherry-picked) ·
  **none** (no schema). Also the finer ablation ladder (poorTax / poorOnt / poorBoth / poorAnno).
  Scripts: `gen_misleading_ontology.py`, `build_*_harm_variants.sh`.
- **Truth-locked discovery banks.** Per dataset, a bank tagged LOOKUP vs DISCOVERY (and optionally
  familiar/mixed/unfamiliar), each question's answer locked by direct query on the store. Banks in
  `banks/`. LOOKUP = control (ontology-invariant); DISCOVERY = treatment.
- **The runner.** `run_factorial.sh` — model × ontology-variant × regime, N trials, one store per cell.
- **The grader.** `grade_judge.py` — truth-based (conclusion vs locked truth), cross-model, comparison-
  tolerant. Keyword grading kept only as an offline sanity cross-check.
- **Deterministic truth.** `qquery` / reference queries for locking answer sets without an LLM.

## 3. How to test — recommendations

1. **Match evaluation complexity AND ontology maturity to production.** This is rule zero. A POC on a
   simplified schema or a preliminary ontology sits in the flat regime and will show nothing — neither the
   value of a good ontology nor the cost of a bad one. [REVIEW: grounding tests] Test at the complexity (entity types, hierarchy
   depth, relationship cardinality, multi-hop distance) your users actually query at.
2. **Split LOOKUP vs DISCOVERY.** Known-subject lookups are ontology-invariant controls (expect a tie);
   discovery (assemble a set you couldn't name in advance) is where the ontology pays or costs. Report them
   separately — a mixed aggregate hides the effect.
3. **Use the 4-point axis, not rich-vs-nothing.** rich/sparse/misleading/none separates two distinct
   questions: *value of an ontology* (rich − none) and *cost of a wrong one* (misleading − none). They move
   differently and both matter.
4. **Lock truth deterministically, from the data/source — not from a capped diagnostic tool** (see §4).
5. **Grade truth-based and cross-model; grade sequentially; human-anchor a sample.** Never let the answerer
   grade itself; never trust a 0-across-the-board without checking raw outputs (see §4).
6. **Pick the right instrument.** To measure *harm* you need a model strong enough to *succeed on the good
   ontology* (something to lose) AND questions that traverse the corrupted axioms. A weak model is a good
   *sensitivity probe for quality degradation* but useless where the task exceeds its capability entirely.
7. **Model the real scenario: private data in a known domain.** Inject unfamiliar/proprietary data (novel
   entities, private figures, internal IDs) into a familiar graph. This is where grounding is most
   decisive — a model reciting priors returns a confidently *incomplete* answer; only the grounded
   substrate is complete. [REVIEW: grounding tests] It also isolates familiarity from complexity.

## 4. What did NOT work — the war stories (and the fix)

Each of these cost a real result before it was caught; they are the framework's most transferable content.

- **A weak local model as the answerer.** gemma4:12b/26b thrash to max-roundtrips or fabricate on discovery
  in *every* ontology condition — the ontology signal drowns in baseline incompetence. [REVIEW: grounding tests] It is the wrong
  instrument for the *harm* test. Fix: use a strong answerer for harm; reserve weak models for the
  quality-degradation sensitivity probe.
- **Keyword / must_not grading.** Tripped on substrings ("in**correct**"), on comparison tables that name
  the rejected value, on ID-vs-name mismatches. A pilot read 41%/25% that was **pure grader artifact** — the
  same answers truth-judged were 100%/0%. Fix: truth-based conclusion grading, comparison-tolerant, with a
  per-question locked `truth`.
- **Bursting the judge.** Firing a full batch of judge calls right after an equal-size answerer run
  exhausted the API credit balance; the grader scored every failed call as wrong — a spurious **0/36 across
  all conditions** that looked catastrophic until the raw answers (present and correct) exposed it. Fix:
  verify credit/rate headroom, grade sequentially, keep a local judge as a credit-free fallback, and always
  sanity-check a 0-everywhere against raw outputs.
- **Trusting a capped diagnostic tool for truth.** `qquery` caps output at 60 rows; naive counts silently
  truncated. Fix: lock discovery truth from the source files / an uncapped reference query, not the CLI.
- **Corrupting axioms the questions never traverse.** The misleading ontology swaps domain/range +
  mis-parents taxonomy — but LOOKUP questions read the intact instance `rdf:type` + label, so they tie in
  every condition (misleading included). Harm only shows where the question actually *uses* the corrupted
  structure (closure, domain/range traversal). Fix: design discovery questions that depend on the corrupted
  axioms; expect the lookup control to tie by construction.
- **Real data fights the toy-clean pipeline.** Marvel's pure-T-box ontology, converted RDF→native, had its
  explicit `owl:Class`/property declarations mis-routed into an *instance* layer (RDF models classes as
  individuals), which the schema loader rejected; the vocabulary labels lived in separate dual-classed SKOS
  files. Fix: a reproducible convert-and-repair step. Lesson: production ontologies arrive mixed and messy —
  budget for data surgery, and script it so it's reproducible.
- **Ephemeral stores.** `/tmp` eval stores vanish between sessions; a re-run silently hit "missing store."
  Fix: make every build reproducible from source (the *results* are versioned; the *stores* are rebuilt).
- **Simple/anonymized data shows nothing — by design of the trap.** Anonymized Northwind (and any flat
  schema) produced a flat curve across all four conditions. That is not "the technology doesn't work"; it is
  the eval sitting below the threshold. [REVIEW: grounding tests] Fix: escalate complexity until the curve moves, and report where it
  did.
- **A null that's still a result: the reverse-relationship ablation.** Declaring + populating inverse
  relations gave a strong model + reverse-capable tooling *zero* discovery lift (already at ceiling). [REVIEW: grounding tests] Don't
  invest in ontology features the retrieval layer already covers; a well-designed eval tells you which
  features are redundant for your model, not just which help.

## 5. A recommended protocol (apply to your own ontology + model + data)

1. Stand up the retrieval-first grounding path against your store.
2. Author a discovery bank at **production complexity**: ~⅓ LOOKUP controls, ⅔ DISCOVERY, truth locked by
   reference query. Include a few **private-data-in-a-known-domain** questions.
3. Build the 4-point ontology axis (rich/sparse/misleading/none) with instance data held identical.
4. Run your production model, N≥5, one store per condition; grade truth-based + cross-model; human-anchor a
   sample.
5. Read it as: rich−none = *value of your ontology*; misleading−none = *cost of getting it wrong*;
   lookup vs discovery = *where it matters*. If the curve is flat, your test data is below the threshold —
   escalate complexity before concluding "no value." [REVIEW: grounding tests]
6. Repeat at 2–3 complexity points if you can, to locate your position on the scaling law.
