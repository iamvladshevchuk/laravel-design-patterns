# Iterator :loop:

Iterator is a behavioral design pattern that lets you traverse elements of a collection without exposing its underlying representation. [Theory](https://refactoring.guru/design-patterns/iterator).

## Table of Contents

1. [Collection](#1-collection)
    * [How it works](#how-it-works)
    * [How Laravel extends it](#how-laravel-extends-it)
2. [LazyCollection](#2-lazycollection)
    * [How lazy evaluation works](#how-lazy-evaluation-works)
    * [How Eloquent uses it](#how-eloquent-uses-it)

## 1. Collection

> File: [src/Illuminate/Collections/Collection.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Collections/Collection.php)

### How it works

PHP has a built-in interface for the Iterator pattern — [IteratorAggregate](https://www.php.net/manual/en/class.iteratoraggregate.php). Any class that implements it must define a `getIterator()` method, and PHP will use that method whenever you `foreach` over an object of that class.

Laravel's `Collection` doesn't implement `IteratorAggregate` directly. Instead, it implements [Enumerable](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Collections/Enumerable.php), which extends `IteratorAggregate` along with other interfaces:

```php
interface Enumerable extends Arrayable, Countable, IteratorAggregate, Jsonable, JsonSerializable
{
    /* ... CODE ... */
}
```

The `Collection` class implements this interface:

```php
class Collection implements ArrayAccess, CanBeEscapedWhenCastToString, Enumerable
{
    /* ... CODE ... */
}
```

And satisfies the `IteratorAggregate` contract with a simple `getIterator()` method. It returns an [ArrayIterator](https://www.php.net/manual/en/class.arrayiterator.php) — a built-in PHP class that knows how to iterate over an array. This is what lets you `foreach` over any Laravel collection:

```php
/**
 * Get an iterator for the items.
 *
 * @return \ArrayIterator<TKey, TValue>
 */
public function getIterator()
{
    return new ArrayIterator($this->items);
}
```

### How Laravel extends it

Instead of exposing the raw `ArrayIterator`, Laravel wraps it inside the `Collection` class and adds dozens of useful methods on top. For example, [filter()](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Collections/Collection.php) iterates over items internally and returns a new `Collection` with only the matching items:

```php
/**
 * Run a filter over each of the items.
 *
 * @param  (callable(TValue, TKey): bool)|null  $callback
 * @return static
 */
public function filter(callable $callback = null)
{
    if ($callback) {
        return new static(Arr::where($this->items, $callback));
    }

    return new static(array_filter($this->items));
}
```

The developer never deals with `ArrayIterator` directly — `Collection` hides it and provides a richer interface. This is the core idea of the Iterator pattern: traverse elements without exposing the underlying representation.

## 2. LazyCollection

> File: [src/Illuminate/Collections/LazyCollection.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Collections/LazyCollection.php)

`LazyCollection` also implements `Enumerable`, so it has the same `getIterator()` contract. The difference is in *when* items are computed. A regular `Collection` stores all items in an array in memory. A `LazyCollection` can accept a closure that produces items one at a time using PHP [generators](https://www.php.net/manual/en/language.generators.overview.php):

```php
/**
 * Create a new lazy collection instance.
 *
 * @param  \Illuminate\Contracts\Support\Arrayable<TKey, TValue>|iterable<TKey, TValue>|(Closure(): \Generator<TKey, TValue, mixed, void>)|self<TKey, TValue>|array<TKey, TValue>|null  $source
 * @return void
 */
public function __construct($source = null)
{
    if ($source instanceof Closure || $source instanceof self) {
        $this->source = $source;
    } elseif (is_null($source)) {
        $this->source = static::empty();
    } elseif ($source instanceof Generator) {
        throw new InvalidArgumentException(
            'Generators should not be passed directly to LazyCollection. Instead, pass a generator function.'
        );
    } else {
        $this->source = $this->getArrayableItems($source);
    }
}
```

When you iterate over a `LazyCollection`, PHP calls `getIterator()`, which delegates to `makeIterator()`:

```php
/**
 * Get an iterator from the enumerable.
 *
 * @return \Traversable
 */
public function getIterator()
{
    return $this->makeIterator($this->source);
}
```

`makeIterator()` handles the three types of sources that the constructor accepts. If the source is a closure (a generator function), it calls it to get a generator. If it's another `IteratorAggregate` (like a `Collection`), it gets its iterator. Otherwise, it wraps a plain array in an `ArrayIterator`:

```php
/**
 * Make an iterator from the given source.
 *
 * @param  \Closure|\Traversable|array  $source
 * @return \Traversable
 */
private function makeIterator($source): Traversable
{
    if ($source instanceof Closure) {
        return $source();
    }

    if ($source instanceof IteratorAggregate) {
        return $source->getIterator();
    }

    return new ArrayIterator($source ?: []);
}
```

### How lazy evaluation works

The key is that methods like `filter()` and `map()` don't execute immediately. Instead, they return a *new* `LazyCollection` with a generator function that will do the work only when iterated:

```php
public function filter(callable $callback = null)
{
    if (is_null($callback)) {
        $callback = function ($value) {
            return (bool) $value;
        };
    }

    return new static(function () use ($callback) {
        foreach ($this as $key => $value) {
            if ($callback($value, $key)) {
                yield $key => $value;
            }
        }
    });
}
```

The `yield` keyword is what makes it lazy — PHP pauses the function after each yielded value and only resumes when the next value is requested. This means chaining `->filter()->map()->first()` won't process the entire dataset. It processes items one at a time and stops as soon as a result is found.

### How Eloquent uses it

> File: [src/Illuminate/Database/Eloquent/Builder.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Database/Eloquent/Builder.php)

Eloquent's `cursor()` method returns a `LazyCollection`. Instead of fetching all database rows into memory (like `get()` does), it fetches them one at a time using a generator:

```php
/**
 * Get a lazy collection for the given query.
 *
 * @return \Illuminate\Support\LazyCollection
 */
public function cursor()
{
    return $this->applyScopes()->query->cursor()->map(function ($record) {
        return $this->newModelInstance()->newFromBuilder($record);
    });
}
```

This is useful when processing large datasets. For example, iterating over a million users with `User::cursor()` will only keep one user model in memory at a time, while `User::all()` would load all of them at once.
