---
name: implement
description: "Implementation session for a GitHub issue labeled ready-for-planning: the window passes messages, agents read, decide and write. One reader grounds the issue, one slicer cuts it into vertical slices ordered in waves, one implementer per slice writes and commits the code red to green in parallel, one document agent per slice commits its docs and writes its comment, one gate agent per check or gate judges its own verdict; the window relays each one-line hand-back, prints the tie questions, forwards the answers, and runs the just implement verbs that merge, land, comment and close. Every step runs once; the branch is the state, so a rerun resumes at the first unmerged slice."
---

# Implement

`/implement <n>` takes an issue from `ready-for-planning` to closed, with its code on `main`, in a session named `implement-<n>` by the `UserPromptSubmit` hook in [session-title.sh](../../hooks/session-title.sh), which the statusline shows in violet.

The issue's description is the whole scope. The session builds what it describes and nothing beside it: no step grows the work, a finding outside a slice is filed as a `needs-triage` issue and never built, and a premise the code refutes is put to the maintainer as adjust-or-stop.

Every step runs once per issue or once per slice, and a failed Merge, Check or Gates step runs once more after its repair. The only ways back are a paused or finished agent resumed with `SendMessage` and a tie question answered by the maintainer; no agent is dispatched a second time for the same work.

```
reader → slicer → per wave: implementers ∥ → document agents → merge → check → gates ∥ → land → comments → close
```

## The window passes messages

The window is skinny: it dispatches every agent, relays every hand-back in one line, prints every question file, forwards every answer with `SendMessage`, and runs the `just implement` verbs. Every reading, cut, decision, verdict and comment body is an agent's or a recipe's, and lives on disk in the handoff folder.

```
window   dispatch · relay one line · cat question file · SendMessage answer · just implement <verb>
agents   reader · slicer · implementer · evidence · general-purpose (research) · document · gate
```

Inside an implement run this file is the one skill the window reads. Every other skill is read by the agent whose work triggers it.

### The hand-back

Every agent's final message is exactly one line, and the full report lives only on disk at the path it names.

```
<agent> <target> · <DONE|PAUSED|PASS|FAIL|NOTHING TO APPLY> · <absolute path of its report or question file>
```

The window relays that line in the terminal the moment it arrives, followed by the wall time, the tokens and the tool uses the completion notification carries in its `<usage>` block. The figures are the harness's own, never estimated.

```
implementer 1a · DONE · /…/.github/issues/922/implemented-1a.md · 5m41s · 98,780 tokens · 17 tool uses
```

### Status

When an issue changes step, a wave merged, a check or gate run finished, a landing, a close, and whenever the maintainer asks, the window runs the status recipe over every issue it orchestrates, one under `/implement <n>`, every in-flight issue under "implement current wave", and prints its output as it comes.

```
just implement status <n>…
```

## Choices are graded by the agent that raises them

The reader, the slicer and the implementer grade every choice and every premise refutation they meet themselves, under the order [grilling/SKILL.md](../grilling/SKILL.md) § Weighing the options fixes, standard fit, then value, then effort, against the standards they already read.

A clear winner is decided by the agent, which appends one line to `<handoff>/decisions.md` and continues without pausing.

```
<agent> <target> · <question title> → <chosen option> · standard 🟢 <X> · value 🟢 ▰▰▰ HIGH · effort 🟡 ▰▰▱ MEDIUM
```

Two cases pause. A tie, top options equal on all three grades as [grilling/SKILL.md](../grilling/SKILL.md) § Weighing the options defines it, so options equal on the standard but different in value are decided for the higher value whatever the effort, and a premise refutation that breaks what the issue asks, so no reading of it can be built as described. The agent writes the full question, in the exact shape of [grilling/SKILL.md](../grilling/SKILL.md) § The shape of a question, to `<handoff>/question-<k>.md`, and returns PAUSED naming that file. `<k>` is the agent's target and a running number, `question-1a-2.md`, `question-slicer-1.md`, so parallel agents never collide.

A fact the agent cannot settle by reading, an empirical claim or an external-docs question, also pauses, since a subagent cannot dispatch one: the agent writes every such line to `<handoff>/needs-<target>-<j>.md` and returns PAUSED naming that file. A best fit that grades below 🟢, which grilling researches before deciding, is one of these `Needs: research` lines. Facts come before decisions, so an agent holding both pauses on its needs first and grades its choices once they return.

## Pause and resume

Agents are dispatched in the background so a pause never blocks the agents beside it, and every implementer's and document agent's id is kept until the Gates step passes, since a repair resumes them.

A PAUSED hand-back naming a question file is printed in full, and the maintainer's answer goes back to the same agent, which appends it to `decisions.md` and resumes in its own context.

```
cat <handoff>/question-<k>.md
SendMessage <agent id> "<the maintainer's answer>"
```

A PAUSED hand-back naming a needs file gets one agent per line, all at once, each prompted with the issue number, the line, the absolute slice worktree path and the absolute handoff folder.

```
Needs: <claim> | proves … | refutes … | runtime …   → subagent_type: evidence, description: "settling claim <stem>"
Needs: research <stem> | <question>                 → subagent_type: general-purpose, per research/SKILL.md, description: "researching <stem>"
```

The research prompt asks for the same one-line hand-back, naming the note it wrote. Once every one has handed back, their report paths go to the paused agent in one message.

```
SendMessage <agent id> "<report path>, <report path>, …"
```

### Stop

An answer of stop is forwarded like any other. The agent appends it to `decisions.md`, writes `<handoff>/stop.md` quoting the refuted claim and its evidence `.out` inline, and hands back DONE naming it. The run then ends without landing anything and leaves the issue clean for whoever takes it next, since a rerun would otherwise resume on the refuted reading and cut. The stop comment stays as the one record of why the run stopped, and every issue the implementers filed stays.

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

## Repair

A first failed check or gate is repaired without asking the maintainer, since the answer is always fix: the window relays the gate's line and resumes, at once, each implementer of a slice the failing report names as owner, with the report path. A merge conflict is repaired the same way: the window relays the recipe's line and resumes the later slice's implementer, telling it to replay every commit of its slice branch onto `implement-<n>` with the conflicting hunks resolved so the earlier slices' changes stay intact. A fix resumes each owning implementer with `SendMessage`, never a fresh dispatch.

```
SendMessage <implementer id> "Repair on <primary>/.claude/worktrees/implement-<n>: <report path> · fix"
```

The implementer fixes the failure on `implement-<n>` and hands back DONE naming `implemented-<id>.md`, whose Repairs section carries the new commits. The window resumes that slice's document agent with the same path, so it rechecks its owned sections and adds the commits to `slice-<id>.md`.

```
SendMessage <document id> "Repair: <handoff>/implemented-<id>.md"
```

The failed step then reruns once. A second failure of the same check or gate is the one put to the maintainer, by printing its report, with stop or a decision the maintainer writes out as the answers; the window starts no second repair on its own. A merge rerun that conflicts on the next slice is repaired the same way, slice by slice, until the wave merges.

```
fail → answer → implementer fix commit → document recheck → step reruns once → fail → stop
```

## Worktrees and handoff folder

The issue's work happens in one worktree, `implement-<n>` on branch `implement-<n>`, and `main` moves once, by a fast-forward of that branch. Each slice of a wave gets its own worktree, `implement-<n>-<id>` on branch `implement-<n>-<id>`, cut from `implement-<n>` when its wave starts.

The session stays parked in the primary checkout, the first path in `git worktree list`; every command aimed at a worktree carries its path, written out literally since a hook rejects shell variable assignments.

```
git -C <primary>/.claude/worktrees/implement-<n> …
```

`EnterWorktree` and a plain `cd` into a worktree are both banned: the first fences every subagent to that worktree, the second persists across Bash calls and turns the worktree into the primary directory.

The handoff folder is the primary checkout's gitignored `.github/issues/<n>/`. It outlives the worktrees, so a rerun finds every report there.

```
<primary>/.github/issues/<n>/
  reading.md              reader        the reading
  ground.md               reader        the ground the slicer cuts on
  overview.md             reader        the overview comment, without its Decisions
  decisions.md            agents        one decision per line, decided or answered
  question-<k>.md         agents        one tie or premise question, in grilling's shape
  needs-<target>-<j>.md   agents        the Needs lines of one pause
  slices.md               slicer        the cut: ids, waves, After, ownership, commit subjects, tests, reds
  brief-<id>.md           slicer        one per slice
  implemented-<id>.md     implementer   tests, research, evidence, refutations, repairs of one slice
  evidence-<slug>.md      evidence      the verdict on one claim
  slice-<id>.md           document      the slice comment, commit subjects not yet linked
  gate-<name>.md          gate          the verdict of one gate or wave check, beside its logs
  stop.md                 agents        the stop comment, only on a stop
```

## Steps

The steps run in the order of the headings below. A rerun walks the same steps and skips each one whose completion already holds.

### Gate and open

The issue must carry `ready-for-planning`, have no open native blocker, and be either unclaimed, no assignee and Status not In progress on the board, or claimed by `@me` alone, as `just board verify-claim` leaves a wave it fired; otherwise the session names what fails and ends.

```
gh issue view <n> --json labels,state,assignees
gh api repos/niconiahi/machina-iuris/issues/<n> --jq .issue_dependencies_summary.blocked_by      0
```

The session's first write is the claim, per [issue-tracker.md](../../../docs/agents/issue-tracker.md) § The board: the issue is assigned to the maintainer and its Status set to In progress, so no other session offers or takes it. On an issue a wave already claimed both writes are no-ops.

```
gh issue edit <n> --add-assignee @me
Status → In progress      the board's item-edit, per issue-tracker.md § The board
```

Local `main` is reconciled with origin once, so the worktree is cut from the true tip. A refused fast-forward means local `main` diverged from origin, which is put to the maintainer.

```
git fetch origin
git merge --ff-only origin/main
```

A worktree a previous run left is reused as it is, since its commits are that run's merged slices, and any uncommitted change in it is stashed, never discarded. A missing worktree is opened from `main`, and its `post-checkout` hook runs `just setup`.

```
present   git -C <primary>/.claude/worktrees/implement-<n> stash push --include-untracked -m "implement-<n> rerun"   # only if dirty
absent    git worktree add <primary>/.claude/worktrees/implement-<n> -b implement-<n> main
```

Completion is the worktree present on `implement-<n>` with a clean status.

### Read

Skipped when `reading.md`, `ground.md` and `overview.md` exist.

Dispatch one reader agent, [reader.md](../../../.claude/agents/reader.md), with the issue number and the absolute handoff folder. It reads the issue once, checks every assertion in it against `main`, decides every load-bearing false verdict it can, writes its three files, and hands back DONE naming `reading.md`.

```
subagent_type: reader
description: "reading issue"
```

Completion is a DONE hand-back from the reader.

### Slice

Skipped when `slices.md` and a `brief-<id>.md` for every slice in it exist.

Dispatch one slicer agent, [slicer.md](../../../.claude/agents/slicer.md), with the issue number, the absolute handoff folder and the absolute `implement-<n>` worktree path. It cuts the work into vertical slices, decides every choice it can, writes `slices.md` and one brief per slice, and hands back DONE naming `slices.md`.

```
subagent_type: slicer
description: "slicing issue"
```

Completion is a DONE hand-back from the slicer.

### Waves

The waves run in order, and inside a wave every slice runs in parallel. The slice ids and waves are the ones `just implement status <n>` prints. A slice whose last code commit subject, from `slices.md`, is already on `implement-<n>` is done and skipped, and a wave whose slices are all done is skipped.

#### Open the slice worktrees

Each slice not done gets its worktree from the current `implement-<n>`. A slice worktree a previous run left is kept: when its branch carries the slice's last code commit its implementer is not dispatched again, and when it carries only some of its subjects its implementer resumes at the first subject the branch lacks.

```
git worktree add <primary>/.claude/worktrees/implement-<n>-<id> -b implement-<n>-<id> implement-<n>
```

#### Implement

Dispatch one implementer agent, [implementer.md](../../../.claude/agents/implementer.md), per slice, all at once, with the issue number, the slice id, the absolute handoff folder and the absolute slice worktree path. It writes the slice red to green, one commit per subject of the brief, in order, writes `implemented-<id>.md`, and hands back DONE naming it.

```
subagent_type: implementer
description: "implementing slice <id>"
```

Each hand-back is relayed the moment it arrives, while the others keep running, and a PAUSED one is routed per Pause and resume.

Completion is a DONE hand-back from every implementer of the wave.

#### Document

Dispatch one document agent, [document.md](../../../.claude/agents/document.md), per slice the moment its implementer hands back DONE, with the issue number, the slice id, the absolute handoff folder and the absolute slice worktree path. It commits the docs the slice's code makes true, writes `slice-<id>.md`, and hands back DONE or NOTHING TO APPLY naming it.

```
subagent_type: document
description: "documenting slice <id>"
```

Completion is a hand-back from every document agent of the wave.

#### Merge

The recipe cherry-picks the wave's slice branches onto `implement-<n>` in id order, so history stays linear and every commit keeps its subject, then removes the slice worktrees and branches, and prints one line.

```
just implement merge <n> <wave>
```

A conflict means two slices of the wave touched the same lines, a seam the slicer cut wrong: the recipe aborts the cherry-pick and its line names both slices and the file. It is repaired per Repair without asking the maintainer, by the later slice's implementer, and the merge reruns.

#### Check

Dispatch one gate agent, [gate.md](../../../.claude/agents/gate.md), for the wave, with the wave number, the absolute `implement-<n>` worktree path, the absolute primary checkout path and the absolute handoff folder. It runs the build and every merged slice's narrow tests from `slices.md`, judges its own verdict, and hands back PASS or FAIL naming `gate-wave-<w>.md`.

```
subagent_type: gate
description: "checking wave <w>"
```

A FAIL is repaired per Repair.

Completion is every wave merged and checked, and no slice worktree left.

### Gates

Dispatch four gate agents at once, one per gate, format, build, vet and test, each with the gate's name, the absolute `implement-<n>` worktree path, the absolute primary checkout path and the absolute handoff folder. Each runs its gate at branch scope, judges its own verdict, and hands back PASS or FAIL naming `gate-<name>.md`.

Every implementer already ran the same four gates at slice scope over its own commits before it handed back, so a gate failure here is one no single slice's diff showed.

```
subagent_type: gate
description: "running gate <name>"      format · build · vet · test
```

A FAIL is repaired per Repair, after which every gate reruns once.

Completion is every gate PASS.

### Land

The recipe fast-forwards `main` onto `implement-<n>` from the primary checkout, pushes, and prints one line; it refuses when `main` diverged.

```
just implement land <n>
```

A refusal means `main` moved during the run, and the answer is always rebase, so it is never put to the maintainer. The window rebases `implement-<n>` onto `main` in its worktree, reruns the Gates step, and lands again; a later refusal rebases again.

```
git -C <primary>/.claude/worktrees/implement-<n> rebase main
```

A conflicting rebase is aborted and repaired like a merge conflict, without asking: the window resumes the implementer of the slice that owns the conflicting file, telling it to rebase `implement-<n>` onto `main` with the conflicting hunks resolved so both sides' changes stay intact, then reruns the Gates step.

```
git -C <primary>/.claude/worktrees/implement-<n> rebase --abort
SendMessage <implementer id> "Rebase <primary>/.claude/worktrees/implement-<n> onto main: conflict in <files> · keep both sides"
```

Completion is `main` on origin carrying every slice's commits.

### Post the comments

The recipe composes the overview from `overview.md` and `decisions.md`, and each slice comment from `slice-<id>.md` with its commit subjects linked to their commits on `main`, upserts them by marker, overview first and slices in id order, and prints one line.

```
just implement comments <n>
```

Completion is the issue carrying exactly one comment per composed body and no other plan marker.

### Close

The recipe closes the issue with a comment naming the landed range, removes `ready-for-planning`, prunes the worktree, the branch, their test databases and the handoff folder, runs the self-check, and prints one line, so a run that died halfway is visible as such to the next. A prune refused because paradedb is down is put to the maintainer.

```
just implement close <n>
```

## Slices and waves

A run is one wave. A slice is one connected group of the issue's work, vertical and true on its own: pieces where one consumes a symbol or signature the other introduces or changes, or where one's tests read a line of a single-point file the other writes, are one slice, and pieces that share nothing are parallel slices. The slicer fixes every slice's ownership, the files and top-level declarations it touches and the doc sections it rewrites, and no two slices own the same one.

A single-point file is owned by exactly one slice, which writes every line of it the issue needs.

```
go.mod
a justfile or config/*.just
a numbered migration
a registry or table both append to
the same golden
```

A slice carries 8 or fewer behavioural criteria, the issue's acceptance criteria without the generic `just test` gate; a larger group is a split at triage, per [triage-labels.md](../../../docs/agents/triage-labels.md#size-labels). Its code lands as an ordered series of commits, one per criterion, preceded by one enabling commit when the criteria share a change none of them names, and its last subject is its done key.

The slice id carries the wave, `1a`, `1b`, `1c`, so ordering by id is ordering by wave.

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
