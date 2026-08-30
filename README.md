# TReS — LLM Grounding Platform

![TReS](static/icon.png)

TReS makes an AI answer accurately over your organization's own data. It is the toolchain for
turning the information you already have — spreadsheets, exports, documents, database extracts —
into knowledge a language model can be trusted with, where every fact carries its source, its
reliability, and the context it applies in.

It is built for the people who know the data: analysts, data stewards, taxonomists, and the
business units that own the information — not only for developers.

**TReS is a hosted, multi-user web application.** Each organization gets its own dedicated
instance with login, role-based access, and its own database. There is nothing to install and no
database to operate.

## The substrate

TReS runs on **QROS**, a knowledge substrate built for this job: statement-level provenance,
validity in time, content-addressable identity, and closed-world answers, so the store can say
*no* rather than *nothing found*.

**RDF and SPARQL are fully supported as a backend.** Every read and write routes through one
storage-neutral seam, so an instance runs on either substrate and can move between them without
the surfaces above changing. QROS-native instances need no separate database; RDF instances run a
bundled Apache Jena Fuseki store private to that instance.

## Accessing an instance

Each customer instance is reached at its own URL. Sign in with the email and temporary password
from your invitation; you set a permanent password on first login. Authentication is delegated to
Amazon Cognito, and sessions enforce an idle timeout and a single active session per user.
Customer names are never used in a TReS URL.

## Roles

Access is role-based, and an admin controls which surfaces each role can see.

| Role | Can do |
|---|---|
| **Admin** | Everything, plus user management, page visibility, audit, and data loading |
| **Power User** | Read and write data — query, edit vocabulary, author data |
| **User** | Read-only across the enabled surfaces |
| **Auditor** | Read-only plus access to the audit log |

An instance can additionally be configured to serve **unauthenticated visitors a read-only view**.
This is off by default and is used for public demonstration instances, never for customer data.

## What you can do

- **Import** — guided loading from spreadsheets and exports, with data problems surfaced as you go
  rather than blocking you first. RDF formats (Turtle, N-Triples, RDF/XML and related) upload
  directly, and RDF can also be pulled from configured Git sources.
- **Model** — author the schema in the browser: classes, object and datatype properties, domains
  and ranges, subclass relationships, with referential and cycle validation.
- **Vocabulary** — manage concept schemes and concepts in an interface built for taxonomists
  rather than programmers.
- **Query and browse** — run queries with validation and highlighting, and browse classes,
  properties, and instances scoped by a graph selector. QROS instances use QQL; RDF instances use
  SPARQL, and the surface adapts.
- **Visualize** — graph views with meaning-bearing layout: search, expand from a node, and find
  paths between nodes.
- **Inspect** — a sectioned view of any entity: labels, types, outbound properties, and inbound
  references, with click-through navigation.
- **Dashboard** — instance metrics plus cards you build over your own data.
- **Ground an AI** — grounded chat over your knowledge, with an evaluation loop that measures
  whether grounding actually improved the answers.
- **Administer** — user and role management, per-role page visibility, and an audit view of
  administrative actions.

## For operators

Each instance is a dedicated AWS Fargate stack — its own Cognito user pool, load balancer, task,
and persistent storage — stamped from Terraform/OpenTofu. **There is no shared runtime between
customers.** Customer-instance deploys happen inside an agreed maintenance window; the
demonstration instance is exempt.

Operating an instance — sizing, maintenance windows, backups and upgrades — is described in
the [Cloud Operations guide](Documentation/public/administrators/OPERATIONS_CLOUD.md).

## Documentation

- **Start here:** [Getting Started](Documentation/public/users/GETTING_STARTED_CLOUD.md) ·
  [QROS Introduction](Documentation/public/users/QROS_INTRODUCTION.md)
- **The substrate:** [QQL Guide](Documentation/public/users/QQL_GUIDE.md) ·
  [Portability](Documentation/public/users/PORTABILITY.md)
- **Working with your data:** [Importing Your Data](Documentation/public/users/IMPORTING_YOUR_DATA.md) ·
  [Ontology Guide](Documentation/public/users/ONTOLOGY_GUIDE.md) ·
  [Vocabulary Guide](Documentation/public/users/VOCABULARY_GUIDE.md)
- **AI and evaluation:** [LLM Provider Guide](Documentation/public/users/LLM_PROVIDER_GUIDE.md) ·
  [Model Improvement](Documentation/public/users/MODEL_IMPROVEMENT_GUIDE.md)
- **Running it:** [Operations](Documentation/public/administrators/OPERATIONS_CLOUD.md) ·
  [Administrators Guide](Documentation/public/administrators/ADMINISTRATORS_GUIDE.md) ·
  [CLI Tools](Documentation/public/administrators/CLI_TOOLS.md)

## License

**Proprietary** — see [LICENSE](LICENSE). All rights reserved.

A Tauru Systems product. Support: info@taurusystems.com
