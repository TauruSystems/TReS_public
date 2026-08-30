# Integration API

The Integration API is how another system reads knowledge out of your TReS
instance: a BI tool refreshing a dashboard, an ETL job, a reporting script, or
an AI agent that needs governed facts instead of a spreadsheet.

It is deliberately small. Your instance also serves a much larger set of
routes, but those are the private backend of the TReS web application. They
change whenever the application changes. **Only the endpoints named in this
document are supported for external use.** Everything else is internal, and
integrating against it will break without notice.

## Who this is for

Someone writing a client — a dashboard connector, a scheduled job, an agent
tool. You need a service-account credential from an administrator of your
instance, and you need to know your instance's base URL.

## Getting a credential

Machine callers authenticate as a **service account**, not as a person. A
service account has its own credential, its own request budget, its own
entries in the audit log — and a **role**, chosen when it is created. The
default, `Integration`, is read-only: queries, taxonomy reads and tabular
export. An administrator can instead give an account the same role a person
would hold — `User`, `PowerUser`, `Admin` or `Owner` — so an AI assistant or a
scheduled job can load a spreadsheet, maintain data or repair dashboard cards
on someone's behalf, through the same endpoints and the same permission
checks that person would meet. An account can never hold a higher role than
the administrator who created it.

An administrator creates one in the application under **Admin > Service
accounts**: give it a name, a purpose and a role, and the credential is
displayed **once**, at creation. It is stored only as a hash, so it cannot be
shown again — if it is lost, revoke it and create another.

Credentials start with `tres_sa_`. Treat one like a password: keep it in your
secret store, never in source control, and revoke it the moment it is no
longer needed. Revocation takes effect on the next request.

How many service accounts your instance may hold depends on your plan. The
Admin screen shows the count in use against the number included.

## Authentication

Send the credential as a bearer token on every request:

```
Authorization: Bearer tres_sa_your_credential_here
```

Nothing else is needed. Service accounts do not use browser sessions, and no
session header applies to them.

| Response | Meaning |
|---|---|
| `401` | The credential is unknown, revoked, or malformed. |
| `429` | Too many requests — see [Request budget](#request-budget). |
| `403` | The credential is valid, but your plan does not include machine access. |

## Endpoint: read a table

```
POST /api/v1/data
```

Runs one read query and returns the answer as a table.

### Request

```json
{
  "query": "SELECT ?name ?status WHERE { ... }",
  "graphs": [],
  "format": "json"
}
```

| Field | Required | Meaning |
|---|---|---|
| `query` | yes | The read query. See [Query language](#query-language). |
| `graphs` | no | Names of the graphs to read. An empty list or an omitted field reads the whole knowledge base. |
| `format` | no | `json` (default) or `csv`. |

You may also request CSV with an `Accept: text/csv` header instead of the
`format` field. An explicit `format` in the body wins over the header.

### Query language

The query language follows your instance's engine:

- **QROS instances** take QQL. See the QQL guide for the language itself.
- **RDF instances** take SPARQL, restricted to `SELECT`, `CONSTRUCT`, `ASK`,
  and `DESCRIBE`.

Anything that would modify data is rejected before it runs. This is not a
setting; a read credential cannot reach a write path.

### Response: JSON

The default response is a table in the standard query-results envelope:

```json
{
  "head": { "vars": ["name", "status"] },
  "results": {
    "bindings": [
      { "name": { "type": "literal", "value": "North Line" },
        "status": { "type": "literal", "value": "Active" } }
    ]
  }
}
```

- `head.vars` lists the columns, in order.
- `results.bindings` is one object per row. A column with no value for that
  row is **absent from the object** rather than present-and-empty — read
  defensively.
- Each cell carries the value and its term type, so a client can tell an
  identifier from a piece of text without guessing.

Content type: `application/sparql-results+json`.

Some answers carry an additional top-level `support` object. It reports how
many statements stand behind the answer, and the point in time the answer
describes. Where it appears it is part of the answer, not decoration; where it
does not, nothing is missing.

### Response: CSV

CSV is derived from exactly the same result, so the two formats can never
disagree. The header row is the column names; one line per row; a column with
no value is an empty cell.

Dialect: RFC 4180, UTF-8, no byte-order mark, `CRLF` line endings, values
quoted only where a comma, quote, or newline requires it. Values are passed
through as the engine returned them — nothing is reformatted to look tidier.

CSV can only express a table. A query whose answer is not tabular returns an
error rather than an invented table.

### Response headers

| Header | Meaning |
|---|---|
| `X-TReS-Dataset-Fingerprint` | Identifies the dataset the last completed index build consumed. Advisory — see below. |
| `X-Tres-Query-Warnings` | Present when the query ran but something about it is worth knowing. Pipe-separated. Not an error. |

**On the fingerprint.** When it changes, the underlying data has changed and
a cached dashboard should refresh. When it does not change, the data has
*probably* not changed — but queries read live data, so during the window
between a data load and its index rebuild the fingerprint still names the
older dataset. It is therefore a refresh hint, not a cache validator: it is
deliberately not an `ETag`, there is no `304` response, and a client must
never use it to decide that a re-read can be skipped for correctness.

### Errors

Errors are JSON: `{"error": "...", "detail": "..."}`. `detail` may be absent.

| Status | Meaning | What to do |
|---|---|---|
| `400` | The query did not parse, or failed validation. | Fix the query. `detail` says what was wrong. |
| `403` | The query was a write, was refused by the safety validator, or your plan does not cover this. | Send a read query, or talk to your administrator. |
| `422` | The query reached a resource limit the instance sets. The body also carries `cause`, `limit` and `spent` (below). | Make the query more selective; `cause` says which way. |
| `429` | Over the request budget. | Slow down; retry after the window. |
| `502` | The knowledge base could not be reached. | Retry with backoff. |
| `504` | The query took longer than the engine allows. | Narrow the query. |
| `507` | The answer was larger than the response cap. | Add a limit, or select fewer columns. |
| `500` | Something failed that should not have. | Retry once; report it if it persists. |

### What a limit means

A `422` names the limit it reached, so you can tell three different things
apart: a query cut off while it was working, a query whose answer would not
fit, and a query refused before it started.

```json
{"error": "limit_exceeded", "cause": "work", "limit": 50000000, "spent": 50000064,
 "detail": "query exceeded the work budget of 50000000 scanned statements/rows (spent 50000064) — bind more terms or narrow the graph scope"}
```

| `cause` | What was reached | `limit` / `spent` are |
|---|---|---|
| `work` | The work budget: statements scanned plus rows produced, while the query was running. | work units |
| `rows` | The cap on how many rows the query may hold at once. | rows |
| `unbounded_pattern` | A pattern with no bound term would read the whole knowledge base; refused rather than truncated. | statements |
| `path_bound` | A property path walked further than the instance allows; refused rather than returned partial. | hops |
| `store_size` | A whole-database operation refused because the database is larger than it may run against. | statements |

A query that completes reports what it cost beside what it was allowed, so an
expensive answer is visible as such and not lost inside a success:

```json
{"head": {...}, "results": {...}, "budget": {"work_spent": 18342, "work_limit": 50000000}}
```

`work_limit` is `null` when the instance runs without a work budget. The
limits themselves are per-site quantities set on your plan; your
administrator can see them.

## Request budget

Each service account has its own per-minute request budget, set per instance.
The default is 120 requests per minute. Over budget, requests are refused with
`429` until the window rolls forward; other service accounts are unaffected.

The budget is sized for dashboard polling and agent tool calls, not for bulk
export. If you need to move a large amount of data, ask for fewer, larger
queries rather than more frequent ones.

## Audit

Every call is recorded in the instance audit log with the service account as
the actor — shown by its name, marked as a service account — the graphs
named, and the query. Administrators see machine activity next to human
activity in one trail, and a write made by an account is attributed to it
exactly as a person's would be.

## Endpoint: connect an AI agent (MCP)

```
POST /api/v1/mcp
```

This is a [Model Context Protocol](https://modelcontextprotocol.io) server. Point
an MCP-capable client or agent framework at that URL with a service-account
credential and your model can read this knowledge base through a set of tools —
without you writing any retrieval code.

The tools are the same ones the TReS assistant uses on itself. Your model
inherits the same discipline and the same refusals: when the data does not
answer something, the tool says so rather than filling the gap.

### Connecting

Transport is MCP streamable HTTP: one HTTP POST per JSON-RPC message, to the URL
above. Authentication is the same bearer credential as everything else here:

```
Authorization: Bearer tres_sa_your_credential_here
Content-Type: application/json
Accept: application/json, text/event-stream
```

Protocol revisions: this server speaks `2026-07-28` and answers the older
`initialize`-handshake revisions (`2025-11-25`, `2025-06-18`, `2025-03-26`) as
well, so current and older clients both connect. Under `2026-07-28` every
request carries its own protocol version and client capabilities, and the
`MCP-Protocol-Version`, `Mcp-Method`, and `Mcp-Name` headers must agree with the
request body — a disagreement is refused with `400` and JSON-RPC error `-32020`.
There are no protocol sessions: `GET` and `DELETE` on this endpoint return `405`,
and session headers are ignored.

Methods served: `server/discover`, `tools/list`, `tools/call`, `ping`, and
`initialize` for older clients. Anything else returns `-32601`.

### The tools

| Tool | What it does |
|---|---|
| `lookup_term` | Turn a word into this ontology's exact term, with its contract: a predicate's domain and range, the tags its statements require or allow, an entity's type. The starting point for everything else. |
| `search` | Find entities, classes, or predicates by label. Returns identifiers, labels, kinds, and classes. |
| `get_entity` | One entity's full record: labels, classes, and every outbound fact with its tags and effective validity. |
| `get_facts` | One entity's facts with their contextual tags, as text. |
| `list_instances` | Enumerate the members of a class, following the class hierarchy. |
| `connections` | The relationships that actually exist from a class, so a model builds a real path instead of guessing one. |
| `traverse` | Follow a declared path across several hops in one call. |
| `find_path` | The valid predicate path between two classes. |
| `verify` | Check a specific proposed fact: true, false, or not derivable from this data. |
| `query` | Run one read query directly. Same language, validation, and limits as `POST /api/v1/data`. |
| `ask` | Ask a plain-language question and get a grounded answer plus the retrieval steps behind it. Uses this instance's AI configuration and counts against its AI limits. |

Every tool is read-only. **No tool on this surface writes, in this version or a
later one** — knowledge enters TReS through the governed import pathway, with a
person accountable for it. That is a design position, not a missing feature.

Tools that read the whole knowledge base do so by default; `get_entity` and
`ask` accept an optional `graph` to narrow the scope.

### Reading a result

Tool results follow the protocol: a `content` array with a text block, and, where
the tool returns structured data (`search`, `get_entity`, `ask`), a
`structuredContent` value carrying the same answer as JSON.

Two kinds of failure, and the difference matters:

- **A tool result with `isError: true`** is something your model can act on — an
  argument it should fix, an index still building, an entity this knowledge base
  holds nothing about. Pass it back to the model.
- **A JSON-RPC `error`** is a problem with the request itself: an unknown tool
  (`-32602`), an unknown method (`-32601`), a header that disagrees with the body
  (`-32020`), an unsupported protocol version (`-32022`, whose `data.supported`
  lists what this server does speak).

"The knowledge base does not contain this" is a normal, successful result. It is
the answer, not a failure.

### Limits

MCP calls draw on the same per-service-account request budget as everything else
(120 requests per minute by default). `ask` additionally spends AI capacity: it
is subject to this instance's AI entitlement and its ceiling on concurrent AI
operations, and it may be refused when the instance is saturated or when the
plan requires machine AI to run on your own model provider.

## What is supported, and what changes

Supported, and covered by the compatibility promise below:

- `POST /api/v1/data`, its request fields, its two response formats, its
  response headers, and its error statuses.
- `POST /api/v1/mcp`, the protocol revisions listed above, the tool names and
  their input schemas, and the shape of their results.
- Bearer authentication with a service-account credential.

Not supported for external use: every other route on the instance. They exist
to serve the TReS application and will change with it.

**Compatibility promise.** The version lives in the path. Within `/api/v1`:

- Fields may be **added** to a request as optional, and to a response.
- A field that has shipped will not be removed, renamed, or re-typed, and the
  meaning of a value will not change.
- Error statuses will not be repurposed.
- A change that would break any of the above ships as a new version path, and
  the existing one keeps working.

Write your client so that unknown fields are ignored rather than treated as
errors, and additive changes will never reach you.

## Not in this version

Named here so that nobody builds around a gap and calls it a contract:

- **No pagination.** One request returns one complete answer, bounded by the
  response cap. Page by putting a limit and an offset in the query itself.
- **No conditional requests.** No `ETag`, no `If-None-Match`, no `304`.
- **No push.** No webhooks and no subscriptions; polling with the fingerprint
  hint is the intended pattern.
- **No writes**, on either endpoint. Knowledge enters TReS through the governed
  import pathway, with a person accountable for it. That is a design position,
  not a missing feature.
- **No MCP resources or prompt templates yet.** The server declares the tools
  capability only; `resources/list` and `prompts/list` return `-32601`.
- **No OAuth on the MCP endpoint.** Authentication is the bearer credential,
  which is what an unattended agent needs. The protocol's OAuth flow suits an
  interactive connector onboarding a person, and will be added when a client
  that requires it is in front of us.
