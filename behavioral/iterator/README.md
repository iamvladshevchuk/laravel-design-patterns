# Iterator :loop:

Iterator is a behavioral design pattern that lets you traverse elements of a collection without exposing its underlying representation. [Theory](https://refactoring.guru/design-patterns/iterator).

## Table of Contents

1. [Collection](#1-collection)
2. [LazyCollection](#2-lazycollection)

## 1. Collection

> File: [src/Illuminate/Collections/Collection.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Collections/Collection.php)

Laravel's `Collection` class implements `Enumerable`, which extends PHP's built-in `IteratorAggregate` interface. This is the Iterator pattern in its simplest form — when you `foreach` over a Collection, PHP calls `getIterator()` under the hood, which returns an `ArrayIterator` wrapping the internal items array:

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

## 2. LazyCollection

> File: [src/Illuminate/Collections/LazyCollection.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Collections/LazyCollection.php)

`LazyCollection` also implements `Enumerable` but uses lazy evaluation instead of loading all items into memory at once. Its `getIterator()` delegates to `makeIterator()`, which supports closures (generators), other `IteratorAggregate` instances, and plain arrays:

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

/* ... CODE ... */

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

Eloquent's `cursor()` method returns a `LazyCollection`, enabling memory-efficient iteration over large database result sets without loading all records into memory at once.
