# Importing Your Data — a walkthrough

The [Import Guide](IMPORT_GUIDE.md) covers the loading paths; this guide walks
the recommended one — giving TReS your folder — end to end, and explains the
handful of decisions the system will ask you to make. None of them require
graph expertise. Each is a question about your own data, and you are the
person who knows the answer.

## Before you start: what to put in the folder

Use the folder your team already keeps — the working files themselves, in any
mix of formats: spreadsheets, graph files (Turtle, N-Triples), native QROS
(`.tqs`/`.tqd`), and documents (PDF, text). Each file is routed to its own
pathway — sheets get mapping proposals, graph and native files load as graph
data, documents are catalogued for extraction. You do not need to clean
anything first. A few conventions the analyzer understands:

- **Versions resolve themselves.** `sales_v2.xlsx` next to `sales_v3.xlsx`:
  only the latest is used, and the plan says so.
- **Only the folder's own contents are read.** Subfolders are left alone —
  park superseded files or out-of-scope material in one (an `archive/`
  subfolder is the natural spot).
- **Duplicates are caught.** Two staged files with identical content, or a
  `sales_v2` sitting next to a `sales_v3`, get a skip proposal — with the
  reason, so you can override it.
- **Graph and native files are read for what they are.** An ontology, a
  vocabulary, and instance data each get a proposed named graph
  automatically; you can change the target before loading.
- **Unsupported formats are ignored** (and listed, so nothing disappears
  silently).

One preparation IS worth the time: if your organization keeps a controlled
vocabulary (product masters, people, teams, territories), load it first.
Sheet values then reconcile against managed entities instead of creating
lookalikes — see "Matching, not minting" below.

## Step 1 — Analyze

**Admin → Load Data → Folder → Choose a folder.** The browser reads the
folder's top-level contents (no zipping, no subfolders) and TReS stages every
file, then builds one plan. Nothing is written to your graph at this stage;
analysis is always safe to run.

## Step 2 — Read the plan

The plan is the folder as TReS understands it. Five sections matter most:

**The identity plan.** Which columns join your sheets, with the evidence
(how many values they share, how completely one covers the other). This is
the backbone of everything: facts from different files become facts about the
same things through these columns. If a join you know exists is not listed,
the columns' values may not actually match (formatting, prefixes) — that is
worth knowing before import, not after.

**Disconnected identifier worlds.** The plan's most valuable warning. If your
sales sheets are keyed by item number and your warehouse sheets by ISBN, and
no staged column carries both, the two worlds cannot be connected by any
import — the link simply is not in the data. The plan names the missing
bridge. The fix is usually one export from the system that knows both
identifiers, dropped into the folder.

**Skip suggestions.** Duplicated tabs, empty sheets, report covers with no
usable header row, identical files staged twice, superseded versions. Every
skip carries its reason, and every row has a checkbox — the analyzer
proposes; you decide.

**Graph and native files.** Each is inspected for what it holds — an
ontology, a vocabulary, or instance data — and gets the matching named graph
proposed as its load target. The proposal is a default: change the target
graph on the row before loading if you keep things differently.

**Repeated observations.** When rows repeat per subject (monthly figures,
per-event statistics), the plan flags them and proposes the column that tells
the repeats apart — the month, the match, the release. Accepting that
proposal maps the column to a context tag, and every observation stays a
distinct fact. Declining it means same-value rows merge into one fact on
load. The preview's merge check shows exactly what would merge before
anything commits, so this decision is never made blind.

## Step 3 — The judgment moments

One rule sits under all of these: **there are no loose facts in TReS. Every
row of data belongs to a category (a class) that you name.** A column that
creates things must say what kind of thing — that is what makes your data
answerable later, and it is why the import will ask rather than guess. It is
also why nothing is ever attached to an existing entity automatically unless
the name matches exactly AND the category matches the one you declared.

These are the questions only you can answer. There are five, and the plan
raises each one only when your data actually poses it.

**1. Identity — which column is "the thing."** Usually obvious (the item
number, the account code), and usually confirmed rather than chosen: the
identity plan will have found it by value overlap.

**2. Matching, not minting.** When a sheet column names things that already
exist — people, products, vocabulary terms — the import should attach facts
to the EXISTING entity, not create a new one spelled the same way. This is
the difference between "Kate Bishop" being one entity with all her facts, and
three lookalikes each holding a fragment. The wizard's reconcile step matches
by label and shows you every match it wants to make; review it rather than
skipping it, especially for names that come in variants.

**3. Blank by design.** Many sheets carry a maximum set of columns that not
every row is meant to fill — an ISBN column that only collected editions
have. The analyzer detects when blanks track a category and says so
("filled only where Item Type is Collection"). Confirming it stops those
blanks from being treated as data-quality problems, and records the
difference between "not applicable" and "unknown" — which matters when you
later ask questions of the data.

**4. Context tags for observations.** The repeated-observations proposal from
Step 2. The rule of thumb: if a row is a *reading* of something (this month's
units, this match's goals), map what distinguishes the readings to a context
tag. If a row simply *describes* something (its name, its type), no tag is
needed and re-imports harmlessly update in place.

**5. The bridge you may need to supply.** From the disconnected-worlds
warning. No system can conjure a mapping your data does not contain;
supplying the one bridge file is often the single highest-value step of an
entire import.

## Step 4 — Import and verify

Work the plan row by row. **Load** on a graph or native file writes it into
its target graph immediately. **Map** on a sheet hands it into the
Spreadsheet wizard, where every preview shows the facts it will write, the
rows it will skip and why, and the merge check before anything commits.
**(One-step commit of a whole plan is coming.)** After loading, the
intelligence layer rebuilds in the background and the new data appears in
search, query, and visualization.

Two habits pay off:

- **Ask a question you know the answer to.** The fastest end-to-end check of
  an import is a question whose answer you can verify against the source
  sheet — a total, a count, a "who has the most."
- **Re-run the analysis after adding files.** The plan improves as the corpus
  grows: new bridges connect old worlds, and the schema your earlier imports
  built makes the next proposals sharper.

## Growing a schema as you go

You do not need a finished ontology to start. Each deliberately mapped import
declares kinds of things and kinds of facts, and those declarations
accumulate into a working schema — a shallow ontology that deepens as you
import more. Teams that invest in that schema (naming the kinds well,
loading a controlled vocabulary, recording which columns apply to which
kinds) get direct returns at question time: the assistant answers with the
engine's numbers instead of guesses, and smaller, cheaper models become
viable. The [Ontology Guide](ONTOLOGY_GUIDE.md) and
[Vocabulary Guide](VOCABULARY_GUIDE.md) cover that investment.

## Where to get help

**info@taurusystems.com** — send the plan's warnings along; they are designed
to be readable by support and by you alike.
