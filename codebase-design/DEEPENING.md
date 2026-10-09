# Deepening

How to deepen a cluster of shallow modules safely, given its dependencies. Assumes the vocabulary in [SKILL.md](./SKILL.md): module, interface, seam, adapter.

## Dependency categories

When assessing a candidate for deepening, classify its dependencies; the category determines how the deepened module is tested across its seam.

### In-process

Pure computation, in-memory state, no I/O. Always deepenable: merge the modules and test through the new interface directly, with no adapter needed.

### Local-substitutable

Dependencies that have local test stand-ins (PGLite for Postgres, an in-memory filesystem). Deepenable if the stand-in exists; the deepened module is tested with the stand-in running in the test suite. The seam is internal, with no port at the module's external interface.

### Remote but owned (ports and adapters)

Your own services across a network boundary (microservices, internal APIs). Define a port (interface) at the seam; the deep module owns the logic and the transport is injected as an adapter, an in-memory one in tests and an HTTP, gRPC or queue one in production. The recommendation takes one shape.

```
Define a port at the seam, implement an HTTP adapter for production and an in-memory adapter for testing, so the logic sits in one deep module even though it's deployed across a network.
```

### True external (mock)

Third-party services (Stripe, Twilio, etc.) you don't control. The deepened module takes the external dependency as an injected port, and tests provide a mock adapter.

## Seam discipline

One adapter means a hypothetical seam and two adapters mean a real one: don't introduce a port unless at least two adapters are justified (typically production plus test), because a single-adapter seam is just indirection.

A deep module can have internal seams (private to its implementation, used by its own tests) as well as the external seam at its interface, and internal seams are not exposed through the interface just because tests use them.

## Testing strategy: replace, don't layer

Old unit tests on shallow modules become waste once tests at the deepened module's interface exist, so delete them, and write new tests at the deepened module's interface, because the interface is the test surface.

Tests assert on observable outcomes through the interface, not internal state, and survive internal refactors because they describe behaviour, not implementation; a test that has to change when the implementation changes is testing past the interface.
