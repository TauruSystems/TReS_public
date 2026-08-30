# Getting Started with TReS Cloud

TReS lets the people who own the data turn it into knowledge they — and their AI — can trust, without waiting on IT or a data team. Every fact carries where it came from, how reliable it is, and where it applies, so the answers hold up. Powered by QROS, a new kind of engine.

TReS Cloud is the hosted, multi-user edition and the primary way most teams run TReS. Your organization has its own instance with its own login and its own database — there is nothing to install and no database to operate.

## Sign in

1. Open your instance URL in a browser — for example `https://your-org.taurusystems.com/`. Your administrator provides the exact address.
2. Sign in with the email and **temporary password** from your invitation email (sender: Amazon Cognito). Check spam if you don't see it.
3. You will set a permanent password on first sign-in.

If your invitation never arrived or expired, ask your administrator to resend it.

## The interface

- **Header** — the TReS logo and your current connection.
- **Main menu** — the surfaces your role can see (Query, Visualization, Vocabulary, and others).
- **Page row** — actions for the current page.
- **Graph selector** — scope your work to a named graph or to all graphs.

## Five-minute tour

1. **Ask a question.** Open **Query** and write a QQL query (see the [QQL Guide](QQL_GUIDE.md)). QQL is designed to be readable and to query a statement's tags — confidence, validity, and context — directly. A scope- or time-aware query also returns its **support set**: what the answer was computed over.
2. **Browse.** Explore classes and instances without writing a query.
3. **Visualize.** Search for an entity by label, place it, **Expand** its relationships, and **Find Path** between two nodes.
4. **Inspect an entity.** Open **Entity Info** to read an entity's labels, types, properties, and what references it. Where statements carry tags (confidence, source, validity, context), you'll see them on the entity.
5. **Browse the vocabulary.** Navigate concept schemes and concepts; create and edit concepts if your role allows.
6. **Check the Dashboard.** Instance metrics and any cards your team has built.

## Writing data on a QROS instance

QROS instances are **query-only at the Query page** — you don't run write queries there. Instead, you author data through purpose-built surfaces:

- **Entity authoring** — add and edit entities, with their contextual tags.
- **Ontology Editor** — author the schema: classes and properties, domains, ranges, and hierarchy.
- **Vocabulary** — manage concept schemes and concepts.
- **Rules** — define named rules that compute scope and inferences (see the [Ontology Guide](ONTOLOGY_GUIDE.md)).

See the [Cloud Operations Guide](administrators/OPERATIONS_CLOUD.md) for the full workflow.

## If you're an administrator

- **Load data** from **Admin → Load Data**: point TReS at a folder of spreadsheets (the recommended path — see [Importing Your Data](IMPORTING_YOUR_DATA.md)), upload files, or pull from a configured Git source (see the [Import Guide](IMPORT_GUIDE.md)). Files are stored natively in QROS.
- **Edit the schema** online in the **Ontology Editor**.
- **Manage users** from the **Admin** console: invite users, assign roles, control page visibility.

## Good to know

- **Your role** determines what you can do — read-only, read-write, or administrator. A missing surface usually means your role lacks access; ask your administrator.
- **Sessions** time out after inactivity, and only one active session per account is allowed.
- **Your data is private to your instance.** No other customer shares your database.

## Getting help

Contact your instance administrator first. For product support, email info@taurusystems.com.
