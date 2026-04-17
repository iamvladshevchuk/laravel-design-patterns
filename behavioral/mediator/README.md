# Mediator :satellite:

Mediator is a behavioral design pattern that lets you reduce chaotic dependencies between objects by restricting direct communications between them and forcing collaboration only through a mediator object. [Theory](https://refactoring.guru/design-patterns/mediator).

## Not Found in Laravel

After thorough investigation of the Laravel framework source at the
[pinned commit](https://github.com/laravel/framework/tree/5cc435df7a99231b1504f100c9f55e44a08bd210),
no code was found that genuinely implements the Mediator pattern with
its required structural elements.

### What Was Investigated

The Event Dispatcher (`Illuminate\Events\Dispatcher`), Bus Dispatcher (`Illuminate\Bus\Dispatcher`), Router (`Illuminate\Routing\Router`), DatabaseManager (`Illuminate\Database\DatabaseManager`), Pipeline (`Illuminate\Pipeline\Pipeline`), and both HTTP and Console Kernels were all examined. In each case, the candidate class either implements a different well-known pattern or lacks the bidirectional colleague coordination that defines Mediator.

### Why It Doesn't Qualify

The Mediator pattern requires three distinct structural participants: a **Mediator** interface declaring coordination methods, a **ConcreteMediator** that holds references to specific named **Colleague** objects and routes messages between them, and **Colleague** classes that hold a back-reference to the mediator and use it to communicate with *other specific colleagues*. The closest candidates in Laravel were:

- **`Illuminate\Events\Dispatcher`** — implements the **Observer** pattern: listeners subscribe anonymously to event names, the dispatcher broadcasts one-way to all subscribers, and there are no mutual colleague references
- **`Illuminate\Bus\Dispatcher`** — implements the **Command** pattern: commands are self-contained request objects routed to handlers, not peer colleagues communicating through a central coordinator
- **`Illuminate\Routing\Router`** — acts as an orchestrator/factory that assembles middleware pipelines; routes call `setRouter()` for shared state access, but routes do not communicate with other routes through the Router, so bidirectional colleague coordination is absent
- **`Illuminate\Database\DatabaseManager`** — a Registry/Factory for database connections; connections hold no reference back to the manager and do not communicate with other connections through it

### Related Patterns

The `Illuminate\Events\Dispatcher` examined here is better described as [Observer](../observer/). The `Illuminate\Bus\Dispatcher` is better described as [Command](../command/).
