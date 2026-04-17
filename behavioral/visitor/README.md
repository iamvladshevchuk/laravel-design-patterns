# Visitor :walking:

Visitor is a behavioral design pattern that lets you separate algorithms from the objects on which they operate, allowing you to add new operations to existing object structures without modifying them. [Theory](https://refactoring.guru/design-patterns/visitor).

## Not Found in Laravel

After thorough investigation of the Laravel framework source at the
[pinned commit](https://github.com/laravel/framework/tree/5cc435df7a99231b1504f100c9f55e44a08bd210),
no code was found that genuinely implements the Visitor pattern with
its required structural elements.

### What Was Investigated

The Query Builder Grammar (`Illuminate\Database\Query\Grammars\Grammar`), Validator (`Illuminate\Validation\Validator`), Schema Blueprint, Eloquent Builder, Validation Rules, and View classes were all examined. Both the Grammar and Validator use a technique that superficially resembles Visitor — dispatching to named methods based on a type tag — but neither implements the double-dispatch protocol that defines the pattern.

### Why It Doesn't Qualify

The Visitor pattern requires two distinct structural elements: an **Element** interface with an `accept(Visitor $v)` method where the element calls back `$v->visitConcreteElement($this)`, and a **Visitor** interface with `visit*()` methods for each element type. The two-way call is what makes it double-dispatch and what allows new operations to be added without modifying elements. The closest candidates in Laravel were:

- **`Illuminate\Database\Query\Grammars\Grammar`** — calls `$this->{"where{$where['type']}"}(...)` to dispatch based on a `type` key in an associative array; the "elements" are plain arrays, not objects, and they never call back on the Grammar — this is single-dispatch type-tag routing, not Visitor
- **`Illuminate\Validation\Validator`** — calls `$this->$method(...)` to route rule strings to validation methods; again, rules are strings rather than objects with `accept()`, and there is no Visitor interface
- **`Query\Expression`** — a simple value object with no `accept()` method; the Grammar processes it directly rather than through double-dispatch

### Related Patterns

The type-tag dispatch used in `Grammar::compileWheresToArray()` and `Validator::validateAttribute()` is a form of **single-dispatch polymorphism** applied to typed data structures. The Grammar's role in compiling query builder clauses for different database engines is better described as the [Strategy](../strategy/) pattern — multiple Grammar subclasses implement a shared interface and are swapped based on the configured database driver.
