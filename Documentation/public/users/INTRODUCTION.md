# An Introduction to TReS

## What it is

TReS is an LLM Grounding Platform. It turns the data your team already owns — spreadsheets and application exports — into knowledge you and your AI can trust. Every fact in TReS can carry with it where it came from, how reliable it is, and the context in which it applies, so an answer can always show its support.

TReS runs on QROS, a knowledge engine built for exactly this job: closed-world, provenance-bearing, and checkable the moment data is written. See [An Introduction to QROS](QROS_INTRODUCTION.md) for the engine on its own terms.

## Who it's for

Teams who own data and need to stand behind it. The typical TReS user is a business-unit worker — technically literate but not a developer — who is responsible for data that lives today in spreadsheets and departmental systems, and who needs that data to be dependable: for reporting, for analysis, and increasingly for AI. You do not need to know a schema language or a query language to start; TReS's import tools read the files you have and its curation surfaces work in plain terms.

## Why it's different

- **Facts carry their own context.** Source, confidence, validity dates, and scope are attached directly to each statement — not bolted on in a side table. When conditions change, the answers change with them.
- **Problems surface immediately.** QROS checks data against your schema the moment it is written, in a closed world where "we don't know" is a detectable state rather than a silent gap.
- **It starts from what you have.** Point TReS at a folder of spreadsheets and promote them into governed knowledge step by step. There is no prerequisite modeling project. Data may be added and edited right in the interface, with specific handling for taxonomy and document catalogs. 
- **AI answers are grounded — and show their work.** Many tools claim grounding; in most of them it means text retrieval you have to take on faith. TReS goes further in two ways. First, on its native QROS engine the model never writes its own queries: it declares what it needs — a term, an entity's facts, a count, a path — and the engine authors and runs the query over the complete data, eliminating the wrong-query failures that text-to-query tools accept as normal. Second, the engine resolves the hard questions before the model reasons: expired values are excluded from "current" answers, untrusted sources are flagged, and conflicting values are reported as disputed rather than silently picked — and the model sees exactly the same resolved facts a human auditor sees on the entity page. Every answer rests on statements you can open, judge, and correct, and because QROS is closed-world, "that is not in the knowledge" is a real answer, not a gap the model fills with a guess.
- **One system, not a stack.** Import, curation, quality evaluation, query, visualization, and grounding are one platform over one engine, with role-based access control throughout.

## How to get it

Try TReS at **[tres.taurusystems.com](https://tres.taurusystems.com)**. Sign-in is open: enter your email address, confirm the verification code, and you are in the demo instance — no invitation or installation required.

When you are ready for an instance of your own, contact us and we will set one up for you; self-service sign-up with a choice of service tier is being prepared and is not yet open (updated 2026-08-28).

Questions? Contact [info@taurusystems.com](mailto:info@taurusystems.com).
