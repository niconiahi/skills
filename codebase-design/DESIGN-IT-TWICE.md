# Design It Twice

When the user wants to explore alternative interfaces for a chosen deepening candidate, use this parallel sub-agent pattern, based on Ousterhout's "Design It Twice": your first idea is unlikely to be the best. It uses the vocabulary in [SKILL.md](./SKILL.md): module, interface, seam, adapter, leverage.

## Process

### Frame the problem space

Before spawning sub-agents, write a user-facing explanation of the problem space for the chosen candidate: the constraints any new interface would need to satisfy, the dependencies it would rely on and which category each falls into per [DEEPENING.md](./DEEPENING.md), and a rough illustrative code sketch that grounds the constraints, not a proposal but a way to make them concrete.

Show this to the user, then immediately proceed to spawning; the user reads and thinks while the sub-agents work in parallel.

### Spawn sub-agents

Spawn three or more sub-agents in parallel, each producing a radically different interface for the deepened module. Prompt each with a separate technical brief (file paths, coupling details, the dependency category from [DEEPENING.md](./DEEPENING.md), what sits behind the seam), independent of the user-facing problem-space explanation, and give each a different design constraint.

```
Agent 1   Minimize the interface — aim for 1–3 entry points max. Maximise leverage per entry point.
Agent 2   Maximise flexibility — support many use cases and extension.
Agent 3   Optimise for the most common caller — make the default case trivial.
Agent 4   (if applicable) Design around ports & adapters for cross-seam dependencies.
```

Include both the [SKILL.md](./SKILL.md) vocabulary and the [CONTEXT.md](../../../CONTEXT.md) vocabulary in the brief so each sub-agent names things consistently with the architecture language and the project's domain language.

Each sub-agent outputs the interface (types, methods, params, plus invariants, ordering and error modes), a usage example showing how callers use it, what the implementation hides behind the seam, the dependency strategy and adapters per [DEEPENING.md](./DEEPENING.md), and the trade-offs, where leverage is high and where it's thin.

### Present and compare

Present designs sequentially so the user can absorb each one, then compare them in prose, contrasting by depth (leverage at the interface), locality (where change concentrates) and seam placement.

After comparing, give your own recommendation, which design is strongest and why, and if elements from different designs would combine well, propose a hybrid. Be opinionated: the user wants a strong read, not a menu.
