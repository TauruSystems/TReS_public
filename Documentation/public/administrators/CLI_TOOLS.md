# Command-Line Tools (QROS)

The `qros` command-line tool runs queries against a QROS store and
converts, inspects, and loads native QROS files (`.tqs` schema, `.tqd` data). It runs
independently of the cloud application — useful for scripting, CI/CD, and
batch conversion of existing data (updated 2026-08-28).

## qros — query and conversion tool

With no subcommand, `qros` runs a query. The file subcommands (`convert`,
`reclassify`, `conformance`, `load`) operate on files and stores.

```bash
# Run a query from a file against a store
qros --data ./mystore query.qros

# Pipe a query from stdin
echo 'FIND ?x WHERE { ?x :type :Product }' | qros --data ./mystore

# Interactive REPL
qros --data ./mystore --repl

# Convert a Turtle file to native QROS (auto-splits schema from data)
qros convert ontology.ttl
```

### Global options

| Flag | Short | Description |
|---|---|---|
| `--data <dir>` | `-d` | QROS storage directory. Required for queries and for `load` |
| `<query_file>` | | Query file to execute. Omit to read the query from stdin |
| `--repl` | | Start the interactive REPL |
| `--format <fmt>` | `-f` | Result format: `table` (default), `json`, `csv` |
| `--limit <n>` | `-l` | Maximum number of results to display |
| `--timing` | | Show query execution time |
| `--explain` | | Show the query plan |
| `--verbose` | `-v` | Verbose output |

In `--repl` mode the prompt accepts queries plus dot-commands: `.help`, `.tables`,
`.format <table|json|csv>`, `.timing on|off`, `.exit`. History is saved to
`~/.qros_history`.

### qros convert — Turtle to native QROS

Converts Turtle (`.ttl`) files, or a whole directory, to native QROS. Several files, or a directory with a per-file flag (`--format`, `--layer`, `--collapse`, `--hash-ids`, `--recursive`), run as a batch: each file is converted on its own into `--out-dir`. A bare directory with none of those flags runs dataset mode: the folder pools into one shared `.tqs` plus one `.tqd` per source (updated 2026-08-28). Without `--layer`, each subject is
classified as schema or data and the file is split into a `.tqs` and/or `.tqd`.
Vocabulary from other graph standards is normalized to QROS-native predicates, declared
prefixes are kept as clean identifiers, and blank nodes (e.g. union domain/range
declarations) are resolved to named statements. The tool reports what each pass did.

```bash
qros convert ontology.ttl                       # auto-split schema/data
qros convert ontology.ttl --format tqs           # force .tqs (schema) out
qros convert data.ttl --format tqd --out out/data # force .tqd (data) out
qros convert source.ttl --collapse --hash-ids
```

| Flag | Short | Description |
|---|---|---|
| `<input>` | | Input Turtle (`.ttl`) file. Required |
| `--out <path>` | `-o` | Output path stem. Defaults to the input path with a `.tqs`/`.tqd` extension |
| `--format <fmt>` | `-f` | Force a single output format: `tqs` (schema) or `tqd` (data). Writes the whole file in that format — point at an ontology, get `.tqs` out. Omit (with no `--layer`) to auto-split |
| `--layer <layer>` | `-l` | Force a single layer for every statement (`ontology`, `vocabulary`, `instance`, `inferred`, `alignment`) instead of auto-classifying. Also picks the format when `--format` is absent |
| `--out-dir <dir>` | | Write every file's outputs into this directory, deriving each name from its input — the batch counterpart of `--out` (updated 2026-08-28) |
| `--recursive` | | With a directory input, include `.ttl` files in subfolders; top level only without it (updated 2026-08-28) |
| `--collapse` | | Collapse reified n-ary instances into a single statement plus tags/scope |
| `--hash-ids` | | Rewrite opaque instance IRIs to QROS hash-style identifiers, preserving the original via `:externalId` |
| `--reports` | | Give report-style n-ary nodes a content-addressed identity instead of a minted UUID |

`--format tqs` and `--format tqd` are the direct "convert to this format" options and
work standalone. `--format` takes precedence over `--layer`'s derived format; with
neither, the converter auto-splits schema from data.

### qros reclassify — fix the schema/data layer of a file

Re-derives each subject's layer in a `.tqs`/`.tqd` and writes corrected files
alongside the original. The original is never overwritten; on a path collision the
outputs gain a `.fixed` infix.

```bash
qros reclassify mixed.tqd
qros reclassify mixed.tqd --out cleaned/mixed
```

### qros conformance — compare data usage against an ontology

Compares the vocabulary a data file *uses* against what an ontology *declares* —
reporting terms in both, declared-but-unused, and used-but-undeclared. Either side may
be Turtle/N-Triples or native QROS; terms are compared by local name.

```bash
qros conformance ontology.ttl data.tqd
```

### qros load — insert a native file into a store

Parses a native QROS file and inserts its statements into the store at `--data`,
making the data queryable. Appends to any existing store. Inference rules in a `.tqs`
(`@rules`) are reported but belong to the inference engine, not the store.

```bash
qros load data.tqd --data ./mystore
```

## Building

`qros` is built as part of the standard TReS build:

```bash
cargo build --release -p tres-qros-cli
```

The release binary is in `target/release/`. It runs standalone with no runtime
dependencies beyond the OS.
