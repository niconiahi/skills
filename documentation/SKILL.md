---
name: documentation
description: How every document in this repo is kept — CONTEXT.md, docs/, README, ADRs, skills. Present tense only, the current state only, each fact in one place, written per the writing skill. Use when amending any doc after a change lands, or when the document agent runs.
---

# Documentation

A document here states how the system is today. Nothing else: no record of how it got there, no note of what is coming. Git is the only history, and the issue that did the work is the only place its investigation lives. Every document is written per [writing/SKILL.md](../writing/SKILL.md), with artifacts interleaved per [artifact/SKILL.md](../artifact/SKILL.md), exactly like any other text in the repo.

## Present tense only

Every sentence describes the current state. `DOCUMENT_ADAPTERS` derives from `source.ADAPTERS`; the census is a ledger row per work; a chunk lands by fast-forward.

When a behaviour changes, the sentence that described the old behaviour is rewritten to the new one and the old one disappears; the reader never learns it was different.

```
stale    The adapter list was hand-written and was replaced by a registry in 936.
current  The adapter list derives from the registry in `internal/source`.
```

The same rule bans the future: a sentence about what will hold once something lands is stale the day it lands, so it is never written.

A change with a reader-facing consequence amends the doc in the same commit series that ships it.

## Skinny and non-repeating

A small set of documents each carrying one subject beats an accumulation.

Every fact lives in exactly one place; when two documents state the same thing, it stays where a reader looks for it and the other points there in one line or says nothing.

A document that grows by appending is the failure mode: when a fact changes, the amendment is a rewrite of the paragraph that owns it, never a second paragraph beside it restating or correcting the first.

```
duplicated   README explains the ledger lifecycle; docs/agents/pipeline.md explains it again
single       docs/agents/pipeline.md owns it; README says "the lifecycle is in docs/agents/pipeline.md"
```

A section holds one or many paragraphs, each carrying one idea, exactly as the writing skill shapes any other text. A new idea that belongs to the section is a new paragraph under its heading, never folded into a paragraph that already carries something else; one overloaded paragraph per heading is as much a failure as a duplicate fact.

```
overloaded   ## Ledger
             One paragraph covering the row shape, the lifecycle states and the census query.

split        ## Ledger
             A paragraph on the row shape.
             A paragraph on the lifecycle states.
             A paragraph on the census query.
```

A fact the environment already states is not repeated either. A command's flags live in the justfile and the just skill, a schema in the migration, a signature in the code; a document caches one only when the lookup is expensive or the reason behind it is unwritten anywhere else.

## Where a fact lives

[CONTEXT.md](../../../CONTEXT.md) holds the vocabulary: what a term means, nothing about how it is implemented. [docs/adr/](../../../docs/adr/) holds decisions, one subject per record, per [adr/SKILL.md](../adr/SKILL.md). [docs/agents/](../../../docs/agents/) holds how an agent operates a piece of the repo. [README.md](../../../README.md) holds how a human runs it. A skill under `.claude/skills/` holds what an agent needs while doing one kind of work.

[docs/research/](../../../docs/research/) is the one exception to the rule that an investigation lives in its issue: a research note is a frozen, dated record written once and never amended, and the current state of anything it describes lives in one of the documents above.

The one line a research note ever gains is a dated banner under its title, naming the later note that corrects it; the corrections themselves live only in that later note.

```
> Superseded in part 2026-10-02: [camara-docket-recovery-delta-2026-09-24.md §7](camara-docket-recovery-delta-2026-09-24.md#7-corrections-this-note-makes) corrects claims below.
```

A new fact goes to the one of these a reader would open for it; when none fits, the gap is a new document, not a paragraph appended to the nearest one.

## When a document is amended

In the same fast-forward as the code that makes it stale: each slice in /implement owns the doc sections its code makes stale, and its document agent rewrites and commits them beside the code.

The change decides what is stale: a renamed or removed symbol, command, file or behaviour that a document names is a sentence to rewrite or delete, and a new behaviour a reader needs is a sentence to add where its subject already lives. A change with no such consequence amends nothing.
