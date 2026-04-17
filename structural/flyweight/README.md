# Flyweight :feather:

Flyweight is a structural design pattern that lets you fit more objects into the available amount of RAM by sharing common parts of state between multiple objects instead of keeping all of the data in each object. [Theory](https://refactoring.guru/design-patterns/flyweight).

## Not Found in Laravel

After thorough investigation of the Laravel framework source at the
[pinned commit](https://github.com/laravel/framework/tree/5cc435df7a99231b1504f100c9f55e44a08bd210),
no code was found that genuinely implements the Flyweight pattern with
its required structural elements.

### What Was Investigated

The `Manager` driver pool, database `Connection` and `Grammar` classes, `BladeCompiler`, and `RouteCollection` were all examined. Several exhibit object reuse or stateless behavior, but none implements the complete Flyweight structure.

### Why It Doesn't Qualify

The Flyweight pattern requires four distinct structural participants: a **Flyweight** interface declaring methods that accept extrinsic state, **ConcreteFlyweight** objects that store intrinsic (shared, immutable) state only, a **FlyweightFactory** that maintains a pool and returns an existing flyweight when one with the same intrinsic state already exists, and a **Client** that computes and passes extrinsic state on each call. The critical requirement is the FlyweightFactory deduplication step — without it, there is no structural sharing, only coincidental reuse. The closest candidates in Laravel were:

- **`Manager` driver pool** — caches one driver instance per named key (`$this->drivers[$driver]`); the cached drivers are mutable, full-service objects (holding connection state, queues, etc.), not lightweight immutable state carriers — this is better described as a **Multiton** or **Registry** pattern
- **`Connection::queryGrammar`** — the Grammar object is genuinely stateless (compilation rules are class-level, query clauses are passed in per call), representing the intrinsic/extrinsic split that Flyweight requires; however, each `Connection` creates its own `new QueryGrammar` instance independently with no shared pool — two connections of the same driver type each get a separate, identical Grammar rather than sharing one instance through a FlyweightFactory
- **`BladeCompiler`** — a single shared compiler instance with stateless compilation methods; template content is extrinsic, but there is no pool of compiler objects sharing intrinsic state and no factory deduplication

### Related Patterns

The `Manager` driver pool is better described as a **Multiton** (a Registry keyed by name), which is closely related to the [Singleton](../singleton/) pattern. The Query Grammar's stateless design — where compilation rules are fixed and query-specific state is always passed in by the Builder — is a characteristic of the [Strategy](../../behavioral/strategy/) pattern, where the Grammar acts as an interchangeable algorithm with no mutable state.
