# Decorator :gift:

Decorator is a structural design pattern that lets you attach new behaviors to objects by placing these objects inside special wrapper objects that contain the behaviors. [Theory](https://refactoring.guru/design-patterns/decorator).

## Table of Contents

1. [Logger](#1-logger)

## 1. Logger

> File: [src/Illuminate/Log/Logger.php](https://github.com/laravel/framework/blob/5cc435df7a99231b1504f100c9f55e44a08bd210/src/Illuminate/Log/Logger.php)

Laravel's `Logger` class is a textbook Decorator. The defining structural property of the Decorator pattern is that the **wrapper** and the **wrappee** implement the same interface, so clients cannot tell them apart. `Logger` implements `Psr\Log\LoggerInterface` — the same PSR-3 contract that every real logging driver (Monolog, NullLogger, etc.) also implements. The constructor accepts another `LoggerInterface` instance, stored as the **ConcreteComponent** being wrapped:

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

The two behaviors added by the decorator — context merging and event dispatch — are invisible to the underlying `$this->logger`. The `withContext()` method accumulates key-value data that gets merged into every subsequent log call without the inner logger knowing anything about it:

```php
public function withContext(array $context = [])
{
    $this->context = array_merge($this->context, $context);

    return $this;
}
```

Client code works with `Logger` through the `LoggerInterface` contract — the same interface as the underlying driver. Swapping the inner logger (e.g., from a file driver to a Slack driver) requires no changes to the decorator or to any code that calls it. Because both `Logger` and the wrapped driver implement `LoggerInterface`, they are fully interchangeable from the caller's perspective — the hallmark of the Decorator pattern.
