---
name: tdd
description: Test-driven development. Use when the user wants to build features or fix bugs test-first, mentions "red-green-refactor", or wants integration tests.
---

# Test-driven development

TDD is the red to green loop, and this skill is the reference that makes that loop produce tests worth keeping: what a good test is, where tests go, the anti-patterns, and the rules of the loop.

Every section applies on every cycle, consulted before and during the loop, not after.

When exploring the codebase, [CONTEXT.md](../../../CONTEXT.md) is read so test names and interface vocabulary match the project's domain language, and the decisions in [docs/adr/](../../../docs/adr/) hold in the area being touched.

## What a good test is

Tests verify behavior through public interfaces, not implementation details, so code can change entirely while tests do not. A good test reads like a specification and survives refactors because it does not care about internal structure.

```
user can checkout with valid cart
```

Examples of good and bad tests are in [tests.md](./tests.md) and the mocking guidelines in [mocking.md](./mocking.md).

## Seams, where tests go

A seam is the public boundary a test sits at: the interface where behavior is observed without reaching inside.

Tests live at seams, never against internals, and only at pre-agreed ones: before any test is written, the seams under test are written down and confirmed with the user, and no test is written at an unconfirmed seam.

Not everything can be tested, and agreeing the seams up front is how testing effort lands on the critical paths and complex logic instead of every edge case.

```
What's the public interface, and which seams should we test?
```

When the shape of that interface is itself in question, how deep the module is, where the seam belongs, what the interface exposes, [codebase-design/SKILL.md](../codebase-design/SKILL.md) is the vocabulary: the shared source of the module, interface, depth, seam, adapter, leverage and locality terms, a reference to consult rather than a session to run.

## Anti-patterns

### Implementation-coupled

A test that mocks internal collaborators, tests private methods, or verifies through a side channel such as querying the database instead of using the interface. The tell is that the test breaks on a refactor while behavior has not changed.

### Tautological

A test whose assertion recomputes the expected value the way the code does, so it passes by construction and can never disagree with the code: a snapshot derived by hand the same way, a constant asserted equal to itself.

```
expect(add(a, b)).toBe(a + b)
```

Expected values come from an independent source of truth: a known-good literal, a worked example, the spec.

### Horizontal slicing

Writing all tests first, then all implementation. Bulk tests verify imagined behavior, testing the shape of things rather than user-facing behavior, go insensitive to real changes, and commit to test structure before the implementation is understood.

Work goes in vertical slices instead, one test then one implementation, repeated, each test a tracer bullet that responds to what the last cycle taught.

## Rules of the loop

Red comes before green: the failing test is written first, then only enough code to pass it, with no anticipation of future tests and no speculative features.

One slice at a time: one seam, one test, one minimal implementation per cycle.

Refactoring is not part of the loop; it belongs to the review stage in [code-review/SKILL.md](../code-review/SKILL.md), not the red to green implementation cycle.
