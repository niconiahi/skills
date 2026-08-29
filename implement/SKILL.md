---
name: implement
description: "Implementation session for a GitHub issue labeled ready-for-agent: execute the plan comment, review, commit."
disable-model-invocation: true
---

# Implement

Takes an issue number. Requires the `ready-for-agent` label — if the issue carries `ready-for-planning` instead, stop: it has no plan yet; run /plan.

## Worktree

Work happens in a git worktree, never on `main` directly. Before starting, confirm `main` is clean; if it isn't, stop and surface it. Create a worktree branched off clean `main` and do all implementation, commits, and review there. `main` stays untouched until close-out.

Name the worktree exactly `issue-<number>` — the issue number only, no title slug (call `EnterWorktree` with `name: "issue-<number>"`). Claude Code prefixes the branch with `worktree-`, so the branch is always `worktree-issue-<number>` and the directory `.claude/worktrees/issue-<number>`.

The plan comment is the spec: read the issue description and the plan comment, then execute. Re-research only where the plan meets reality and breaks — and note the divergence in an issue comment.

Use /tdd at the seams the plan names.

Run typechecking regularly, single test files regularly, and the full test suite once at the end.

Once done, use /code-review to review the work.

Commit your work to the worktree branch.

## Close out

Merge the worktree branch into `main` directly (no PR), then push. Remove the worktree so `main` is left clean — the same clean `main` you started from, plus the merged work.

After the merge lands and is pushed: post a final comment on the issue listing every commit made for it — one line per commit, the message as a clickable link to the commit on GitHub (`https://github.com/<owner>/<repo>/commit/<sha>`) — plus any divergences from the plan. Then close the issue.
