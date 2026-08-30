# Portability — TReS is an on-ramp, not a cage

TReS is built for a team or a process to turn its own data into knowledge it can trust,
quickly and at low risk. Part of "low risk" is this promise: **everything you build in TReS
is yours and portable.** When you outgrow TReS — bigger LLM, your own vector database, your
own servers — you take it all with you. Nothing important is locked inside our platform or
inside a proprietary model.

## What's portable — all four layers

1. **Your data.** Every fact, with its tags (source, validity, context, confidence), exports
   as native QROS (`.tqs` / `.tqd`) or as standard **graph text** (Turtle / N-Triples) that any
   graph database can load. No proprietary container.
2. **Your ontology.** Classes, properties, vocabulary, scope, and the tag-contracts — the whole
   model — exports the same way. It's text, diffable, standard.
   **Rules export separately.** They live in their own store, not in a graph, so a graph
   download does not contain them: use **Database Management → Download rules** for a `.tqs`
   holding every rule (named rule sets and rules authored in the app alike). It re-loads
   through the ordinary Load page. A complete backup is the graph files **plus** this one.
3. **Your AI's competence — the corpus.** The LLM Testbed's question sets plus the answers you've
   graded are exportable as a **training / evaluation corpus** (`/api/v1/eval/question sets/{id}/corpus`).
   This is the record of what the AI answered correctly over your knowledge — the material for
   tuning *any* model, if you choose to. It is yours; it leaves with you. Only human-graded **pass** answers enter it — see
   `LLM_EVALUATION_TRAINING_GUIDE.md` for the full loop and the corpus formats.
4. **The grounding approach itself.** TReS makes a model "QROS-ready" through an **open,
   model-agnostic** method — a grounding preamble, an ontology lookup tool, and the tag-contract
   the model reads — *not* through secret model weights. The same approach works on whatever LLM
   you bring. There's no proprietary model you'd be stranded without.

## Why there's no lock-in

TReS uses an LLM as a *client* — it calls a model over a standard interface. The AI's
competence does **not** live in tuned weights you can't take. It lives in:

- your **data and ontology** (portable, standard formats), and
- your **corpus** (portable data that can tune any model), consumed through
- an **open grounding method** (works on any capable LLM).

A small model tuned for QROS, if used, is only a cost optimization for the small-team phase —
and because it's reproducible from the corpus, even that isn't a trap.

## QROS itself travels — the standard, the data, the engine

The promise extends to QROS, not just the data inside it:

- **The standard is documented and open.** The QROS data model and the `.tqs`/`.tqd` format —
  the tag, scope, and rules semantics — are written down (the QROS Technical Paper and the
  [Ontology Guide](ONTOLOGY_GUIDE.md)). It's readable and implementable, not a black box.
- **The data keeps full QROS fidelity.** Native `.tqs`/`.tqd` export preserves every tag,
  scope, validity bound, and rule — not just a flattened rendering. Standard graph-text export
  is also there for any graph database (where contextual richness degrades to
  reification/named-graphs — your data, in universal form).
- **The engine is self-contained.** QROS runs as an embedded engine — in-process storage, no
  external services — so it isn't tied to our cloud to keep working.

So moving on has two honest exits, neither a downgrade trap:

1. **Keep QROS** — take your native `.tqs`/`.tqd` and keep running it (QQL, tags, scope, rules
   all intact).
2. **Go standard graph text** — export into any graph database (universal; context degrades,
   but the data is whole).

## The graduation path

When the time is right to move to an enterprise stack:

1. **Export** your ontology + data (standard graph text or native) and your corpus.
2. **Point your own LLM / vector DB / servers** at them.
3. You are **QROS-ready immediately** via the same open grounding method — no re-learning from
   scratch. Optionally **fine-tune your model** with the corpus for efficiency.

You leave with your data, your model, your ontology, *and* your AI's competence. TReS got you
to good, trustworthy, AI-ready data on a small footprint; graduating to scale doesn't cost you
any of it.
