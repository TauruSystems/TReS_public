# TReS Cloud — Operations Guide

TReS lets the people who own the data turn it into knowledge they — and their AI — can trust, without waiting on IT or a data team. Every fact carries where it came from, how reliable it is, and where it applies, so the answers hold up. Powered by QROS, a new kind of engine. This guide covers day-to-day operation of a TReS Cloud instance for users and administrators.

## Signing in and sessions

- Access your instance at its URL (e.g. `https://tres260901a.taurusystems.com/`) — the address is assigned when the site is created and does not change. Authentication is delegated to Amazon Cognito.
- New users receive a temporary password by email and set a permanent one on first sign-in.
- Sessions enforce an **idle timeout** and a **single active session** per account — signing in elsewhere ends the first session.

## Roles and access

| Role | What it can do |
|---|---|
| **Admin** | All surfaces, plus user management, page visibility, audit, and data loading |
| **Power User** | Read and write: query, author entities, edit vocabulary and schema, define rules |
| **User** | Read-only across the enabled surfaces |
| **Auditor** | Read-only, plus the audit log and every user's questions to the assistant (Enterprise-tier role) |

Administrators control which surfaces are visible to each role; a few core surfaces are always available.

## Working with data

All operations route through the instance's QROS substrate. Most surfaces honor the **graph selector**.

- **Query** — ask questions in **QQL**, the QROS query language. QQL queries a statement's tags (confidence, validity, context) directly, supports point-in-time `AS OF` queries, and returns a **support set** (the basis an answer was computed over) for scope- or time-aware queries. See the [QQL Guide](../QQL_GUIDE.md). The Query page is **read-only** on QROS instances — author data through the editing surfaces below.
- **Browse** — explore classes and instances without writing a query.
- **Visualization** — search for an entity, place it, expand its relationships, and find the shortest path between two nodes.
- **Entity Info** — inspect any entity's labels, types, properties, inbound references, and the contextual tags on its statements (confidence, source, validity, context), with click-through navigation.
- **Entity authoring** — add and edit entities and their statements, including contextual tags. See the [Ontology Guide](../ONTOLOGY_GUIDE.md).
- **Vocabulary** — manage concept schemes and concepts: create, edit, move, and delete.
- **Ontology Editor** — author the schema online: classes and properties, domains, ranges, and subclass/subproperty hierarchy. Edits are validated for referential integrity and cycles. New classes trace to one of the QROS base classes.
- **Rules** — define, name, edit, and test rules. A rule's `WHEN`/`THEN` pattern computes inferences, and a named rule can serve as a query scope (`@context = rule:YourRule`). See the [Ontology Guide](../ONTOLOGY_GUIDE.md).
- **Dashboard** — instance metrics and team-built cards.
- **LLM Testbed** — ask a question in natural language; TReS generates a QQL query, runs it, and shows both the query and the result, so you can see how the answer was derived.

## Loading data

From **Admin → Load Data** (Admin, and Power User where enabled):

- **Folder** (recommended for spreadsheets, QROS instances) — point TReS at the folder your sheets live in; every sheet is analyzed together and you review one plan. See [Importing Your Data](../IMPORTING_YOUR_DATA.md).
- **Spreadsheet** — the single-sheet mapping wizard.
- **Upload** files — graph interchange files (.ttl, .nt, .rdf/.xml) are converted to native QROS on load; native QROS files (`.tqs` schema, `.tqd` data) load directly.
- **Load from Git** — pull from a configured Git source. Private sources authenticate with a stored access token managed by the operator.

After a load, the intelligence layer (VDB) rebuilds in the background so search, visualization, and pathfinding reflect the new data; the application stays responsive. See the [Import Guide](../IMPORT_GUIDE.md).

## Administration

From the **Admin** console (Admin role):

- **Users & roles** — invite or create users, assign roles, enable/disable accounts, resend invitations.
- **Page visibility** — turn surfaces on or off per role for this instance.
- **Usage & audit** — review administrative actions with actor and timestamp.

## Maintenance and updates

Instances are updated in **agreed maintenance windows**, with a brief unavailability during a deploy; the SPA retries transparently. Deployment procedures are in [`Documentation/deployment/`](../deployment/).

## Data protection and backups

Every TReS Cloud instance's data volume is protected automatically:

- **Daily backups.** A recovery point is taken every day and retained for
  **35 days**. Backups are encrypted and stored in a vault separate from
  the instance itself; taking one requires no downtime and does not
  interrupt work.
- **On-demand snapshots.** The operator can take an additional recovery
  point at any moment — for example immediately before a large load or a
  bulk change. Ask your operator when you want a known-good point saved.
- **Verified restore.** Restores recover the full data volume to a chosen
  recovery point. The restore procedure is exercised against real
  recovery points, not assumed from configuration.
- **Backups are the safety net, exports are yours.** Backups protect the
  instance; they are operator-side infrastructure. Your own copy of the
  knowledge — data, ontology, and training corpus — is always available
  through the export surfaces, portable and in open formats. See
  [Portability](../PORTABILITY.md).

To request an on-demand snapshot or a restore, contact your operator
(info@taurusystems.com for hosted instances).

## Troubleshooting

- **No invitation email** — check spam (sender: Amazon Cognito); an administrator can resend.
- **"Signed in on another device"** — expected when the same account signs in elsewhere; sign in again to take over.
- **A surface is missing** — your role may lack access, or an administrator hid it for your role.
- **Search/visualization looks incomplete right after a load** — the VDB may still be rebuilding; retry shortly.

## Getting help

Contact your instance administrator first. For product support, email info@taurusystems.com.
