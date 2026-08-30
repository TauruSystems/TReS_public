# Model Improvement Guide — making data groundable with closed-world V
> **[REVIEW 2026-08-28 — relies on the grounding tests.** Figures or conclusions in this document come from grounding evaluation runs that `design/CLAIMS_LEDGER.md` rates PROVISIONAL or DO-NOT-CITE (C1, C4–C6). Verify against the ledger before any external use. Passages are tagged inline.]

**Status:** methodology guide (2026-06-16). Companion to `QROS_NATIVE_FOUNDATION_DESIGN.md`
("The core (the bare math)" + the domain/range doctrine). Worked example: the Marvel dataset,
**9,689 → 144 falsified (−98.5%)** by ontology edits alone.

## The idea

Closed-world `V` (QROS Law 2) **falsifies** any assertion whose subject/object type falls outside
the predicate's declared domain/range. The **falsified count is the ontology-quality metric**:
improve the ontology and the count drops, and the previously-falsified vectors become **groundable**
(an LLM can answer over them; the substrate can assert them). This is the tangible bridge from
"better ontology" to "better question-answering." [REVIEW: grounding tests]

This guide is the repeatable workflow and the fix patterns. Each fix tightens the valid region to
match the data the owner actually has.

## Workflow

1. **Reconvert fresh.** `qros convert <dirs> --out-dir <d> --recursive`. Never measure a stale
   conversion — old output can carry dropped `subClassOf` etc. (see "pitfalls").
2. **Measure.** `ontology_quality <dir>` reports: the falsified count, a by-predicate breakdown,
   **validatable coverage** (fraction of data assertions under a domain/range-constrained
   predicate), the **undomained-predicate worklist**, and the **model-improvement worklist**
   (`predicate · domain|range · count · actual type(s) of the offenders`).
3. **Diagnose** each line of the worklist: is it a domain or range gap? what class does the data
   actually use vs what's declared?
4. **Fix** with the patterns below (edit the source ontology).
5. **Re-measure.** Confirm the count fell; repeat on the next predicate.

## Fix patterns

### A — Domain too narrow (the most common)
The predicate carries real data on a type outside its declared domain. **Widen the domain,
additively** (a union of the existing domain + the real type(s)).
*Marvel:* 15 distribution/identity properties (`directOSD`, `ISBN`, `shippingDate`, `FOCdate`, …)
had domain `Job`/`ComicIssue` but were used on `TradePaperback`s and other products → widened to
`Product`/`Job`. **9,689 → 1,717.**

### B — Range names the wrong (or undefined) class
The range references a class the data doesn't use — often an undefined name or a drifted one. Point
the range at the real class.
*Marvel:* `hasTalent`/`createdBy`/`writtenBy` range was `:Talent` (a class that isn't even defined)
while the data is `:MarvelTalent`; `hasEditor`/`hasAsstEditor` were `:MarvelPersonnel` vs data
`:MarvelPerson`; `variantProgram` was `xsd:string` vs an entity `:VariantProgram`. **1,717 → 144.**

### C — CRITICAL: widen with union, never *replace* (unless the old class is provably unused)
The worklist shows only the **violators** — the objects that *don't* match the current range. The
matching majority is **invisible**. If you *replace* a range with the violator's type, you flip the
violation onto the previously-passing majority.
*Marvel (a real mis-step):* `hasWorkType` range was `:CreatorWorkType` (560 objects passing) with 70
`:CreatorDiscipline` violators. Replacing the range with `:CreatorDiscipline` made it **560**
violations. The fix is the **union** `(:CreatorWorkType, :CreatorDiscipline)`. **Rule:** always
widen with a union (additive); only "replace" when the declared class is provably unused (e.g. the
undefined `:Talent` in pattern B, which nothing is typed as).

### D — Dual-class vocabulary noise
Vocab terms dual-typed `:Concept` + a domain class: the converter drops the redundant `:Concept`
and keeps the domain class (`drop_redundant_concept_type`), so classification/viz/grounding see the
meaningful type. *This is cleanliness; it does **not** change the `V` count* — `V` passes on
any-type-match, so an extra `:Concept` never caused a violation.

### E — Undomained predicate (coverage gap, not a falsification)
A predicate with no domain/range is invisible to `V` — its uses can't be validated. Declare a domain
(suggested from usage; **the human confirms** — never auto-accept, that's circular). *Marvel:*
biggest blind spot is `pubYear` (37,219 uses, undomained). Raising validatable coverage (89.5% on
Marvel) is the "encourage" lever; it is never required.

### F — Class-hierarchy gap (fix at the class, not the predicate)
The predicate's domain is the right *concept* but the data's type isn't linked to it. *Marvel:*
`UPC`/`issueNumber`/`EAN` are used on `Poster`/`GraphicComicBox`/`Pin`/`Omnibus` — these *are*
products, but lack `subClassOf Product`. The fix is at the class level (`:Poster :subClassOf
:Product`), which repairs every product-identifier predicate at once — not per-predicate widening.

## The discipline (doctrine)

- **Measure declared schema only** (`infer_schema:false`). Never validate against inferred domains —
  that validates data against constraints derived from the same data, and can never falsify.
- **Inference proposes, the human disposes.** Suggest domains from usage to make authoring easy; only
  a human-confirmed declaration becomes a validation contract.
- **Encourage, never require.** Tolerate undomained predicates and imperfect data; surface gaps;
  never block a load (tolerance is existential for the no-IT thesis).
- **Always additive, reconvert fresh, re-measure.** Widening can only reduce violations; replacing
  can increase them (pattern C).

## Marvel case study

| stage | falsified | pattern |
|---|---|---|
| baseline (fresh reconvert, declared schema) | 9,689 | — |
| domain widening (15 predicates → `Product`/`Job`) | 1,717 | A |
| range correction (9 predicates) | 144 | B, C |
| **remaining** | **144** | below |

**98.5%** of falsifications closed by ontology edits. Validatable coverage 89.5% (raise via E).

### Remaining 144 — flagged for owner review
- **`hasRoleType` (53):** range `:StoryRoleType` vs data `:TeamRole`/`:StoryRole`. Owner decides:
  union the range, or retype the role-type values. *(Left unchanged pending review.)*
- **`UPC` (33), `issueNumber` (30), `EAN` (7):** product-identifier domain gaps on
  `Poster`/`GraphicComicBox`/`Pin`/`TradePaperback`/`Omnibus`. Best fixed by pattern F
  (`:Poster`/etc. `:subClassOf :Product`), not per-predicate.
- **long tail (~17):** `writtenBy` domain on `Storyline`, `hasOrder`, `hasPrinter`, `hasRace`,
  `inNarrativeFamily`, etc. — one-off domain/range widenings.

## Pitfalls (verified the hard way)

- **Stale conversions lie.** An old `qros_native/` had dropped a `subClassOf` and failed to parse
  the vocabulary file; it inflated the count with artifacts. Always reconvert with the current converter.
- **Read the whole native block before blaming the converter.** A multi-line block (`:s :type :A ;
  :type :B`) can look single-typed if you only grep the first line. Two "converter bugs" were
  investigator error; the converter was faithful.
- **The violator list is a subset.** See pattern C — never infer the correct class from violators
  alone; widen, don't replace.
