# Composite :evergreen_tree:

Composite is a structural design pattern that lets you compose objects into tree structures and then work with these structures as if they were individual objects. [Theory](https://refactoring.guru/design-patterns/composite).

## Not Found in Laravel

After thorough investigation of the Laravel framework source at the
[pinned commit](https://github.com/laravel/framework/tree/5cc435df7a99231b1504f100c9f55e44a08bd210),
no code was found that genuinely implements the Composite pattern with
its required structural elements.

### What Was Investigated

Laravel's routing subsystem (route groups, `Router::group()`, `RouteGroup::merge()`), middleware name resolution (`MiddlewareNameResolver::parseMiddlewareGroup()`), the console scheduler (`Schedule`, `Event`, `CallbackEvent`), validation rules, and Blade components were all examined. Several of these exhibit tree-like or recursive structures, but none satisfies the Composite requirement that a shared Component interface be implemented by both individual objects and container objects.

### Why It Doesn't Qualify

The Composite pattern requires three structural participants: a **Component** interface (or abstract class) declaring a common operation, **Leaf** objects that implement the Component interface directly, and **Composite** objects that also implement the Component interface while holding a collection of child Component objects. The defining characteristic is that clients call the same operation on a Leaf and a Composite and get a meaningful result from both — the tree is transparent to the caller. The closest candidates in Laravel were:

- **Route groups (`Router::group()`)** — a route group is a transient stack frame (an associative array pushed onto `$this->groupStack`) during route registration, not an object; route groups and `Route` objects do not share a Component interface, and you cannot pass a group where a `Route` is expected
- **Middleware name resolution (`MiddlewareNameResolver`)** — group names resolve recursively to class-name strings, which is tree-like; however, a middleware class (an object implementing `handle()`) and a middleware group (a config array) are entirely different types with no shared Component interface
- **`CallbackEvent extends Event`** — `CallbackEvent` is a subclass of `Event` in the console scheduler, but this is simple inheritance; `Schedule` holds a flat list of `Event` objects, and `Event` objects do not hold child `Event` objects, so there is no recursive containment

### Related Patterns

The middleware group resolution in `MiddlewareNameResolver::parseMiddlewareGroup()` is recursive name resolution of a nested map — a useful structure, but better described as a **tree traversal** utility. The route group stack in `Router::group()` uses a stack-based attribute-inheritance idiom that has no established GoF pattern name; it is a practical design choice, not a cataloged pattern.
