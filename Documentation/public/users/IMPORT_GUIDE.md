# Loading Data into a QROS Instance

How data gets into TReS, and what happens to it on the way in. For a
step-by-step walkthrough of the recommended approach, see
[Importing Your Data](IMPORTING_YOUR_DATA.md). For the native file formats and
the command-line converter, see the [CLI Tools](administrators/CLI_TOOLS.md) guide.

## The recommended approach: give TReS the folder

Most business data does not start as a graph. It starts as a folder — a sales
report here, a production schedule there, an ontology file from an earlier
project, a PDF of reference material. The single most important thing to know
about loading data into TReS is this: **do not import those files one at a
time. Point TReS at the folder.** The folder can hold any mix of importable
formats — spreadsheets, graph files (Turtle, N-Triples), native QROS
(`.tqs`/`.tqd`), and documents — and each file is routed to its own pathway:
sheets get mapping proposals, graph and native files load as graph data, and
documents are catalogued for extraction.

The reason is that the decisions that make your data connect are decisions
about the files *together*, not any file alone:

- **Which columns join your sheets.** An item number that appears in four
  workbooks is what makes a cost row from one file and a sales row from
  another meet on the same product. TReS finds those shared identifiers by
  comparing the actual values across every sheet.
- **Which sheets to leave out.** Real folders hold summary tabs, report
  covers, duplicated exports, and superseded versions. TReS proposes what to
  skip — and says why, so you can override it.
- **Which rows are repeated observations.** Monthly sales, per-match
  statistics, per-release records: rows about the same thing that differ by
  period. These need a context tag to stay distinct — imported naively, they
  silently merge. TReS detects the shape and proposes the tag.
- **Which identifiers never connect.** If your item-number world and your
  ISBN world share no column, no import can join them. TReS names the missing
  bridge so you can supply it — usually one export from the system that knows
  both.

From **Admin → Load Data → Folder** (QROS instances), choose your folder —
its own contents only (subfolders are left alone), no zipping — and TReS
stages every file, analyzes the corpus, and presents one reviewable plan:
per-sheet mappings, the identity plan, skip suggestions, content analysis on
graph and native files (ontology, vocabulary, or instance data — each with a
proposed named graph), duplicate and superseded-version flags, and anything
only you can resolve. Every row is selectable, so you can skip any file the
plan got wrong. Analysis is free of side effects: nothing is imported until
you act on a row — **Load** writes a graph or native file into its target
graph (the proposed graph is a default you can change), and **Map** hands a
sheet into the Spreadsheet wizard for mapping and commit. **(One-step commit
of a whole plan is coming.)**

### Single sheets still work — and they grow your schema

The one-sheet path from **Admin → Load Data → Spreadsheet** remains the
on-ramp for quick loads: a guided wizard maps each row to a subject entity
(minting a fresh one or reconciling against existing entities), assigns its
data and relationship properties, and can attach a tag to every fact a row
asserts. The result commits as native QROS.

A mapped import does more than move rows: the mapping itself declares kinds
of things and kinds of facts, so each deliberate import **fills in a shallow
ontology as you go**. Starting small is a legitimate strategy — one
well-mapped sheet gives the next sheet something to connect to, and the folder
analysis gets better as your schema grows.

Before committing, the preview includes a **merge check**: if any rows would
produce identical facts and silently combine on load, the wizard shows exactly
which ones and suggests the context tag that keeps them distinct.

## Where your files can come from

- **Your computer** — the folder picker and the file pickers above.
- **Git** — pull graph interchange or native QROS files from a configured Git source.
  Private sources authenticate with an access token managed by the operator.
  Useful for keeping the instance's data under version control.
- **Cloud storage** (Google Drive, Dropbox, OneDrive, S3) — connect a drive
  and analyze a folder there. **(Planned — not yet on Preview.)**

## More sources

- **Documents** (PDF, Word) — pull structured knowledge out of unstructured
  documents. **(Planned — not yet on Preview.)**
- **JSON / application exports** — bring in a JSON-flavored API response or an
  export from a non-graph application. **(Planned — not yet on Preview.)**
- **Existing databases** (SQL and other non-graph stores) — connect to a
  database you already run and import from it. **(Planned — not yet on
  Preview.)**

## Advanced: graph files

If you already have graph data, **graph interchange files** (.ttl, .nt) are
converted to native QROS on load: the converter classifies each subject as
schema or data, normalizes vocabulary from other graph standards to QROS-native
predicates, keeps declared prefixes as clean identifiers, and resolves blank
nodes (for example, union domain/range declarations) to named statements rather
than skolemizing them. **Native QROS files** load directly: `.tqs` (schema —
classes, properties, controlled vocabularies) and `.tqd` (data — instances,
inferred facts, alignments). This path suits users who already work with graph
interchange formats; most data does not start here.

## What happens after a load

The intelligence layer (the VDB) rebuilds in the background, so search,
visualization, and pathfinding reflect the new data. The application stays
responsive during the rebuild; if search or visualization looks incomplete
right after a load, retry shortly.

## Schema and data stay separate

QROS keeps schema (`.tqs`) and data (`.tqd`) in separate files by design. This
lets a single body of data be viewed through different domain schemas —
toggling an ontological "lens" on or off — with data always falling back to
the QROS base classes when no domain schema is applied. When you load a graph
file that mixes schema and instances, the converter splits them for you.

## Authoring vs. loading

Loading brings in existing data in bulk. For ongoing authoring — adding
entities and their contextual tags, editing the schema, managing vocabulary,
defining rules — use the editing surfaces described in the
[Operations Guide](administrators/OPERATIONS_CLOUD.md). The Query page itself is read-only on
QROS instances.

## Where to get help

- **Admin → Usage & audit** records administrative actions including loads.
- **info@taurusystems.com** — for support.
