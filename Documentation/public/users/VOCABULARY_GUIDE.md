# TReS Vocabulary Management Guide

**updated 2026-08**

A guide for managing controlled vocabularies and taxonomies in TReS. Written for vocabulary managers and taxonomists -- no programming required.

---

## How TReS Models Vocabulary

TReS treats a vocabulary as **controlled reference data that lives natively in your graph** -- not as a separate artifact bolted on beside your business data. One distinction makes everything else clear.

### Functional data vs. metadata

Every dataset holds two kinds of things:

- **Functional (transactional) data** -- the records your system creates through normal operation. A case-management system mints new *cases* every day. A catalog system mints new *titles* and *storylines*. A product system mints new *products* and *releases*. These grow continuously as people use the system.
- **Vocabulary (metadata)** -- the controlled values those records refer to, which change slowly and are maintained deliberately, often from an external authority. Think *license types*, *business functions*, *specifications*, *classification codes*, *roles*, *regions*. Normal use does **not** mint new ones; a curator adds or revises them on purpose.

The test for "is this vocabulary?" is **how a term is used, not what type it is.** Vocabulary is the slowly-changing, controlled reference set that describes and classifies your functional data. That functional-vs-metadata split is the heart of the model -- get it right and the rest follows.

### Vocabulary is a layer, not a separate scheme

TReS keeps vocabulary in a **vocabulary layer** -- an informal, in-graph partition on the same substrate and the same term model as the rest of your data. It carries its own small set of classes (its "mini-ontology") alongside the terms themselves. This separates vocabulary for editing and review **without divorcing it from the data it describes**: a term stays a first-class node that your functional records point at directly.

Because the layer is what marks a term as vocabulary, it doesn't matter which named graph physically holds it -- TReS identifies your vocabulary by its layer, wherever it lives.

### Classes and terms -- no concept schemes

TReS does not use concept schemes. There are only **classes** and **terms** (instances):

- **Classes** do the classifying. A class such as *Region* or *License Type* groups the terms that belong to it. Any class declared under the built-in `Vocabulary` base automatically belongs to the vocabulary layer.
- **Terms** are the controlled values themselves -- the individual regions, license types, or codes.
- A term may carry a **broader** link to another term where a genuine ladder helps (a narrower region inside a wider one, for instance). The ladder is optional, not required scaffolding.
- **`broader` needs no concept scheme to work.** It links any two terms directly, and the engine acts on it: when rules run, classification **propagates up the ladder automatically** -- a record classified to a narrower term also answers rules and queries written against any broader ancestor, as a derived fact with full lineage (the original classification plus the broader links it climbed). Curate the ladder once and every level of it becomes queryable.

This is deliberate. Real controlled vocabularies are often **richly interconnected rather than strictly hierarchical**: a character relates to teams, equipment, and settings; a specification relates to the components and standards it governs. Those are a web of relationships, not a single parent-child tree. A rigid scheme pushes everything toward one hierarchy; TReS lets the classes handle classification and lets terms interconnect as a graph where reality calls for it -- and form a ladder where that is the clearer picture. You get both.

### Why it matters for AI grounding

Because the vocabulary layer is a native, labeled part of the graph, it doubles as a **glossary of controlled values for grounding.** When an AI assistant works over your data, the vocabulary layer tells it exactly which values are the sanctioned, controlled terms for domain-specific concepts -- the authorized answers, distinct from the transactional records. A well-maintained vocabulary is the difference between an assistant that guesses and one that speaks your organization's controlled language.

---

## Understanding Your Vocabulary Data

### What is a Named Graph?

Your database stores different kinds of data in separate containers called **named graphs**. Think of them as folders within the same filing cabinet. The filing cabinet is your database; each folder holds a different collection of data.

For example, your organization might have:
- A **schema graph** containing the data model (classes, properties, rules)
- A **vocabulary graph** containing your controlled terms and their hierarchies
- An **instances graph** containing the actual business data

All of these live in the same database, but they are organized into their own named graphs so they don't get mixed together.

**Why this matters here:** the Vocabulary tab never asks you to pick a graph. Because vocabulary is identified by its **layer** (see *How TReS Models Vocabulary*, above), the tab gathers every vocabulary class and term in the database, whichever named graph physically holds it. New classes and terms you create in the tab are written to the dedicated vocabulary graph automatically.

### Interchange with other vocabulary standards

Controlled vocabularies are often exchanged in standard graph formats built around **concept schemes**. TReS imports and exports those interchange formats -- so if your vocabulary already lives in one, it loads without conversion. More broadly, third-party vocabularies of any kind can be loaded and used as-is: external namespaces and datatypes remain valid, and nothing forces conversion.

Inside TReS, though, the working model is **classes and terms**, as described above -- not concept schemes. On import, TReS treats each scheme as a class and each concept as a term; on export, it regenerates the concept-scheme form on the way out. You get interchange compatibility at the edges and a simpler, more flexible model in the middle. Familiar fields map straight across:

- **Preferred Label** -- the term's main display name
- **Alternative Label** -- synonyms or other names for the same term
- **Broader / Narrower** -- optional parent-child links between terms
- **Definition** -- a description of what the term means

---

## Getting Started

### Opening the Vocabulary Tab

1. Click **Vocabulary** in the main menu bar
2. TReS loads every vocabulary class and term in the vocabulary layer -- no graph selection needed

The tree opens with each class and its first level of terms expanded, so you see the structure at a glance. A count above the tree tells you how many classes and terms were loaded.

If the tab shows no vocabulary yet, load one via the Database Management page and it will appear here.

### Refreshing Data

The tree reloads automatically after every change you make in the tab. To pick up changes a teammate has made in the meantime, reload the page in your browser.

---

## Browsing Your Vocabulary

### The Two-Panel Layout

The Vocabulary tab is divided into two panels:

**Left Panel -- Vocabulary Tree**
This is where you browse and search your vocabulary. Each vocabulary class appears as a collapsible heading, with its terms organized in a hierarchy beneath it based on broader/narrower relationships. A class's description appears under its heading when the class is expanded, and hovering over any row reveals a copy button for that row's URI.

**Right Panel -- Detail**
The full record for whatever you select on the left.

### Navigating the Tree

- Click the arrow on a **vocabulary class** heading to expand or collapse it
- Click the **class name itself** to see the class's own record in the detail panel -- its label, URI, and everything the database knows about it. Handy for verifying the exact class before mapping an upload against it.
- Click a **term** to see its details:
  - **Also Known As** -- alternative labels (synonyms), with language tags where present
  - **Definition** -- what the term means
  - **Scope Note** -- usage guidance for taxonomists
  - **Notes** -- comments, notes, and examples
  - **Broader / Narrower** -- the term's neighbors in the hierarchy; click one to navigate to it
  - **Class** -- the vocabulary class (or classes) the term belongs to
  - **URI** -- the technical identifier, with a copy button
  - **Statements & Tags** -- the term's underlying statements and their tags, for a curator who wants to see exactly what is recorded

A term with no broader or narrower links is shown as a standalone term. That is perfectly valid -- the ladder is optional.

### Searching

Type in the **search bar** at the top of the tree to filter terms by preferred label. Matches are highlighted, branches auto-expand so every match and its ancestors stay visible, and a match count appears below the search bar. Click the **X** button to clear the search.

### Expand All / Collapse All

Use these links above the tree to open or close every class and term at once -- helpful when you need to scan the entire vocabulary or return to a compact view.

---

## Editing Vocabulary

Editing requires a role with data-editing permission. Without it, the tab is read-only and the controls below do not appear.

### Enabling Edit Mode

Click the **Edit Mode** button at the top of the page. This reveals editing controls throughout the tree and the detail panel. **When you are done editing, click Edit Mode again to turn it off.** This prevents accidental changes while browsing.

### Creating a Vocabulary Class

Click **+ Vocabulary** above the tree to create a new vocabulary class. The dialog asks for:

- **Preferred label** (required) -- the class's display name, e.g. *Genre*
- **Base class** -- the class this vocabulary descends from; any existing class qualifies, and the default is the built-in *Concept*
- **Description** (optional) -- a short note on what this vocabulary is for

### Adding Terms

- Hover over a class heading and click **+ Concept** to add a top-level term under that class
- Hover over any term and click **+** to add a narrower (child) term below it

Either way, the dialog asks for:

- **Preferred label** (required)
- **Alternative labels** (optional) -- synonyms, comma- or newline-separated
- **Additional class** (optional) -- also types the new term as another class picked from the list
- **Definition** (optional) -- what the term means
- **Scope note** (optional) -- usage guidance for taxonomists

TReS generates the term's URI for you and places the new term in the tree, selected, so you see its detail immediately.

### Editing a Term

Select the term, then click **Edit** in the detail panel. You can change the preferred label, replace the alternative-label list, edit definitions, scope notes, and other annotations, and adjust which classes the term belongs to -- the classes appear as removable chips, with a picker to add more. This is the fastest way to clean up a term that an import typed against the wrong class.

### Moving a Term

Click **Move** in the detail panel to give a term a new parent. In the picker, click a vocabulary class to make the term a top-level term of that class, or click another term to make it a child of that term. The term being moved and its own descendants are grayed out so a move can never create a cycle. After a move, an **Undo** banner appears for one minute in case you have second thoughts.

### Editing or Deleting a Class

Hover over a class heading for its **Edit** and **Delete** buttons.

- **Edit** changes the class's title or description.
- **Delete** removes the class. If the class has terms, you choose what happens to them: delete them along with the class, or move them all to another class first.

### Deleting a Term

Click **Delete** inside the term's edit view and confirm. A term with narrower terms cannot be deleted until its children are removed -- TReS tells you exactly which ones are blocking.

---

## Exporting Your Vocabulary

Use the **Documentation** buttons at the top of the page to generate a formatted document of your vocabulary and download it through your browser:

| Format | Best For |
|--------|----------|
| **HTML** | Styled, self-contained documentation to share with a wider audience |
| **Markdown** | Wikis, Notion, Confluence -- and Word, which imports Markdown directly |

The document is generated from the live vocabulary: classes, terms, the broader/narrower hierarchy, labels, and definitions.

---

## Working with Multiple Languages

If your vocabulary contains labels in multiple languages (e.g., English, French, Spanish), TReS shows content based on your language preference:

1. Go to **Preferences** (in the hamburger menu)
2. Set your **preferred language**
3. TReS displays labels in your preferred language first

The fallback order is: your preferred language, then English, then untagged labels, then whatever is available. You can change this at any time without affecting the data.

---

## Tips for Taxonomists

**Start from the counts:**
The class-and-term count above the tree is a quick sanity check after a load. If the number of terms is not what you expected, the upload may have landed against the wrong class -- click the class name to inspect its URI and verify your mapping.

**Checking vocabulary quality:**
Scan term details as you browse. Watch for:
- Missing **definitions** -- a term without one invites inconsistent use
- Few **alternative labels** -- synonyms are what make search and matching forgiving
- Terms with **no class** -- these will not appear under any class heading in the tree
- Very deep ladders -- hierarchies more than a few levels deep may be hard to navigate

**Fixing a bad import:**
A term's edit view shows every class the term is typed as, as removable chips. If an import attached the wrong classes, remove the bad chips and add the right one -- no reload required.

**Exporting for review:**
Use the Markdown documentation export when you need to review vocabulary with stakeholders in a wiki or in Word. Use the HTML export for polished documentation to share with a wider audience.

---

## Keyboard Reference

| Action | How |
|--------|-----|
| Search | Type in the search bar |
| Clear search | Click the X button |
| Expand the whole tree | Click Expand all above the tree |
| Collapse the whole tree | Click Collapse all above the tree |
| Toggle edit mode | Click the Edit Mode button |
| Close a dialog | Press Escape |

---

**TReS** - Vocabulary management designed for taxonomists.

*For general application usage, see the [Operations Guide](administrators/OPERATIONS_CLOUD.md). For database setup, see [Getting Started](GETTING_STARTED_CLOUD.md).*
