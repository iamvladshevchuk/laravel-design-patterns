# Decorator :gift:

Decorator is a structural design pattern that lets you attach new behaviors to objects by placing these objects inside special wrapper objects that contain the behaviors. [Theory](https://refactoring.guru/design-patterns/decorator).

## Table of Contents

1. [Logger](#1-logger)
2. [Cache Repository](#2-cache-repository)

## 1. Logger

> File: [src/Illuminate/Log/Logger.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Log/Logger.php)

Laravel's `Logger` class is a textbook Decorator. It implements `LoggerInterface` — the same PSR-3 interface that every real logging driver (Monolog, etc.) also implements. The constructor accepts another `LoggerInterface` instance and stores it:

```php
class Logger implements LoggerInterface
{
    protected $logger;
    protected $dispatcher;
    protected $context = [];

    public function __construct(LoggerInterface $logger, Dispatcher $dispatcher = null)
    {
        $this->logger = $logger;
        $this->dispatcher = $dispatcher;
    }

    /* ... CODE ... */
}
```

When you call any log method, `Logger` does not write to the log itself. It delegates to the wrapped logger, but first merges any accumulated context and then fires a log event so other parts of the application can react:

```php
protected function writeLog($level, $message, $context): void
{
    $this->logger->{$level}(
        $message = $this->formatMessage($message),
        $context = array_merge($this->context, $context)
    );

    $this->fireLogEvent($level, $message, $context);
}
```

The `withContext()` method is one of the behaviors added by the decorator. It accumulates key-value data that gets merged into every subsequent log call, without the inner logger knowing anything about it:

```php
public function withContext(array $context = [])
{
    $this->context = array_merge($this->context, $context);

    return $this;
}
```

The client code works with `Logger` through the `LoggerInterface` contract — the same interface as the underlying driver. Swapping the inner logger (e.g., from a file driver to a Slack driver) requires no changes to the decorator or to any code that calls it.

## 2. Cache Repository

> File: [src/Illuminate/Cache/Repository.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Cache/Repository.php)

The `Repository` class wraps a `Store` instance — the low-level cache backend (Redis, Memcached, file, etc.) — and decorates it with event dispatching, tag support, and convenience methods. The constructor accepts a `Store` and stores it as a dependency:

```php
class Repository implements ArrayAccess, CacheContract
{
    public function __construct(Store $store)
    {
        $this->store = $store;
    }

    /* ... CODE ... */
}
```

The `get()` method shows the decoration in action: it delegates the actual read to `$this->store->get()`, then fires either a `CacheHit` or `CacheMissed` event depending on the result. The underlying `Store` has no knowledge of events — that behavior is added purely by the decorator layer:

```php
public function get($key, $default = null): mixed
{
    if (is_array($key)) {
        return $this->many($key);
    }

    $value = $this->store->get($this->itemKey($key));

    // If we could not find the cache value, we will fire the missed event and get
    // the default value for this cache value. This default could be a callback
    // so we will execute the value function which will resolve it if needed.
    if (is_null($value)) {
        $this->event(new CacheMissed($key));

        $value = value($default);
    } else {
        $this->event(new CacheHit($key, $value));
    }

    return $value;
}
```

When a store supports tags, `tags()` returns a `TaggedCache` — itself another decorator layer that extends `Repository` and scopes all cache operations to a namespace. This makes it possible to stack decorators: `$cache->tags(['users'])->get('profile:1')` routes through `TaggedCache` (which prefixes the key), then through `Repository` (which dispatches events), and finally through `Store` (which performs the actual I/O):

```php
public function tags($names)
{
    if (! $this->supportsTags()) {
        throw new BadMethodCallException('This cache store does not support tagging.');
    }

    $cache = $this->store->tags(is_array($names) ? $names : func_get_args());

    if (! is_null($this->events)) {
        $cache->setEventDispatcher($this->events);
    }

    return $cache->setDefaultCacheTime($this->default);
}
```
