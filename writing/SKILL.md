---
name: writing
description: How every piece of text is written — a reply, a skill, a doc, an issue comment, a commit body, a plan. One shape for all of them: a heading hierarchy that carries the structure, prose paragraphs that carry the meaning, artifacts interleaved with the prose in small pieces.
---

# Writing

Everything written here has the same shape, whatever it is for. The structure lives in headings, the meaning lives in paragraphs of prose under them, and anything visualisable is shown as an artifact right where the prose mentions it. This holds for the answer to a prompt, a skill file, a document under `docs/`, an issue comment, a plan chunk and a research note alike. A reader should be able to tell what is important from the heading depth, read top to bottom without reassembling anything, and never have to imagine a result that could have been shown.

## Headings carry the hierarchy

Headings are the outline of the explanation, and their depth is the importance of the thing they name. A `#` is the one top-level title. A `##` is a major group. A `###` is a child of the `##` above it, less important on its own but placed there because it belongs to that parent. A `####` is a minor group that still earns a heading because leaving it unmarked would flatten a hierarchy the reader needs to see. Depth means membership: a `###` is always inside the nearest `##` above it, never a free-floating section that happens to be smaller.

```
# Top level title

Introductory paragraph: the high-level explanation.

## Important group

Paragraph about this group.

### Child of the group above, less important

Paragraph about this child.

#### Minor group, marked only to show where it sits

Paragraph about this minor group.

## Another important group

Paragraph about this group.
```

Use as many levels as the explanation has, and no more. A document with one idea has one heading. A document that explains a system with parts and sub-parts has the depth the system has.

Skipping a level (a `####` directly under a `##`) breaks the membership reading and is never done.

## Paragraphs carry the meaning

Under each heading sits prose: connected sentences where the reasoning lives inside the sentences, not chopped into bullets.

The title introduces with a paragraph that gives the high-level explanation before any subgroup starts, so the reader knows the shape of what follows. Each subsequent heading gets its own paragraph or paragraphs that say what that group is and why it matters, with the connective tissue between facts kept in the text.

Bullets are for genuinely enumerable, order-free items: files touched, commands to run, a handful of options to pick from. They are never the skeleton of an analysis. A list of bold lead-ins each followed by a clause forces a context switch per item and loses the reasoning that would have joined them; the same content as a paragraph reads top to bottom.

Length follows the content. A question with a one-sentence answer gets one sentence, no heading. A staff engineer answering a peer says the thing that matters and stops; the hierarchy above is for when there genuinely is a hierarchy to show.

## Artifacts interleave with the prose

Whenever a paragraph describes something that can be shown, the rendering follows it immediately and the next paragraph picks up after it: say a little, draw a little, again and again. Everything written here is dense with these, because a reader understands a fragment of code, a diff or two values side by side faster than any sentence about them.

What counts as an artifact, how small it is, and the interleave rhythm are fixed in one place, [artifact/SKILL.md](../artifact/SKILL.md); read it with this skill.

```
The store today scans a fixed list.

    var DOCUMENT_ADAPTERS = []string{"saij", "bo", "infoleg", "csjn", "cij"}

The writers pass a name that list never sees.

    PutDocument(judgment_id+"#settled", "settle", ...)
```

Code, commands and error text always go in a fenced block, never inline in a sentence.

Numbers that change what the reader does go on their own line or in a short table, not buried in prose.

## Whatever can be a link is a link

A file, a skill, an issue, a commit or a doc named in prose is a link the reader can follow, never a bare path in backticks. The link text is the name as the reader knows it and the target is relative to the file being written, so it resolves on GitHub and in an editor alike.

```
bare    read `README.md` and `.claude/skills/git/SKILL.md`
linked  read [README.md](./README.md) and [git/SKILL.md](./.claude/skills/git/SKILL.md)
```

An issue is `[#936](https://github.com/niconiahi/machina-iuris/issues/936)`, a commit its message linked to the commit URL.

Backticks stay for identifiers that are not files: a symbol, a flag, a value.

## Close with what is needed

Every reply to the maintainer ends with a `## What I need from you` section, the last thing in the message, so they never have to ask what is wanted of them. It numbers each thing needed in one line, followed by the exact replies to choose from, one per line, or the exact command in a fenced block, written as `! <command>` so it runs in the session.

````
## What I need from you

1. Pick the commit layout:

B
A

2. Log in to GitHub, run in this prompt:

```
! gh auth login
```
````

When nothing is needed, the section says so in one line with what runs next.

```
## What I need from you

Nothing. Running the gates now.
```

## Where this applies

This skill is the shape; other skills add their own constraints on top of it. The artifact skill fixes what an artifact is and how it interleaves. The grilling skill fixes the order of prose and artifact inside a question. The plan skill fixes the marker lines and the code-first content of plan comments. The documentation and ADR skills fix what a doc or a decision record must contain. None of them changes the shape: headings for hierarchy, prose for meaning, artifacts interleaved.
