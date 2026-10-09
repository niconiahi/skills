---
name: artifact
description: What an artifact is and how it sits beside prose — a small concrete rendering (code fragment, diff, side-by-side values, diagram, data sample) placed immediately after the sentence that describes it, over and over, whiteboard rhythm. Use whenever writing explains something that can be shown, and when another skill (writing, grilling, plan) needs the artifact rules.
---

# Artifact

An artifact is a concrete rendering of what the prose is talking about. The reader decides and understands in code, not in prose; a sentence that forces them to imagine the result is the failure mode, and the artifact right after it is the cure. This skill is the single home of the rules: what counts as an artifact, how big it is, and how it interleaves with the text around it.

## What counts

Anything that shows instead of describes. A code fragment, real or proposed. A diff against the real file. Example values side by side: two IRIs, two keys, two payloads, so the difference is visible at a glance. An ASCII or mermaid diagram. A data sample, a row, a log record. A short table when numbers matter.

Commands and error text are always artifacts, never inline in a sentence.

```
before   name.saij.html
after    name.settle.html    ← the suffix the scan never tries
```

An artifact shows the fragment that differs, not the whole file.

If two options are on the table, each option gets its own artifact and the artifacts line up so the eye compares them without rereading.

## Interleave

Prose and artifacts alternate in small pieces, again and again. The rhythm is a person at a whiteboard: say a little, draw a little, say a little more, draw again. Every sentence or two that describes something visualisable is immediately followed by its rendering, and the next sentence picks up after it.

```
The store today scans a fixed list.

    var DOCUMENT_ADAPTERS = []string{"saij", "bo", "infoleg", "csjn", "cij"}

The writers pass names that list never sees.

    PutDocument(judgment_id+"#settled", "settle", ...)

So a remote object under that suffix is unreachable to a read.
```

Two shapes are the failure mode. A block of prose followed by one picture at the end makes the reader hold every claim in their head until the picture finally arrives. A block of artifacts with no prose makes them guess what each one is for.

Never more than two sentences between artifacts when the subject can be shown; never two artifacts back to back without the sentence that joins them.

## Where the artifact goes

Directly under the sentence that names it, as a fenced block or an indented block, with nothing between them.

When an option or a step is being weighed, the artifact is the option: the prose says what it changes, the artifact shows it, the next prose line moves to the next option.

A recommendation or a conclusion comes after its artifacts have been seen, never before.
