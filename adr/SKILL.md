# ADR

How a decision record in [docs/adr/](../../../docs/adr/) is written and kept. Read before creating or editing any file there, together with [documentation/SKILL.md](../documentation/SKILL.md), which owns the rules every document shares: present tense, the current state only, each fact in one place, written per the writing skill.

## One record answers one question

An ADR states what one decision is, right now. Its subject is atomic: a single choice a reader can name in one line, such as where persistence lives or what a unit is.

A new decision that extends an existing record's subject is a rewrite of that record; a new number exists only for a new subject.

```
extends 0012   postgres also holds telemetry        → rewrite 0012
new subject    the embedder is reached over http    → 0005
```

Two small records about one subject become one, and deleting a record repoints every `ADR-NNNN` reference in the repo in the same change, leaving no stub behind.

The investigation that produced the decision, its sampling, its numbers and its verification trail, lives in the issue that did the work; the record states the conclusion and links the issue.

## Front matter

Two lines only: `Status:` and `Date:`, the date of the last rewrite.

A decision replaced wholesale carries `Status: Superseded by NNNN`; every other change is a rewrite in place under `Status: Accepted`.

## Skeleton

Three headings, prose paragraphs under each, artifacts interleaved where a fragment shows the decision faster than a sentence about it.

Subsections of the decision are `###` headings, never bold-led bullets.

```markdown
# NNNN — Title

Status: Accepted
Date: YYYY-MM-DD

## Context

Why the decision exists: the minimum a reader needs to understand it, motivation
rather than history.

## Decision

What holds today, present tense, with `###` subsections when the decision has parts.

## Consequences

What follows from the decision, including the tests or gates that enforce it.
```
