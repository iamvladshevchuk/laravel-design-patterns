# Abstract Factory :factory:

Abstract Factory is a creational design pattern that lets you produce families of related objects without specifying their concrete classes. [Theory](https://refactoring.guru/design-patterns/abstract-factory).

## Not Found in Laravel

After thorough investigation of the Laravel framework source at the
[pinned commit](https://github.com/laravel/framework/tree/5cc435df7a99231b1504f100c9f55e44a08bd210),
no code was found that genuinely implements the Abstract Factory pattern with
its required structural elements.

### What Was Investigated

`Illuminate\Database\Connectors\ConnectionFactory`, all database Connection subclasses (`MySqlConnection`, `PostgresConnection`, `SQLiteConnection`, `SqlServerConnection`), Auth's `AuthManager` and `CreatesUserProviders` trait, `CacheManager`, `QueueManager`, `SessionManager`, `MailManager`, and their associated contract interfaces were all examined.

### Why It Doesn't Qualify

The Abstract Factory pattern requires five structural participants: an **AbstractFactory** interface declaring multiple creation methods (one per product type), at least two **ConcreteFactory** classes that each implement that interface to produce a complete *family* of related products, **AbstractProduct** interfaces for each product type, and **Client** code that depends only on the abstract interfaces. The closest candidates in Laravel were:

- **`ConnectionFactory`** — a single concrete class using a `match($driver)` statement to select which products to create; there is no `AbstractConnectionFactory` interface and no separate `MySqlConnectionFactory` or `PostgresConnectionFactory` class — this is a **Simple Factory** (Parameterized Factory), not Abstract Factory
- **`MySqlConnection` / `PostgresConnection`** — each overrides `getDefaultQueryGrammar()`, `getDefaultSchemaGrammar()`, and `getDefaultPostProcessor()` to produce a matching family of grammar and processor objects; this is the [Template Method](../../behavioral/template-method/) / [Factory Method](../factory-method/) pattern using inheritance, not Abstract Factory using parallel concrete factory classes
- **`AuthManager`, `CacheManager`, `QueueManager`, `SessionManager`** — each is a single class with `createXDriver()` methods keyed by driver name; there is one manager per domain, not multiple concrete factory classes implementing a shared factory interface

### Related Patterns

Laravel's dominant approach to creating families of driver-specific objects is the **Manager** pattern: a single class with `createXDriver()` methods and an `extend()` hook for custom drivers. This is most closely related to [Factory Method](../factory-method/). The per-connection family consistency (`MySqlConnection` always returning `MySqlGrammar` and `MySqlProcessor`) is achieved through [Template Method](../../behavioral/template-method/) in the Connection hierarchy.
