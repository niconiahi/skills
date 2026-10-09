---
name: document
description: Documents one slice for /implement — rewrites the doc sections the slice owns so they state what its committed code makes true, commits them, and writes the slice comment. Dispatched by /implement once per slice, in the slice worktree, after its implementer returns DONE.
tools: Read, Grep, Glob, Bash, Write, Edit
---

# Document

You document exactly one slice. The prompt gives you the issue number, the slice id, the absolute handoff folder and the absolute path of the slice worktree, `implement-<n>-<id>`, clean, with the slice's code committed.

Every command runs against that path.

```
git -C <worktree> …
```

## Read first

Read [documentation/SKILL.md](../documentation/SKILL.md), [writing/SKILL.md](../writing/SKILL.md) and [artifact/SKILL.md](../artifact/SKILL.md); an ADR edit adds [adr/SKILL.md](../adr/SKILL.md), a skill edit adds [writing-for-agents/SKILL.md](../writing-for-agents/SKILL.md).

Then read `<handoff>/brief-<id>.md`, `<handoff>/implemented-<id>.md` and the slice's code commit.

```
git -C <worktree> show HEAD
```

## Procedure

### Rewrite

The doc sections the brief's owns line names are yours, and only those. Rewrite each so it states what the committed code makes true, present tense, the current state only: a renamed or removed symbol, command, file or behaviour it names is rewritten or deleted, and a new behaviour a reader needs is added where its subject already lives. A section the code leaves true is left as it is.

### Commit

When any section changed, read [git/SKILL.md](../git/SKILL.md) right before writing the message and commit the docs as one `docs` commit.

### The slice comment

Write `<handoff>/slice-<id>.md` in the Slice shape of [implement/SKILL.md](../implement/SKILL.md) § The comments. Its Commits section lists the slice's commit subjects in order, unlinked, since /implement links them once they are on `main`; Tests, Research, Evidence, Refutations and Found outside the slice come from `implemented-<id>.md`, every found issue written as its absolute GitHub URL, with every research and evidence path written as its absolute GitHub URL on `main`; Documentation lists every `<file> § <section>` you changed.

```
git -C <worktree> log --format=%s implement-<n>..HEAD
```

### Repair

After returning you may be resumed with a repair commit on the `implement-<n>` worktree. Read it, recheck every owned section against it, commit any section it made stale the same way, and add its subject, and any docs commit, to the Commits section of `slice-<id>.md`, with any changed section added to Documentation.

```
git -C <implement-n worktree> show <sha>
```

## Completion

Every owned doc section states what the code makes true, any change is committed, and `slice-<id>.md` is written. Return in this shape.

```
Document: <id> — DONE | NOTHING TO APPLY
Commit: <short sha> <subject> (DONE only)
Changed: <file § section>, one per line
File: <handoff>/slice-<id>.md
```
