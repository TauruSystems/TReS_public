# LLM Evaluation & Training Guide

How to test what your AI knows, act on what it gets wrong, and accumulate
training data you own. Companion to `PORTABILITY.md` (where the corpus goes
when you leave) and `MODEL_IMPROVEMENT_GUIDE.md` (the ontology-quality side
of the same coin).

## Grounding, not training

TReS grounds data; it does not train models. Every fact in TReS carries its
grounds — its source, its reliability, and the context it applies in — and
the model is given those grounds, on every question, at the moment it
answers. Nothing persists inside the model between questions.

What happens on each question:

- The model receives a briefing: the ontology (classes, properties, labels,
  and the annotations you wrote), the alias table, and the corrections your
  team has entered on this site.
- The model declares what it needs; the engine authors and runs the query
  over the closed world and returns the facts with their grounds.
- The answer is shown with the query and the retrieved facts, so you can
  see what it stood on.

What never happens:

- **No model's weights change.** TReS calls no fine-tuning or model-creation
  endpoint on any provider. The corrections, grades, and questions your
  team enters change the briefing, not the model, and every one of them is
  visible and removable.
- **Nothing crosses between sites.** Each TReS site has its own store,
  index, alias table, corrections, and question sets. There is no shared
  model, no pooled index, and no dataset built from one customer's data that
  another customer can reach.
- **Nothing is applied to a model on your behalf.** The training set, the
  corpus, the Ollama Modelfile, and the datasets generated from your
  ontology are files. TReS builds them; you download them and decide
  whether, and on which model, to use them. That is the only sense in which
  TReS "trains": it produces material you own.

What is computed from your data, and where it lives: the semantic index,
the embeddings behind it, and the alias table are built from your data, on
your site's own storage, and belong to your site. They are indexes over
your knowledge, not a model of it, and they are rebuilt from your data
whenever it changes.

Where your data goes: the LLM provider configured for your site receives
the briefing and the retrieved facts for each question — nothing more, and
nothing from any other surface (see `LLM_PROVIDER_GUIDE.md`). The site's
operational log records each query the model wrote and a short excerpt of
its result, so the operator can diagnose a bad answer; the log is kept for
thirty days.

This is why the same grounded data serves any model: changing providers is
a configuration change, and what the AI knows about your organization is in
your data and your corpus — both of which leave with you (`PORTABILITY.md`).

## The surfaces

- **LLM page — Quick question.** Ask one grounded question. The model never
  writes a query: it declares what it needs and the engine authors and runs
  the retrieval. The answer shows its tool round-trips; the trace is the
  ground truth of what was retrieved. Your questions and answers are
  visible to you; Owners, Admins and Auditors can review every user's.
- **LLM page — Question sets.** Your evaluation suite: each question carries
  the question text, a type (ontology or data), your authored expected
  answer, and optionally an **Unanswerable** mark — for questions whose
  correct grounded answer is a decline ("the data does not contain this").
  Question sets are editable after they have run history — each run snapshots the
  questions as asked, so past results stay faithful.
- **Run view.** Executes a question set against a model and mode, optionally scoped
  to one graph. Results stream in; the summary separates the model's run
  (answered, errored) from your review (graded, pass/fail/partial, score).
- **Recommendations.** Computed from the run's traces, your grades, and the
  run history; updates as you grade. The panel leads with
  **Since last run** (which questions were fixed, which broke, which still
  fail versus your previous comparable run) and a **work list** that names
  the exact terms retrieval could not satisfy, each with a diagnosis from
  the readiness analyzer — "declared but carries no data", "used in the
  data but never declared", or "the vocabulary does not cover it".

## The evaluation loop

1. **Author** questions with expected answers. Write the expected answer as
   the fact a correct grounded response must contain, not as ideal prose.
2. **Run** the question set. Grounded answers vary run to run — conclusions come from
   rates over repeated runs, not from a single result.
3. **Grade** each answer: pass, partial, or fail, with reviewer notes.
   Grades are the signal the system computes from. The **grading order
   hint** pre-sorts ungraded results by checking whether each answer
   contains your authored expected fact — start with the ones flagged
   "start here"; the hint is advisory and never becomes a grade. Notes are
   for people: they are stored with the result and shown on review, but no
   engine consumes them today.
4. **Read the recommendations.** They name the lever your failures point
   at — and, where the traces allow it, the exact artifact to fix:
   - **Data** — the work list groups failures by the term retrieval could
     not satisfy and says why the knowledge came up empty (a declared
     predicate carrying no statements, a data-minted predicate with no
     domain or range, a term the vocabulary does not cover). Fix the named
     knowledge, not the model.
   - **Model** — answers failed even though retrieval returned data, the
     model looped or malformed its retrievals, or it answered a question
     you marked unanswerable instead of declining (the hallucination flag).
   - **Methodology** — too few graded results or too few runs to conclude
     anything; grade more, run again.
   - **Configuration** — budget caps cut answers off mid-retrieval.
5. **Pull the lever, re-run, re-grade.** A question that failed for missing
   knowledge passes once the knowledge is loaded. That flip — fail, fix the
   data, pass — is the loop working as designed, and **Since last run**
   shows it to you: fixed, broke, and still-failing lists, computed
   per-question against your previous run of the same question set, model, mode,
   and graph. Runs over different graphs are different experiments and are
   never compared.

## What a failed answer is (and is not)

A grounded model that answers "that is not in the knowledge" when the
knowledge is absent is behaving correctly — the closed world makes "we don't
know" a detectable state instead of an invitation to guess.

Test that behavior deliberately: mark such questions **Unanswerable** in the
bank. When a run answers one confidently instead of declining, the
recommendations panel raises the hallucination flag with the offending
questions listed — the most dangerous failure mode, caught before you have
graded anything.

**Failed answers are never training data.** A fail's entire role is
diagnostic: it drives failure clustering and points at the lever. Only
answers a human graded **pass** enter the training corpus. Nothing the model
got wrong — and nothing you haven't approved — can teach the next model.

## The training corpus — what you own

Select a question set on the LLM page and use **Export training set** — a
fine-tune-ready `.jsonl` file, one chat-format pair per question: user = the
question, assistant = the latest human-graded-pass answer, falling back to
your authored expected answer; a question with neither is skipped because it
has nothing to teach. **Export corpus (JSON)** downloads the full portable
object: every question with its type, expected answer, verified answer, and
the query that produced it.

The same exports are available to integrations at
`GET /api/v1/eval/banks/{id}/corpus` (`?format=jsonl` for the training set).
See `PORTABILITY.md`: the corpus is layer 3 of what leaves with you.

## What the system does not do

- **No automatic retraining.** The corpus accumulates verified,
  human-approved pairs automatically; applying them to a model (fine-tuning,
  few-shot selection) is a deliberate step you take, on any model you
  choose. The on-board model does not silently learn from your grades.
- **Notes are not machine-read.** Reviewer notes inform the next human
  reviewer; today they do not alter recommendations or the corpus.
- **Your questions are yours to see.** Every user sees their own questions
  and answers; Owners, Admins and Auditors can review every user's. The
  audit log, readable by those same three roles, records the text of each
  question and each correction — never the answers.

## Training data from the knowledge itself

The Training data tab generates question/answer material directly from your
graphs — a second corpus source, drawn from what the knowledge contains
rather than from graded runs. Generated material is a draft for your review,
not automatic gold: the same rule applies — a human approves what teaches.
