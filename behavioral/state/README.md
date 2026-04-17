# State :traffic_light:

State is a behavioral design pattern that lets an object alter its behavior when its internal state changes. It appears as if the object changed its class. [Theory](https://refactoring.guru/design-patterns/state).

## Not Found in Laravel

After thorough investigation of the Laravel framework source at the
[pinned commit](https://github.com/laravel/framework/tree/5cc435df7a99231b1504f100c9f55e44a08bd210),
no code was found that genuinely implements the State pattern with
its required structural elements.

### What Was Investigated

The Eloquent Model (`$exists` flag), database Connection (`$pretending` flag), Queue Worker and job classes, HTTP Client, Authentication guards, Session handlers, and Notification channels were all examined. Every candidate uses boolean-flag conditional branching — the very anti-pattern the State pattern is designed to replace — rather than polymorphic delegation to separate State objects.

### Why It Doesn't Qualify

The State pattern requires three distinct structural participants: a **Context** that holds a reference to a current **State** object and delegates behavior to it; a **State** interface or abstract class that all concrete states implement; and **ConcreteState** classes that each encapsulate the behavior for one particular state, potentially triggering transitions to other states. The closest candidates in Laravel were:

- **`Illuminate\Database\Eloquent\Model` (`$exists`)** — checks a boolean flag with `if`/`else` inside `save()` and `delete()`; there is no State interface, no ConcreteState classes, and `performInsert`/`performUpdate` are methods on the Model itself rather than on separate state objects
- **`Illuminate\Database\Connection` (`$pretending`)** — a boolean checked via early returns inside each query method; again no State interface or polymorphic delegation
- **Queue job state** (pending/processing/failed) — managed through flag properties and conditional logic in the Worker; no state objects are swapped at runtime
- **Session and Cache handlers** — simple flag-based or driver-pattern implementations with no state-object delegation

### Related Patterns

The boolean-flag branching seen in `Model::save()` and `Connection::select()` is the scenario the State pattern is designed to eliminate. The framework's driver-selection mechanism — where multiple classes implement a shared interface and the context delegates to whichever is configured — is better described as the [Strategy](../strategy/) pattern.
