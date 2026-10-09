---
name: git
description: Git commit conventions. Use always when creating commits — covers message format, scope usage, and type selection.
---

# Git Commit Conventions

Follow conventional commits. Every commit message follows: `type(scope): description`

## Types

| Type | When |
|------|------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `chore` | Maintenance, config, tooling, dependencies |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `test` | Adding or updating tests |
| `ci` | CI/CD changes |
| `deploy` | Deployment-related changes |

## Scope

A scope names the domain the change lives in, never a file or folder: `fix(acquire):`, `feat(search):`, `chore(evidence):`, `docs(research):`.

A cross-cutting change omits it: `chore: allow just test without a permission prompt`.

A skill or agent file change uses the `skill` scope: `docs(skill): add the result is the name minus the action naming rule`.

## Description

The description is one line, all lowercase (component names, file names and proper nouns included), with no period at the end, no body and no trailer, ever.

It starts with a verb, `add` for wholly new things, `update` for enhancements, `fix` for bugs, and `replace`, `remove`, `move`, `split`, `rename` and `improve` as the change warrants, and says specifically what changed: `add zone filtering`, not `update properties`.

There is no character cap; a description is as long as saying exactly what changed takes, and a specific 95-character line beats a vague 50-character one.

Two tightly coupled things join with `+`: `feat(skill): add review agent + dispatch it per axis from code-review`.

## Examples from this repo (mandatory reference — this is exactly how every commit must look)

```
feat(skill): add review agent + dispatch it per axis from code-review
chore(evidence): rename 873 bo aviso files to notice for the spanish filename guard
chore: allow just test without a permission prompt
style(acquire): format laws_test.go fixtures and tests
docs(skill): add the result is the name minus the action naming rule
fix(acquire): surface bo ley decode refusal as law-bo-decode-failed on decode-golden instead of law-bo-no-match
refactor(test): rename get_target_works to get_targets
feat(skill): add verify agent + dispatch it from the plan step
docs(skill): replace the 72 char commit cap with specificity over length
docs(research): update snippet gap to the matched-term window search stubs carry
style(search): apply formatting to the scoped chunk scan and its tests
test(search): report semantic lane candidates for golden queries under year and subject filters
fix(search): scope the chunk scan and keep scanning so filters never empty the semantic lane
feat(route): window search_corpus stubs on the matched query terms
chore(evidence): put the semantic lane evidence test behind a build tag
docs(evidence): show near-duplicate decoys are hnsw-unreachable for 957
docs(skill): ban a plain cd into a chunk worktree from the implement orchestrator
```

## No trailers

Never append `Co-Authored-By`, `Signed-off-by`, or any other trailer to commit messages. Keep messages clean — just the `type(scope): description` line.

## What NOT to do

```bash
# WRONG — vague description
feat: update stuff

# WRONG — uppercase or period
Feat: Add new feature.

# WRONG — scope is a file path
feat(src/lib/components): add button

# WRONG — missing verb
feat: notification system

# WRONG — trailer appended
feat: add button
Co-Authored-By: ...

# WRONG — multi-line message with body
refactor: make Tab a styled wrapper that accepts children

Tab is now a <div> that handles styling. The caller decides the inner
element — <a> for navigation, <button> for interaction.
```
