---
name: codebase-design
description: Shared vocabulary for designing deep modules. Use when the user wants to design or improve a module's interface, find deepening opportunities, decide where a seam goes, make code more testable or AI-navigable, or when another skill needs the deep-module vocabulary.
---

# Codebase Design

Design deep modules: a lot of behaviour behind a small interface, placed at a clean seam, testable through that interface. Use this language and these principles wherever code is being designed or restructured. The aim is leverage for callers, locality for maintainers, and testability for everyone.

## Glossary

Use these terms exactly and never substitute "component," "service," "API," or "boundary"; consistent language is the whole point.

A module is anything with an interface and an implementation, deliberately scale-agnostic: a function, class, package, or tier-spanning slice (avoid: unit, component, service).

Its interface is everything a caller must know to use it correctly: the type signature, but also invariants, ordering constraints, error modes, required configuration, and performance characteristics (avoid: API, signature, which are too narrow because they refer only to the type-level surface).

Its implementation is what's inside it, its body of code, distinct from an adapter: a thing can be a small adapter with a large implementation (a Postgres repo) or a large adapter with a small implementation (an in-memory fake), so reach for "adapter" when the seam is the topic and "implementation" otherwise.

Depth is leverage at the interface: the amount of behaviour a caller (or test) can exercise per unit of interface they have to learn. A module is deep when a large amount of behaviour sits behind a small interface, shallow when the interface is nearly as complex as the implementation.

A seam (Michael Feathers) is a place where you can alter behaviour without editing in that place, the location at which a module's interface lives; where to put the seam is its own design decision, distinct from what goes behind it (avoid: boundary, overloaded with DDD's bounded context).

An adapter is a concrete thing that satisfies an interface at a seam, describing role (what slot it fills), not substance (what's inside).

Leverage is what callers get from depth: more capability per unit of interface they learn, one implementation paying back across N call sites and M tests.

Locality is what maintainers get from depth: change, bugs, knowledge and verification concentrate in one place rather than spreading across callers, so a fix once is fixed everywhere.

## Deep versus shallow

A deep module is a small interface over a lot of implementation.

```
┌─────────────────────┐
│   Small Interface   │  ← Few methods, simple params
├─────────────────────┤
│                     │
│  Deep Implementation│  ← Complex logic hidden
│                     │
└─────────────────────┘
```

A shallow module is a large interface over little implementation, and is avoided.

```
┌─────────────────────────────────┐
│       Large Interface           │  ← Many methods, complex params
├─────────────────────────────────┤
│  Thin Implementation            │  ← Just passes through
└─────────────────────────────────┘
```

When designing an interface, ask whether you can reduce the number of methods, simplify the parameters, and hide more complexity inside.

## Principles

Depth is a property of the interface, not the implementation: a deep module can be internally composed of small, mockable, swappable parts that just aren't part of the interface, so a module can have internal seams (private to its implementation, used by its own tests) as well as the external seam at its interface.

The deletion test: imagine deleting the module; if complexity vanishes, it was a pass-through, and if complexity reappears across N callers, it was earning its keep.

The interface is the test surface: callers and tests cross the same seam, and if you want to test past the interface, the module is probably the wrong shape.

One adapter means a hypothetical seam and two adapters mean a real one, so don't introduce a seam unless something actually varies across it.

## Designing for testability

Good interfaces make testing natural. Accept dependencies rather than creating them.

```typescript
// Testable
function processOrder(order, paymentGateway) {}

// Hard to test
function processOrder(order) {
  const gateway = new StripeGateway();
}
```

Return results rather than producing side effects.

```typescript
// Testable
function calculateDiscount(cart): Discount {}

// Hard to test
function applyDiscount(cart): void {
  cart.total -= discount;
}
```

Keep the surface area small: fewer methods mean fewer tests needed, and fewer params mean simpler test setup.

## Relationships

A module has exactly one interface, the surface it presents to callers and tests, and depth is a property of the module measured against that interface.

A seam is where a module's interface lives, and an adapter sits at a seam and satisfies the interface.

Depth produces leverage for callers and locality for maintainers.

## Rejected framings

Depth as the ratio of implementation lines to interface lines (Ousterhout) rewards padding the implementation, so depth-as-leverage is used instead.

"Interface" as the TypeScript `interface` keyword or a class's public methods is too narrow, because interface here includes every fact a caller must know.

"Boundary" is overloaded with DDD's bounded context; say seam or interface.

## Going deeper

Deepening a cluster given its dependencies, with dependency categories, seam discipline and replace-don't-layer testing, is in [DEEPENING.md](./DEEPENING.md). Exploring alternative interfaces, spinning up parallel sub-agents to design the interface several radically different ways and comparing on depth, locality and seam placement, is in [DESIGN-IT-TWICE.md](./DESIGN-IT-TWICE.md).
