# Statuses (shared by /plan and /implement)

An issue moves through two sessions after triage, one label per stage. `/plan` and
`/implement` are gated on these labels and hand off through them — this file is the
single description both skills point at.

| Status              | Meaning                                                                 | Set by  | Consumed by  |
| ------------------- | ----------------------------------------------------------------------- | ------- | ------------ |
| `ready-for-planning`| Triaged and fully specified, but has no plan yet. Awaiting a planning session. | triage  | `/plan`      |
| `ready-for-agent`   | Plan comment attached. A cold session can implement from the issue body + plan comment alone. | `/plan` | `/implement` |

## Transitions

```
triage ──► ready-for-planning ──/plan──► ready-for-agent ──/implement──► closed
```

- **`/plan`** requires `ready-for-planning`. If it sees `ready-for-agent` instead, it stops
  (already planned → run `/implement`). On success it removes `ready-for-planning` and adds
  `ready-for-agent`, having posted the plan as an issue comment.
- **`/implement`** requires `ready-for-agent`. If it sees `ready-for-planning` instead, it stops
  (no plan yet → run `/plan`). On success it merges the work and closes the issue.

The plan comment posted at the `ready-for-planning → ready-for-agent` transition is the spec
`/implement` executes against: a cold session reading only the issue description and that comment
can implement with zero further research.
