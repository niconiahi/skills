---
name: implement
description: "Implementation session for a GitHub issue labeled ready-for-planning: the window orchestrates, agents read and write. One reader grounds the issue, one slicer cuts it into vertical slices ordered in waves, one implementer per slice writes and commits the code red to green in parallel, one document agent per slice commits its docs and writes its comment; the window merges each wave, runs the gates, fast-forwards main, posts the overview and slice comments and closes the issue. Every step runs once; the branch is the state, so a rerun resumes at the first unmerged slice."
disable-model-invocation: true
---

# Implement

`/implement <n>` takes an issue from `ready-for-planning` to closed, with its code on `main`, in a session named `implement-<n>` by the `UserPromptSubmit` hook in [session-title.sh](../../hooks/session-title.sh), which the statusline shows in violet.

The issue's description is the whole scope. The session builds what it describes and nothing beside it: no step grows the work, a finding outside a slice is filed as a `needs-triage` issue and never built, and a premise the code refutes is put to the maintainer as adjust-or-stop.

Every step runs once per issue or once per slice, and a failed Merge, Check or Gates step runs once more after its repair. The only ways back are a paused or finished agent resumed with `SendMessage` and a grilling question to the maintainer; no agent is dispatched a second time for the same work.

```
reader → ❓ → slicer (cut) → ❓ → slicer (briefs) → per wave: implementers ∥ → document agents → merge → narrow checks → gates → main → comments → close
```

## The window is an orchestrator

The window dispatches agents, routes the summaries they return, asks grilling questions, appends each confirmed decision to `decisions.md`, runs git and gh, runs the checks and gates, and composes the comment bodies it posts.

```
window   dispatch · route · grill · append decision · git/gh · checks and gates · compose comments
agents   reader · slicer · implementer · evidence · general-purpose (research) · document
```

Every agent return is said aloud, in the terminal, before the window routes it: one line naming the agent and its target, then the wall time, the tokens and the tool uses the completion notification carries in its `<usage>` block. The figures are the harness's own, never estimated, and the line is printed even when the return is a STOP.

```
implementer 1a finished · 5m41s · 98,780 tokens · 17 tool uses
```

## Read before the first tool call

Three skills govern the window's own steps and are read at the start of the run.

```
grilling   every decision put to the maintainer     → grilling/SKILL.md
just       the check and gate commands             → just/SKILL.md
git        every commit the window makes or names   → git/SKILL.md
```

Once the reading names a profiled standard, akn, skos, otel or retrieval-telemetry, that skill and its `STANDARD.md` are read in full before the first question, per the Standard fit section of [grilling/SKILL.md](../grilling/SKILL.md).

## Pause and resume

A slicer or an implementer that meets something it cannot settle alone returns PAUSED with the lines the window routes, and stops. The window settles each line, a grilling question or an evidence or research dispatch, then sends the answers to the same agent with `SendMessage`, and the agent resumes in its own context.

```
PAUSED   Choices | Needs | Premise   → ❓ or evidence/research → SendMessage <agent> "<answers>" → resumes
```

Agents are dispatched in the background so a pause never blocks the agents beside it, and every implementer's and document agent's id is kept until the Gates step passes, since a repair resumes them.

## Repair

A failed merge, check or gate is one grilling question naming the failure and the slices that own its files. When the answer is to fix, the window resumes each owning implementer with `SendMessage`, never a fresh dispatch, with the failing lines, the answer and the `implement-<n>` worktree path.

```
SendMessage implementer-<id> "Repair on <primary>/.claude/worktrees/implement-<n>: <failing lines> · <answer>"
```

The implementer fixes the failure on `implement-<n>`, reruns the failing command, and commits one `fix(<scope>): …` line, or, for a merge conflict, cherry-picks its own slice onto `implement-<n>` with the conflict resolved, keeping its subject. Then the window resumes that slice's document agent with the commit, so it rechecks its owned sections and adds the commit to `slice-<id>.md`.

```
SendMessage document-<id> "Repair commit <sha> <subject> on <primary>/.claude/worktrees/implement-<n>"
```

The failed step then reruns once. A second failure is a grilling question whose only answers are stop or a decision the maintainer writes out; it never starts a second repair on its own.

```
fail → ❓ → implementer fix commit → document recheck → step reruns once → fail → ❓ stop
```

## Worktrees and handoff folder

The issue's work happens in one worktree, `implement-<n>` on branch `implement-<n>`, and `main` moves once, by a fast-forward of that branch. Each slice of a wave gets its own worktree, `implement-<n>-<id>` on branch `implement-<n>-<id>`, cut from `implement-<n>` when its wave starts.

The session stays parked in the primary checkout, the first path in `git worktree list`; every command aimed at a worktree carries its path, written out literally since a hook rejects shell variable assignments, and `just` takes both flags because it refuses `--working-directory` without `--justfile`.

```
git -C <primary>/.claude/worktrees/implement-<n> …
just --justfile <wt>/justfile --working-directory <wt> …
```

`EnterWorktree` and a plain `cd` into a worktree are both banned: the first fences every subagent to that worktree, the second persists across Bash calls and turns the worktree into the primary directory.

The handoff folder is the primary checkout's gitignored `.github/issues/<n>/`. It outlives the worktrees, so a rerun finds every report there.

```
<primary>/.github/issues/<n>/
  reading.md            reader        the reading
  ground.md             reader        the ground the slicer cuts on
  overview.md           reader        the overview comment, without its Decisions
  decisions.md          window        one confirmed decision per line
  slices.md             slicer        the cut: ids, waves, After, ownership, commit subjects, tests
  brief-<id>.md         slicer        one per slice
  implemented-<id>.md   implementer   tests, research, evidence, refutations of one slice
  slice-<id>.md         document      the slice comment, commit subjects not yet linked
  gate-<name>.log       window        one per check or gate run
```

## Steps

The steps run in the order of the headings below. A rerun walks the same steps and skips each one whose completion already holds.

### Gate and open

The issue must carry `ready-for-planning`, have no open native blocker, and be unclaimed, no assignee and Status not In progress on the board; otherwise the session names what fails and ends.

```
gh issue view <n> --json labels,state,assignees
gh api repos/niconiahi/machina-iuris/issues/<n> --jq .issue_dependencies_summary.blocked_by      0
```

The session's first write is the claim, per [issue-tracker.md](../../../docs/agents/issue-tracker.md) § The board: the issue is assigned to the maintainer and its Status set to In progress, so no other session offers or takes it.

```
gh issue edit <n> --add-assignee @me
Status → In progress      the board's item-edit, per issue-tracker.md § The board
```

Local `main` is reconciled with origin once, so the worktree is cut from the true tip. A refused fast-forward means local `main` diverged from origin, which is a grilling question.

```
git fetch origin
git merge --ff-only origin/main        # refused → ❓ local main diverged from origin
```

A worktree a previous run left is reused as it is, since its commits are that run's merged slices, and any uncommitted change in it is stashed, never discarded. A missing worktree is opened from `main`, and its `post-checkout` hook runs `just setup`.

```
present   git -C <primary>/.claude/worktrees/implement-<n> stash push --include-untracked -m "implement-<n> rerun"   # only if dirty
absent    git worktree add <primary>/.claude/worktrees/implement-<n> -b implement-<n> main
```

Completion is the worktree present on `implement-<n>` with a clean status.

### Read

Skipped when `reading.md`, `ground.md` and `overview.md` exist.

Dispatch one reader agent, [reader.md](../.agents/reader.md), with the issue number and the absolute handoff folder. It reads the issue once, checks every assertion in it against `main`, writes its three files, and returns the reading and one line per load-bearing false verdict.

```
subagent_type: reader
description: "reading issue"
```

The reading is said out loud, verbatim from the summary. Each load-bearing line is a grilling question, the premise was wrong, what does the issue want, and its answer is appended to `decisions.md`.

Completion is the reading said and every load-bearing line answered in `decisions.md`.

### Slice

Skipped when `slices.md` and a `brief-<id>.md` for every slice in it exist.

Dispatch one slicer agent, [slicer.md](../.agents/slicer.md), with the issue number, the absolute handoff folder and the absolute `implement-<n>` worktree path. It cuts the work into vertical slices, writes `slices.md`, and returns PAUSED with the cut and every open choice.

```
subagent_type: slicer
description: "slicing issue"
```

The cut is said out loud. Each `Choices` line is a grilling question, its Standard subsections filled from the line's fit, and its answer is appended to `decisions.md`. Then the slicer is resumed once with every answer, writes one brief per slice, and returns DONE.

```
SendMessage slicer "Decisions: <line>, <line>, …  Write the briefs."
```

A slicer with no open choice still pauses on the cut, and is resumed with "No decisions. Write the briefs."

Completion is `slices.md` and every brief on disk, from a DONE return.

### Waves

The waves run in order, and inside a wave every slice runs in parallel. A slice whose code commit subject, from `slices.md`, is already on `implement-<n>` is done and skipped, and a wave whose slices are all done is skipped.

```
git -C <primary>/.claude/worktrees/implement-<n> log --format=%s main..implement-<n>
```

#### Open the slice worktrees

Each slice not done gets its worktree from the current `implement-<n>`. A slice worktree a previous run left whose branch already carries the slice's code commit is kept, and its implementer is not dispatched again.

```
git worktree add <primary>/.claude/worktrees/implement-<n>-<id> -b implement-<n>-<id> implement-<n>
```

#### Implement

Dispatch one implementer agent, [implementer.md](../.agents/implementer.md), per slice, all at once, with the issue number, the slice id, the absolute handoff folder and the absolute slice worktree path. It writes the slice red to green with tdd, commits it under the brief's commit subject, writes `implemented-<id>.md`, and returns DONE.

```
subagent_type: implementer
description: "implementing slice <id>"
```

Each return is routed the moment it arrives, while the others keep running. A PAUSED return carries three kinds of line.

```
Needs     an empirical claim   → evidence agent, prompt: issue, the four fields, the slice worktree path
          an external-docs one → general-purpose agent per research/SKILL.md, prompt: the slice worktree path
Choices   a design choice      → ❓ → decisions.md
Premise   a refutation that breaks what the issue asks → ❓ adjust within the issue | stop
```

```
subagent_type: evidence
description: "settling claim <evidence file stem>"
subagent_type: general-purpose
description: "researching <note file stem>"
```

The verdicts and answers go back to the implementer with `SendMessage` in one message, and it resumes. A refutation that stays inside the slice never reaches the window: the implementer absorbs it and records it.

An answer of stop ends the run without landing anything, and leaves the issue clean for whoever takes it next: the issue's `ready-for-planning` is swapped for `needs-triage`, one comment quotes the refuted claim and its evidence `.out` inline, the claim is released, and every worktree, every `implement-<n>*` branch and the handoff folder are deleted, since a rerun would otherwise resume on the refuted reading and cut. The stop comment stays as the one record of why the run stopped, and every issue the implementers filed stays.

```
gh issue edit <n> --remove-label ready-for-planning --add-label needs-triage
gh issue comment <n> --body-file <primary>/.github/issues/<n>/stop.md
gh issue edit <n> --remove-assignee @me
Status → Todo             the board's item-edit, per issue-tracker.md § The board
git worktree remove --force <primary>/.claude/worktrees/implement-<n>-<id>      every slice worktree
git worktree remove --force <primary>/.claude/worktrees/implement-<n>
git branch -D implement-<n> implement-<n>-<id> …
rm -rf <primary>/.github/issues/<n>
```

Completion is a DONE return from every implementer of the wave.

#### Document

Dispatch one document agent, [document.md](../.agents/document.md), per slice the moment its implementer returns DONE, with the issue number, the slice id, the absolute handoff folder and the absolute slice worktree path. It commits the docs the slice's code makes true, writes `slice-<id>.md`, and returns DONE or NOTHING TO APPLY.

```
subagent_type: document
description: "documenting slice <id>"
```

Completion is a return from every document agent of the wave and a `slice-<id>.md` for every slice.

#### Merge

The wave's slices are cherry-picked onto `implement-<n>` in id order, so history stays linear and every commit keeps its subject; then the slice worktrees and branches are removed.

```
git -C <primary>/.claude/worktrees/implement-<n> cherry-pick implement-<n>..implement-<n>-<id>
git worktree remove <primary>/.claude/worktrees/implement-<n>-<id>
git branch -D implement-<n>-<id>
```

A conflict means two slices of the wave touched the same lines, a seam the slicer cut wrong: the cherry-pick is aborted and it is a grilling question naming both slices and the file, answered per Repair by the later slice's implementer.

```
git -C <primary>/.claude/worktrees/implement-<n> cherry-pick --abort    → ❓ → Repair
```

#### Check

The window runs the build and every merged slice's narrow test commands, from `slices.md`, on `implement-<n>`, each writing its full output to its log while the window reads only the exit code and the last forty lines. A failure is one grilling question naming the test and the slices whose files it touches, answered per Repair; it is never a redispatch.

```
just --justfile <wt>/justfile --working-directory <wt> build > <handoff>/gate-build-<wave>.log 2>&1; echo "exit $?"
just --justfile <wt>/justfile --working-directory <wt> test "<TestName>" > <handoff>/gate-<id>.log 2>&1; echo "exit $?"; tail -n 40 <handoff>/gate-<id>.log
```

A test command whose test lives in a `//go:build integration` file carries the tag, `test "-tags integration ./<package>/ -run <TestName>"`; without it the file is compiled out and the check passes on `[no tests to run]`, which counts as a failure.

Completion is every wave merged and checked, and no slice worktree left.

### Gates

Every gate runs once, all in parallel, as background Bash calls on `implement-<n>`, each writing its full output to its log while the window reads only the exit code and the last forty lines.

| Gate | Command | Pass when |
|------|---------|-----------|
| format | `git -C <wt> diff --name-only --diff-filter=AM main...HEAD -- '*.go' ':!evidence' \| sed 's\|^\|<wt>/\|' \| xargs -r go tool golines --tab-len=2 -l` | output empty |
| build | `just … build` | exit 0 |
| vet | `just … vet` | exit 0 |
| test | `just … test -tags integration ./...`, Bash `timeout` at `600000` | exit 0, or the failing tests are exactly the expected reds |

An expected red is a pending case the issue exists to capture, listed in its slice's line in `slices.md`; a listed expected red that passes, or any other failing test beside it, fails.

```
grep -E -- '--- FAIL' <handoff>/gate-test.log
```

A failure is a grilling question naming the gate, the failing lines and the slices whose files they touch, answered per Repair, after which every gate reruns once.

Completion is every gate PASS.

### Land

From the primary checkout, `main` fast-forwards onto `implement-<n>` and is pushed. A refused fast-forward or push means `main` moved during the run, which is a grilling question, never a loop.

```
git merge --ff-only implement-<n>
git push
refused → ❓ main moved: rebase implement-<n> onto main and rerun the Gates step | stop
```

Completion is `main` on origin carrying every slice's commits.

### Post the comments

The window composes the posted bodies. The overview is `overview.md` with `decisions.md` appended under its `### Decisions` heading. Each slice comment is `slice-<id>.md` with every commit subject in its Commits section linked to its commit on `main`.

```
git log --format='%H %s' <first slice commit>~1..main
- feat(akn): resolve decree iri from census key   →   - [feat(akn): resolve decree iri from census key](https://github.com/niconiahi/machina-iuris/commit/<sha>)
```

The bodies are upserted by marker, the overview first and the slices in id order, so the issue reads in the order the work ran: a comment whose marker exists is edited in place, a new marker is posted, and a plan marker on the issue that no body carries is deleted.

```
gh api repos/niconiahi/machina-iuris/issues/<n>/comments --paginate --jq '.[] | {id, marker: (.body | capture("^<!-- (?<m>plan:[^ ]+) -->").m)}'
marker exists   → gh api -X PATCH repos/niconiahi/machina-iuris/issues/comments/<id> -F body=@<path>
marker new      → gh issue comment <n> --body-file <path>
marker gone     → gh api -X DELETE repos/niconiahi/machina-iuris/issues/comments/<id>
```

Every issue a slice filed or commented on, listed in its Found outside the slice section, gets its `slice <id>` linked to that slice's comment, now that the comment exists.

```
Found while implementing [#<n>](…/issues/<n>) · slice <id>   →   Found while implementing [#<n>](…/issues/<n>) · [slice <id>](…/issues/<n>#issuecomment-<id>)
gh api -X PATCH repos/niconiahi/machina-iuris/issues/<found>            -F body=@<edited body>       a filed issue
gh api -X PATCH repos/niconiahi/machina-iuris/issues/comments/<id>      -F body=@<edited body>       an "Also found" comment
```

Completion is the issue carrying exactly one comment per composed body and no other plan marker, and every found issue's slice linked.

### Close

The issue closes with a comment naming the landed range and its `ready-for-planning` label is removed. Then `just worktree prune`, from the primary checkout, removes the worktree, the branch, their test databases and the handoff folder, since the issue is closed; a prune refused because paradedb is down is a grilling question.

```
gh issue close <n> --comment "Landed <first short sha>..<last short sha>"
gh issue edit <n> --remove-label ready-for-planning
just worktree prune
```

The last action is the self-check, printed, so a run that died halfway is visible as such to the next.

```
gh issue view <n> --json state,labels
git worktree list | grep implement-<n>     # empty
git branch --list 'implement-<n>*'         # empty
```

## Slices and waves

A slice is a vertical cut through the issue: one behaviour, end to end, with its own test, committed on its own and true on its own. The slicer fixes every slice's ownership, the files and top-level declarations it touches and the doc sections it rewrites, and two slices in one wave never own the same one.

Two slices go in different waves, the later one listing the earlier in its `After` line, when one consumes a symbol or signature the other introduces or changes, or when both need a single-point file. A single-point file is owned by exactly one slice.

```
go.mod
a justfile or config/*.just
a numbered migration
a registry or table both append to
the same golden
```

The slice id carries the wave, `1a`, `1b`, `2a`, so ordering by id is ordering by wave.

## The comments

The issue carries one overview comment and one comment per slice, each opening with its marker line. The top heading of every comment is `##`, because the issue title is the `#`, and every evidence or research link is the absolute GitHub URL of the file on `main`.

```
https://github.com/niconiahi/machina-iuris/blob/main/evidence/<n>/<file>
```

### Overview

`<!-- plan:overview -->` carries the reading, the reader's verdict table and the decisions.

```
<!-- plan:overview -->
## Plan: <issue title>
### Builds / Why / Practical / Gates / Blockers    the reading, one ### each
### Verdicts      | statement | verdict | ground |
### Decisions     decisions.md, one per line
```

### Slice

`<!-- plan:slice:<id> -->`, one per slice, is the record of what the slice did and why; the code lives behind its commit links.

```
<!-- plan:slice:<id> -->
## Slice <id> · wave <w>: <goal>
After: <ids> | none
### Commits         one linked commit subject per line
### Tests           the tests that went red then green, one per line
### Research        absolute link per note, or none
### Evidence        absolute link per file, or none
### Documentation   <file> § <section>, one per line, or none
### Refutations     <claim> → <what changed in the slice>, one per line, or none
### Found outside the slice   one linked issue per line, filed or commented on, or none
```
