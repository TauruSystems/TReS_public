# What's New on Your Upgraded Instance

Your TReS site has moved to a new version. This is a short guide to what changed
and — mostly — what didn't.

The headline: **your day-to-day stays the same.** Same login, same data, same tabs,
same way of working. You'll find a few new things you can do (most of them around
*tags*), and one thing that works differently (*how queries are written*). That's
it. Nothing you rely on has been taken away from the way you actually use the
product.

## Signing in

Nothing to do. Your account came across automatically — **the same email and the
same password you already use.** You won't be asked to re-register or reset
anything. The first time you sign in, it may take a second longer than usual while
your account is set up behind the scenes; after that it's normal.

## What stays exactly the same

- Your **data** — everything that was there is there, with the same labels and the
  same structure.
- The **layout** — the same header, the same tabs (Welcome, Query, Viz, Database
  Management, Vocabulary), the same menus.
- **Browsing and searching** — finding an entity, opening its page, seeing what
  it's connected to, the type-ahead search.
- **Visualization** — searching for a node, expanding connections, finding a path
  between two things, improving the layout.
- The **dashboard**, the **catalog**, the **ontology editor**, the **vocabulary
  editor**, and **import/export**.
- Your **roles and permissions** — who can see and do what is unchanged.

If your job is mostly browsing, searching, and visualizing, you may not notice any
difference at all beyond a couple of new buttons.

## What's new: tags

This is the main thing you gain. You can now attach a few facts *to a fact*. When
you record a statement — say that one thing relates to another — you can also note,
right there on that statement:

- **Where it came from** — a source, so anyone (or any assistant) looking at it
  later can see where it originated.
- **How confident you are** — a simple confidence level, when something is known
  versus estimated.
- **When it applies** — a valid-from / valid-until period, so a fact that was true
  for a stretch of time carries its own dates instead of living in your memory or a
  separate spreadsheet.
- **The context it holds in** — the situation a fact is true *within*, so two facts
  that would otherwise look contradictory can each carry the circumstances they
  apply to.

You add these through the editor when you create or edit a statement — there's a
small panel for it, with a field for confidence and inputs for the rest. You
don't have to use any of it; a plain statement with no tags works exactly as
before. Tags are there for when the extra context is worth capturing.

Why it matters: this is how your data starts carrying its own provenance and
context, instead of that knowledge living only in people's heads. It's also what
lets an assistant give you an answer *with its sources shown*.

## What's different: how queries are written

If you write your own queries, this is the one real change. The query language is
now **QQL**. It does the same job — ask a question, get rows back — and the Query
tab works the same way (type a query, run it, see results, export them). The syntax
is different, and QQL can do a couple of things the old language couldn't, like
filtering on the tags above or asking a question *as of* a particular date.

- The full reference, with examples, is in **QQL_GUIDE.md**.
- If you don't write queries yourself — if you explore through Browse and
  Visualization — nothing here affects you.

One related change: queries are now for *asking*, not *changing*. The old "Update"
mode (editing data by writing a query) is gone. You still edit data — you do it
through the **Ontology, Entity, and Vocabulary editors**, the same screens you'd
use to add a class, mint an entity, or fix a label. For most people this is how
they edited anyway; if you used update-by-query, those edits now happen in the
editors.

## If your instance includes the assistant

Not every instance has the AI assistant turned on. If yours does, you'll notice it
answers from your actual data and **shows you the sources** behind an answer — and
when your data doesn't contain the answer, it tells you so plainly instead of
guessing. That honesty is deliberate: a straight "there's no record of that" is
more useful than a confident wrong answer.

## In a nutshell

| | |
|---|---|
| **Same** | login, data, tabs, browsing, search, visualization, dashboard, editors, import/export, permissions |
| **New** | tags on statements — source, confidence, valid dates, context — added in the editor |
| **Different** | queries are written in QQL (see QQL_GUIDE.md); data is edited in the editors, not by query |

Questions or anything that doesn't look right: [info@taurusystems.com](mailto:info@taurusystems.com).
