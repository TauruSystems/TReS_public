# TReS Administrator's Guide

**Revision 1.0 — 2026-08-20.** Describes the product as it ships today.

This guide is for the person who administers a TReS site — one instance,
serving one organization. It covers what an administrator can do, where each
of those things lives, what only Tauru Systems can do for you, and how to
tell which is which.

Your site is a complete, isolated instance: its own address, its own
storage, its own user directory, its own backups, its own credentials. It
shares nothing with any other TReS site. Nothing here requires installing or
compiling anything — administration happens in the application, in a
browser.

Where this guide says **Level 3**, it means work performed by Tauru Systems
on the infrastructure your site runs on (§10). That support is always
provided by us.

---

## 1. Roles, and what each one may do

Everyone who signs in holds exactly one role.

| | User | Power User | Admin | Owner | Auditor |
|---|---|---|---|---|---|
| Run queries, browse, visualize | yes | yes | yes | yes | queries only |
| Export results | yes | yes | yes | yes | yes |
| Ask the AI assistant | yes | yes | yes | yes | yes |
| Write and delete data, import, edit the vocabulary | — | yes | yes | yes | — |
| Manage dashboards and plugins | — | yes | yes | yes | — |
| Manage users (below Admin) | — | — | yes | yes | — |
| Invite, promote, demote or disable an Admin or an Owner | — | — | — | yes | — |
| Create and revoke service accounts | — | — | — | yes | — |
| Change instance settings | — | — | yes | yes | — |
| View the audit log | — | — | yes | yes | yes |
| See every user's questions to the assistant (everyone sees their own) | — | — | yes | yes | yes |
| Report a bug from inside the product | yes | yes | yes | yes | yes |

**Owner is the highest role in the application** — your organisation's seat.
An Owner holds everything an Admin holds and, alone, has authority over the
administrators: only an Owner can invite an Admin or another Owner, change their
role, or disable them. Nobody can change their own role or disable themselves.
That split is what lets you give our support staff an Admin or Auditor seat
without giving them the keys to your administrators. What sits above Owner is
not a role — it is the infrastructure your site runs on, which is ours to
operate.

**Auditor is deliberately narrow.** It reads the audit log, reviews every
user's questions to the assistant, and runs queries, and it changes nothing. Give it to someone whose job is to check what
happened, including checking on the administrators.

Two identities exist that no person holds. **Service accounts** are machine
identities (§4), read-only by construction. **Anonymous** applies only if
your site publishes a read-only public surface; most do not, and switching it
is Level 3 work.

---

## 2. The Admin page

Everything administrative lives in one place, and the tabs you see depend on
your role.

| Tab | What it is for | Who sees it |
|---|---|---|
| **Users & roles** | Invite people, set roles, disable accounts (§3) | Admin |
| **Service accounts** | Machine credentials for integrations (§4) | Owner creates and revokes; Admin sees |
| **Pages** | Which parts of the product each role sees (§5) | Admin |
| **Store** | How the store spells namespaces; the one-time move from folding to preserving them | Admin |
| **AI provider** | Which model answers questions, and whose key pays (§7) | Admin |
| **Usage & audit** | What has been asked, loaded and changed (§9) | Admin, Auditor |
| **User activity** | Logins and time on site (§9) | Admin, Auditor |

Data itself is not administered here — loading, graphs and exports live on
the **Database Management** page (§8), and your site's welcome page is
customized on the **Preferences** page (§6).

---

## 3. Managing people — Admin → Users & roles

- **Invite someone.** They receive a temporary password by email and must set
  their own on first sign-in. You choose their role at invitation.
- **Change a role.** Takes effect at their next sign-in, not mid-session.
- **Disable someone.** This is the correct action when a person leaves — not
  deletion. A disabled account cannot sign in and the audit trail stays
  intact; deleting it would break that history.
- **Resend an invitation** or **reset a password** for someone who never got
  the first mail or is locked out. Both issue a fresh temporary password and
  force a reset at next sign-in.

Every one of these is written to the audit log under your name. That is
deliberate: user administration is exactly what an auditor needs to review.

**Review who has access on a schedule.** Quarterly suits most organizations.
Departed staff retaining access is the most common finding in any access
review, and the most preventable.

---

## 4. Machine access — Admin → Service accounts

A service account is how a machine reads your data: a reporting tool, a BI
dashboard, an AI agent, an integration you have built.

- The key is shown **exactly once**, at creation, and cannot be retrieved
  afterward. If it is lost, revoke the account and create another.
- Revocation takes effect on that key's next request.
- A service account holds the role it was given at creation — read-only
  (Integration) unless you chose otherwise — and cannot sign in.
- Every call is logged under the service account's identity, and the list
  shows when each key was last used. **A key that has never been used, or has
  stopped being used, is worth questioning** — either something is broken or
  something is no longer needed.

Your site is entitled to a set number of active service accounts, and each
one's traffic is budgeted per minute. Both are part of your agreement;
raising either is Level 3 work.

### Creating one

Only an Owner may create or revoke a service account: a machine identity reads
your data on its own key, and that is the site owner's to grant. An Admin sees
the list and when each key was last used.

1. Go to **Admin → Service accounts** and choose to add one.
2. Give it a **name** and a **purpose**. Both are for you and whoever reads
   the audit log later — name the tool that will use it, not the person who
   made it, because the tool outlasts the person.
3. The key appears once. Copy it into wherever it will be used before you
   leave the page. There is no way to see it again.

Keys begin with `tres_sa_`. If a key ever appears in a ticket, a chat message
or a screenshot, revoke it and issue another — a key is a credential, not a
reference number.

If creation is refused because the account limit is reached, revoke one you no
longer need. The list shows when each key was last used, which usually settles
the question.

### Using the key

Every request carries the key in an `Authorization` header, and goes to your
site's own address (§11):

```
Authorization: Bearer tres_sa_...
```

Nothing else authenticates a machine. There is no username, no session and no
sign-in page for a service account.

### Reading data — the data endpoint

`POST /api/v1/data` is how a reporting tool, a dashboard or your own code
reads the graph. Send a query and get a result table back:

```
POST https://your-site/api/v1/data
Authorization: Bearer tres_sa_...
Content-Type: application/json

{"query": "FIND ?case WHERE { ?case :type :Case } LIMIT 10"}
```

- **`graphs`** optionally scopes the read to named graphs. Leave it out to
  read the whole store.
- **`format`** is `json` by default. Ask for `csv` there, or send
  `Accept: text/csv`, and the same result comes back as CSV.
- Query syntax is the same one the Query page uses; the QQL Guide covers it.
- Every response carries an **`X-TReS-Dataset-Fingerprint`** header, which
  changes when the data behind it changes. Use it to notice that a refresh is
  worthwhile. Do not use it as a cache validator: queries read live data,
  so just after a load the fingerprint briefly lags what a query already
  returns.

The endpoint is read-only. So is the account.

### AI agents — the MCP endpoint

`POST /api/v1/mcp` lets an AI agent use your site as a tool, over the Model
Context Protocol. Point the agent's MCP client at that address and give it the
same key in the same `Authorization` header.

The agent is offered three tools:

| Tool | What it does |
|---|---|
| `search` | Finds things by name or description when the agent does not yet know what exists |
| `query` | Runs a query and returns rows, the same engine the data endpoint uses |
| `ask` | Answers a question in language, using your data as its grounding |

`ask` needs an AI provider configured for your site (§7). The other two do not.

Two things commonly surprise people setting this up:

- **The endpoint only accepts POST.** A browser visit, or a client that opens
  a listening stream first, gets a polite refusal saying so. That is correct
  behaviour, not a fault.
- **A client running in a browser on some other website is refused.** Agents
  that run on a server are unaffected. This stops a page you did not write
  from using your key through someone's browser.

The agent's access is exactly the account's access: read-only, rate limited,
and written to the audit log under the service account's name.

### When machine access does not work

| What you see | What it means |
|---|---|
| `401 authentication required` | No key, a mistyped key, or one that has been revoked |
| `403` mentioning your plan | Machine access is not part of your agreement — Level 3 (§10) |
| `403` mentioning the account limit | You are at your entitled number of active accounts |
| A refusal saying POST only | The client used the wrong method on the MCP endpoint |
| A web page instead of an answer | The address is wrong. Any address the product does not recognise returns the application page, so a typo looks like success until you read the body |

---

## 5. What each role sees — Admin → Pages

You control which parts of the product are enabled and which roles reach
them. Turn off what your organization does not use — a page nobody needs is
a page that generates questions.

One rule to keep in mind: **signed-out visitors see what the User role sees.**
If your site serves anonymous readers, scoping a page to User is what makes
it public. If it does not, this is simply the read-only tier.

---

## 6. Your welcome page — Preferences → Customize

The Welcome page can carry your organization's logo and a title of your
choosing — "TReS for" your programme's name, for instance — in place of the
TReS logo and "Welcome to TReS". Set both in the **Customize** section of the
Preferences page ("Custom Welcome Page") — it appears there for Admin and above
only: type the title (up to 80 characters; blank restores the default), set or
hide the tagline under it, and upload the logo
(PNG or JPEG, up to 512 KB; it is shown centred, up to 128 pixels tall, so
a wide mark on a transparent background reads best). The change is live on
the next page load for everyone, including signed-out visitors if your site
serves them.

The header keeps the TReS logo and name in every case. Each change is
written to the audit log under your name.

## 7. The AI provider — Admin → AI provider

Your site answers questions using a model. By default that is the provider we
configure for you. You can attach **your own** instead — Ollama, Claude, or
OpenAI — by entering the provider, model, optional endpoint, and your key.
The change takes effect immediately, and removing your configuration reverts
to the provider we supply.

**Your key is write-only.** Once saved it is never returned by the
application to anyone, at any role. The page shows only that a key is
configured, when it was set, and who set it. If you need the key itself, get
it from the provider — we cannot show it to you, and neither can your own
administrator.

Bringing your own key moves the cost of inference to your account. It does
not change your entity ceiling: retrieval and indexing still happen on your
instance.

---

## 8. Your data — Database Management

Three tabs: **Load**, **Graphs**, and **Log**.

### Load

Load from a spreadsheet, a folder of files, or a connected Git source. **A
load writes exactly one graph, and you pick it on the Load tab** — deliberate,
so data never lands somewhere you did not choose.

Loads are **all-or-nothing**: either the whole load is applied or none of it
is. The load report states what was written. Compare against it before
concluding anything went missing.

### Graphs

- **Clear** a graph — empties it, keeps it.
- **Delete** a graph — removes it and its role declaration.
- **Download** a graph, or your rules, in open formats.
- **Delete by source** — remove stored statements by where they came from.
  This is the tool for retracting one bad load or one retired feed without
  disturbing anything else, because every statement carries its source.

Clearing and deleting are destructive and take effect immediately. Your
recovery point is the last backup, not an undo button.

### Log

The history of loads against this site — what ran, when, and what it wrote.

### Capacity

Your site has an entity ceiling. The index badge turns amber at 80% and shows
the count against the ceiling; at 100% new loads are refused, with a message
naming the ceiling. That refusal is deliberate, and better than what it
replaced — the instance running out of memory mid-load. Raising the ceiling
is Level 3 work.

### The search index

The index behind search, visualization and the assistant updates itself
whenever data changes, and your site keeps answering from the previous index
while it does — the badge reads "updating · serving N entities", then shows
how far the update has got and about how long remains once it has a rate.
Two times matter when you load something large, and we will tell you both:
**load time**, which the load itself reports as it runs, and **index time**,
the catch-up behind it. As a guide from our measurements: a file of a million
statements loads in a few minutes and indexes in well under an hour; one
that adds more than a fifth of your data at once is what we call a major load
event, and we size your site up for it in advance — ask us before a load of
that shape, and we will schedule it and tell you what to expect.

There is a manual rebuild control, and you should
almost never need it: **every write path already schedules a rebuild, so an
index that needs rebuilding by hand means something did not fire.** If you
ever press it, tell us. It is evidence of a defect, not routine maintenance.

### Backups

Your site is backed up **twice daily**, retained 35 days in a locked vault
with an off-region copy. What follows is a **12-hour recovery point
objective**: the most work a failure can cost you is half a day. Restoring is
Level 3 work — tell us what happened and roughly when, and we will tell you
which recovery points exist before anything is restored.

### Export

You can take your data out at any time, in open formats, without asking us.
That is a deliberate property of the product, not a courtesy.

---

## 9. Seeing what happened — Admin → Usage & audit, User activity

**Usage & audit** shows usage for the instance, and the audit log beneath it.
The log records questions asked of the assistant and what it was taught
(each correction added or removed, with its text), queries run, data loads,
every user and role change, and edits to the vocabulary, ontology and rules —
each with the actor's identity and the time. It exports.

**User activity** shows logins and time on site.

Both tabs are visible to **Auditor** as well as Admin, so oversight does not
require giving someone the ability to change things.

---

## 10. What only Tauru Systems can do — Level 3 support

Some things that shape how your site behaves are not exposed in the
application. They live in the infrastructure, we change them on request, and
the change is recorded.

Our reach does not depend on the application being healthy. Level 3 work is
done with command-line tooling against your instance directly — the storage,
the user directory, the configuration, the running version — so a site that
will not start, an index that will not rebuild, or a user directory the
application cannot repair are all still fixable. That is precisely when it
matters.

| Request | What it affects |
|---|---|
| Raise the entity ceiling | How much data your site holds |
| Raise the service-account count or its per-minute budget | How much machine traffic your site serves |
| Change the operator-provided model | Which model answers when you have not attached your own (§7) |
| Turn a public read-only surface on or off | Whether unauthenticated visitors can read |
| Change session and idle timeouts | How long a signed-in session survives |
| Restore from a recovery point | Recovering from data loss or a bad load |
| Upgrade the version your site runs | New features and fixes |
| Recover the user directory | Sign-in accounts the application cannot repair itself |

**How to ask.** Tell us the outcome you need rather than the setting you think
produces it — the mapping changes between releases, and describing the
outcome lets us tell you if there is a better route or a consequence you
would not have expected.

**What we cannot do:** recover a password, or read your data on your behalf
as a shortcut. Password resets go through the application (§3).

---

## 11. Your site's address

Your site is served at its own address on `taurusystems.com`, assigned when the
site is created:

```
https://tres260901a.taurusystems.com
        ────┬───────
        tres  the date the site was created (YYMMDD), then a letter
              distinguishing sites created on the same day
```

**The address does not change.** Bookmarks, saved sign-ins, the configuration
inside any agent that connects through a service account (§4), and anything you
have integrated all point at it. An address that moved would break them at the
same moment, so treat it as permanent.

**Moving between tiers does not change it either.** A change of tier adjusts
what your site is allowed to do and, if needed, the size of the machine it runs
on. Your data stays where it is and your address stays what it was.

**Vanity and custom addresses.** Sites at Curation, Connection and Automation are
served on the assigned address; we do not offer chosen or vanity addresses at
these tiers. A custom domain on your own DNS may be considered at the Operations
tier.

**If your organization runs more than one site,** the date in the address is what
tells them apart: a site created on 1 September 2026 is `tres260901a`, and a
second site created that day is `tres260901b`. Record which is which somewhere
your team will look — the address is deliberately neutral, and it will not tell
them for you.

---

## 12. When your site changes

**Changes to a live site happen between 17:30 and 22:30 US Eastern, any day
of the week.** Upgrades and configuration changes that replace the running
application are made inside that window.

The window exists because replacing the running application is not invisible.
Everyone signed in is signed out and must authenticate again. Requests in
flight at the moment of the swap are dropped. The search index reloads on the
way back up, and search and visualization are degraded until it does. None of
that belongs in the middle of a working day.

**Two exceptions,** stated so they are not surprises: your site is already
down, in which case the window protects nothing — we fix it and tell you what
happened; or a security fix that should not wait, in which case we tell you
before, or as close to before as the situation allows.

If those hours do not suit your organization, tell us. It is a schedule, not
a law of physics.

---

## 13. When something looks wrong

Three checks answer most questions, and all three are yours to run.

**Can anyone sign in?** If one person cannot, check the account is enabled and
holds the role you expect (§3). If nobody can, escalate.

**Is the data wrong, or did the load not land?** Loads are all-or-nothing and
the report states what was written. A load refused for capacity says so
explicitly.

**Is the index rebuilding?** After a version change, search and visualization
are briefly degraded while it reloads. Well beyond a few minutes, escalate.

**When you escalate:** what you were doing, what you expected, and roughly
when. If it reproduces, the exact question or query. The audit log (§9) often
answers who did what and when before we can.

---

## 14. What this guide does not cover

Writing queries, designing an ontology, and preparing data for import are
covered by the QQL Guide, the Ontology Guide, and the Import Guide. How the
service is hosted and recovered is ours to operate.

---

## Getting help

Your first stop for anything in §§3–9 is this guide and the audit log. For
Level 3 requests (§10), anything you believe is a defect, and anything
affecting more than one person on your site, email
**info@taurusystems.com**.

When you write, include what you were doing, what you expected, and roughly
when it happened. If it reproduces, the exact question or query. That is
usually the difference between a same-day answer and a conversation.
